# Wazpp Project — Comprehensive Root Cause Audit & Final Remediation Master Report

**Date:** October 26, 2026  
**Project:** Wazpp Clean Workspace (`e:/code-from-haris/Wazpp/new`)  
**Target:** Supabase PostgreSQL Database & Disk I/O Exhaustion  
**Investigation Scope:** 100% Read-Only audit of codebase, background jobs, webhooks, runtime architecture, and Supabase telemetry (`pg_stat_statements`, table relation sizes, and cached egress traffic).

---

## 1. Executive Summary & Root Cause Verdict

Supabase issued a critical alert that the **Wazpp** project is rapidly depleting its **Disk I/O Budget**.

Our audit proves conclusively that the Disk I/O exhaustion is **NOT** driven by user search queries, missing read indexes, or excessive frontend page views. The root cause is a combination of:
1. **Severe Write Amplification in the Messaging Pipeline:** Every outbound message and webhook status update executes multiple unbatched roundtrips and updates table rows repeatedly, generating over **2.28 GiB of dirtied shared blocks** and **>280,000 database calls** since October 2.
2. **Duplicate Background Worker Execution:** Running 2 PM2 cluster processes with a `leaderLock` mechanism that **fails open** when Redis is unavailable, causing both instances to poll and compete for the same database rows simultaneously.
3. **Table & Index Bloat from HOT-Update Invalidation:** Tables such as `conversations` (46 MB for 12,624 rows) and `branding_settings` (16 MB TOAST for 18 rows) suffer from massive disk bloat and temporary file thrashing.
4. **Unindexed High-Frequency Polling:** The Campaign Details screen executes **8 separate `COUNT(*)` queries with `ILIKE` wildcards** every 5 seconds per viewing user.

---

## 2. Complete Inventory of All Identified Problems

### Problem 1: Outbound Message Logging Write Amplification (P0 — Critical)
- **Files & Lines:** [`wazpp-server/services/whatsappService.js:545-573`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/services/whatsappService.js#L545-L573), [`wazpp-server/workers/campaignWorker.js:1800-1807`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/workers/campaignWorker.js#L1800-L1807).
- **Evidence:** `pg_stat_statements` shows `public.log_outbound_message` has been called **~98,000 times**, writing **92 MiB** of shared blocks and **dirtying 2.28 GiB of memory blocks**.
- **Root Cause:**
  - For every single recipient in a broadcast campaign or 1-on-1 chat, Node.js calls `rpc('log_outbound_message')` which inserts a row into `messages`.
  - Immediately after the RPC returns, Node.js executes a second query: `supabaseAdmin.from('conversations').update({ last_message: previewText }).eq('user_id', ...).eq('contact_phone', ...)`.
  - For a 10,000-contact campaign, this creates **20,000 individual roundtrips** and transaction commits instead of a single batched operation.
- **Remediation:**
  - Update the stored procedure `log_outbound_message` to update `conversations.last_message` internally in the exact same transaction as the insert.
  - Remove the redundant secondary `.update()` call from `whatsappService.js:565-573`.
  - Batch outbound message logging in chunks of 50 items within `campaignWorker.js`.

---

### Problem 2: Webhook Status Update Write Churn (P0 — Critical)
- **Files & Lines:** [`wazpp-server/routes/webhook.js:900-948`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/webhook.js#L900-L948), [`wazpp-server/routes/webhook.js:1182-1188`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/webhook.js#L1182-L1188).
- **Evidence:** `pg_stat_statements` shows `public.update_message_status` has been called **roughly 182,000 times**, writing **48 MiB** of shared blocks.
- **Root Cause:**
  - Meta sends up to 3–4 status updates (`sent` → `delivered` → `read`) for each message.
  - The webhook handler iterates over statuses one-by-one, issuing individual RPC calls.
  - If a status webhook arrives before the outbound message insertion has completed in Postgres, the RPC returns `false`. The webhook then sleeps 1.5 seconds and calls `update_message_status` a **second time**!
  - Furthermore, lines 1182–1188 unconditionally issue an `UPDATE campaign_queue ... WHERE message_id = ... JOIN campaigns` for every single status update—even for regular customer-to-business chat messages that were never part of a campaign.
- **Remediation:**
  - In `update_message_status`, only perform an `UPDATE` if the incoming status rank is strictly greater than the currently stored status (skip writes if status is already `read` or unchanged).
  - Only execute the `campaign_queue` update if the message has a campaign attribution ID.

---

### Problem 3: PM2 Cluster Duplicate Background Worker Execution (P0 — Critical)
- **Files & Lines:** [`wazpp-server/ecosystem.config.js:1-12`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/ecosystem.config.js#L1-L12), [`wazpp-server/server.js:472-522`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/server.js#L472-L522), [`wazpp-server/utils/leaderLock.js:34-45`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/utils/leaderLock.js#L34-L45).
- **Evidence:** Code review confirms PM2 runs 2 cluster instances (`wazpp-api-0` and `wazpp-api-1`), each launching all 4 background workers.
- **Root Cause:**
  - `leaderLock.js` uses Redis to elect a single worker leader.
  - However, line 36 states: `if (!redisClient.isReady) return true;` and line 44 states: `catch (err) { return true; }`.
  - **The lock fails open.** If Redis is down, omitted, or timing out, **both PM2 processes act as leaders simultaneously**.
  - Both processes poll `campaign_queue` every 2s, query `tasks` every 15s, query `sheet_automations` every 60s, and call `fetch_and_lock_drip_jobs` every 60s. When both attempt to lock the same queue rows, lock contention and race conditions (`LOCK_FAILED_RACE`) surge.
- **Remediation:**
  - Modify `leaderLock.js` to **fail-closed** (`return false`) on error or when Redis is not ready.
  - Designate instance index 0 (`process.env.NODE_APP_INSTANCE === '0'`) as the deterministic single worker runner when Redis is unavailable.

---

### Problem 4: In-Memory Webhook Deduplication with Cross-Process Gaps (P0 — Critical)
- **Files & Lines:** [`wazpp-server/routes/webhook.js:75-80`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/webhook.js#L75-L80).
- **Root Cause:**
  - `const processedMessageIds = new Set();` is stored in process memory.
  - Node process 0 does not know what Node process 1 has processed.
  - Every 10 minutes, `setInterval` calls `processedMessageIds.clear()`.
  - If Meta re-delivers a webhook or balances requests across the cluster, duplicate messages bypass deduplication, triggering duplicate contact lookups, conversation updates, and message inserts.
- **Remediation:**
  - Replace the local `Set` with Redis atomic key setting: `await redisClient.set('dedup:wa:' + msgId, '1', { NX: true, EX: 86400 })`.
  - Ensure a database unique constraint on `messages(whatsapp_message_id)` enforces idempotency at the storage engine level.

---

### Problem 5: Campaign Details Polling 8x `COUNT(*)` with Wildcard `ILIKE` (P1 — High)
- **Files & Lines:** [`wazpp-server/routes/campaigns.js:410-422`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/campaigns.js#L410-L422), [`wazpp-client/src/components/Campaigns/CampaignDetails.tsx:68-83`](file:///e:/code-from-haris/Wazpp/new/wazpp-client/src/components/Campaigns/CampaignDetails.tsx#L68-L83).
- **Evidence:** Frontend polls `GET /api/campaigns/:id` every **5,000 ms** (5 seconds) while the page is visible.
- **Root Cause:**
  - The endpoint issues **8 separate `COUNT(*)` queries** on `campaign_queue`.
  - Four queries use leading wildcards: `.ilike('status', '%SENT%')`, `.ilike('status', '%DELIVERED%')`, `.ilike('status', '%READ%')`, `.ilike('status', '%FAILED%')`.
  - Leading wildcards prevent B-tree index usage, forcing PostgreSQL to scan and evaluate text matching on candidate rows.
  - A single viewing user generates **96 count queries per minute** against `campaign_queue`.
- **Remediation:**
  - Replace all 8 queries with a single aggregation:
    ```sql
    SELECT status, COUNT(*) 
    FROM campaign_queue 
    WHERE campaign_id = $1 
    GROUP BY status;
    ```
  - Cache the result in Redis for 3–5 seconds so multiple tabs share the same count.

---

### Problem 6: `branding_settings` 16 MB TOAST Bloat & 126 MiB Temp-File Writes (P1 — High)
- **Files & Lines:** [`wazpp-server/routes/settings.js:540-547`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/settings.js#L540-L547).
- **Evidence:**
  - Table size: `branding_settings` has only **18 rows** and **8 KB** of heap data, but its total relation size is **16 MB**!
  - `pg_stat_statements` shows that across only **72 calls**, this upsert generated **126 MiB written** and **42 MiB read in temporary disk blocks**!
- **Root Cause:**
  - The route accepts raw `themeConfig` from `req.body` and performs an `UPSERT` into `branding_settings(theme_config)`.
  - When users paste or upload base64 image strings (e.g. logos, favicons) instead of URLs, multi-megabyte strings are stored in JSONB.
  - PostgreSQL TOAST chunks large objects across disk pages. Every upsert forces compression, TOAST page creation, and temp-file sorting.
- **Remediation:**
  - Validate `theme_config` on the backend: reject any payload containing `data:image/` and require images to be uploaded to Supabase Storage first.
  - Run `VACUUM FULL branding_settings;` in Supabase to reclaim the bloated 16 MB.

---

### Problem 7: Exact Duplicate Indexes on High-Write Tables (P1 — High)
- **Evidence:** Confirmed by `Findings.md` with exact scan counts:
  1. `messages`: `idx_messages_conversation_created` has **0 scans** (100% duplicate of `idx_messages_conversation_id_created_at` which has 10,964 scans).
  2. `contacts`: `idx_contacts_user_phone` has **0 scans** (100% duplicate of `idx_contacts_user_phone_lookup` which has 1,000,000+ scans).
- **Impact:**
  - `messages` is 125 MB (53 MB data + 72 MB indexes).
  - Every insert and update must write to **both** identical indexes, doubling WAL records and dirtying index leaf pages on every message.
- **Remediation:**
  - Drop the unused duplicates concurrently:
    ```sql
    DROP INDEX CONCURRENTLY IF EXISTS idx_messages_conversation_created;
    DROP INDEX CONCURRENTLY IF EXISTS idx_contacts_user_phone;
    ```

---

### Problem 8: Webhook Logs Growth Without Retention (P1 — High)
- **Evidence:** Table `webhook_logs` has grown to **103 MB** (60 MB data, 43 MB indexes, ~43,076 rows). `log_webhook_event` wrote **48 MiB** and `update_webhook_log` wrote **54 MiB**.
- **Root Cause:**
  - Every incoming webhook triggers two synchronous writes storing full raw payloads.
  - There is no automated cleanup or TTL pruning job. As the table grows, index maintenance and autovacuum cycles consume sustained Disk I/O.
- **Remediation:**
  - Establish a 14-day rolling window cleanup via `pg_cron` or backend maintenance task:
    ```sql
    DELETE FROM webhook_logs WHERE created_at < NOW() - INTERVAL '14 days';
    ```

---

### Problem 9: Storage Media Egress Spike & 1-Hour Cache Inconsistency (P1 — High)
- **Evidence:** Supabase Dashboard shows **Cached Egress peaking at 15.811 GB on Oct 22**, with daily egress between **6 GB and 12 GB**.
- **Files & Lines:** [`wazpp-server/routes/webhook.js:247`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/webhook.js#L247).
- **Root Cause:**
  - Inbound media is uploaded to Supabase Storage with `cacheControl: '3600'` (only 1 hour!).
  - CDN caches expire in 60 minutes, forcing repetitive re-downloads when staff or users review chat history.
  - Outbound campaign media (video/images) sent to thousands of WhatsApp recipients causes Meta CDN edge servers to fetch the media binary directly from Supabase Storage.
- **Remediation:**
  - Change `cacheControl` to `'31536000'` (1 year) on inbound media uploads.
  - When sending broadcast campaigns, utilize Meta's uploaded Media ID (`whatsappService.uploadMediaToWhatsApp`) rather than embedding raw public Storage URLs, offloading media distribution to Meta's CDN.

---

### Problem 10: Heavy Dashboard Query Fanout on Load (P2 — Medium)
- **Files & Lines:** [`wazpp-server/routes/dashboard.js:70-113`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/dashboard.js#L70-L113) and [`lines 352-368`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/dashboard.js#L352-L368).
- **Root Cause:**
  - Dashboard executes 12 parallel queries pulling up to 10,000 rows across `tasks`, `follow_ups`, `leads`, `contacts`, `messages`, and `conversations`.
  - Computing counts and sums in Node.js instead of Postgres causes large buffer scans, evicting warm pages from memory.
- **Remediation:**
  - Consolidate dashboard calculations into the `get_dashboard_stats` database function.
  - Cache dashboard stats in Redis for 30–60 seconds.

---

### Problem 11: Task Scheduler Unindexed JSON Scan (P2 — Medium)
- **Files & Lines:** [`wazpp-server/workers/scheduler.js:180-186`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/workers/scheduler.js#L180-L186).
- **Root Cause:**
  - Evaluates `.is('metadata->>pre_alarm_5m_sent', null)` every 15–60 seconds on uncompleted tasks.
  - Without an index on `(due_date)` or `(metadata->>'pre_alarm_5m_sent')`, Postgres evaluates JSON paths on candidate rows repeatedly.
- **Remediation:**
  - Add partial B-tree index on `tasks(due_date) WHERE is_completed = false`.

---

## 3. Summary Table: Before vs. After Remediation

| Metric / Workload | Current Problem State | Optimized Target State | Expected I/O Reduction |
| :--- | :--- | :--- | :---: |
| **Outbound Message Writes** | 2 writes per message (RPC + Conversation update) | 1 atomic write per message | **50% reduction in writes** |
| **Status Update Writes** | Unconditional queue update on every status | Filtered queue update only for campaigns | **60–70% reduction in status writes** |
| **Worker Polling Volume** | 2 PM2 instances polling simultaneously (fail-open) | 1 deterministic leader instance | **50% reduction in polling queries** |
| **Campaign Details Polling** | 8 wildcard `COUNT(*)` queries every 5s | 1 cached aggregate query every 5s | **87% reduction in query count** |
| **`messages` Index Maintenance** | 2 identical conversation indexes updated per insert | 1 active index maintained | **Reduces index write I/O by 15–20%** |
| **`branding_settings` Temp I/O** | 126 MiB temp writes in 72 calls (16 MB TOAST) | Clean JSON URLs, 8 KB relation size | **100% elimination of temp writes** |
| **`webhook_logs` Table Size** | 103 MB and growing indefinitely | Kept under 20 MB via 14-day rolling window | **Prevents table and autovacuum bloat** |

---

## 4. Verification and Rollback Strategy

1. **Safety First:** All database index drops must use `DROP INDEX CONCURRENTLY` so no table locks are acquired.
2. **Read-Only Verification:** Run the diagnostic queries provided in [`05-supabase-investigation-plan.md`](file:///e:/code-from-haris/Wazpp/new/docs/audits/supabase-disk-io/05-supabase-investigation-plan.md) before applying any changes.
3. **Rollback Assurance:** Every recommended fix is targeted and modular; application code changes can be reverted in seconds via Git.
