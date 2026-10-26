# 02. Database Query & Workload Audit

## 1. Executive Query Workload Summary

The PostgreSQL `pg_stat_statements` telemetry (`Findings.md`) combined with the table size metrics demonstrates that **Write Amplification** and **Unbatched Row-by-Row Operations** are the primary contributors to Supabase Disk I/O budget depletion. Cumulative metrics since October 2, 2026 reveal:

- **`log_outbound_message`**: ~98,000 calls; 92 MiB blocks written; **2.28 GiB dirtied blocks**.
- **`update_message_status`**: ~182,000 calls; 48 MiB blocks written.
- **`log_webhook_event` & `update_webhook_log`**: >100 MiB blocks written combined.
- **`branding_settings`**: 126 MiB written and 42 MiB read in temporary files across just 72 calls!

Below is the detailed technical breakdown of each identified query bottleneck in the current codebase.

---

## 2. Suspicious Queries & Inefficient Data Access Patterns

### Finding Q-01: Unbatched Outbound Message Logging Write Storm (P0)
- **File & Lines:** [`wazpp-server/services/whatsappService.js:545-573`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/services/whatsappService.js#L545-L573) called by [`wazpp-server/workers/campaignWorker.js:1800-1807`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/workers/campaignWorker.js#L1800-L1807).
- **Calling Context:** Executed on **every individual recipient** when sending broadcast campaigns or individual WhatsApp messages.
- **Database Tables Involved:** `messages`, `conversations`.
- **Observed / Expected Frequency:** Up to 50 concurrent messages per batch; for a campaign of 10,000 contacts, this executes 10,000 individual RPC calls and 10,000 subsequent `conversations` updates.
- **Queries Executed per Batch:**
  1. `rpcClient.rpc('log_outbound_message', { ... })` — Creates/finds conversation and inserts into `messages`.
  2. `rpcClient.from('conversations').update({ last_message: previewText }).eq('user_id', ...).eq('contact_phone', ...).select('id')` — Redundant second update on `conversations` immediately following the RPC!
- **Why It Increases Disk I/O:**
  1. Each single-row `INSERT` and `UPDATE` produces its own WAL (Write-Ahead Logging) records and page dirties.
  2. In Postgres, `messages` has a total size of 125 MB (53 MB data + 72 MB indexes). Each single insert must update all B-tree indexes (`idx_messages_conversation_id_created_at`, `idx_messages_whatsapp_message_id`, etc.), dirtying multiple 8 KB index leaf pages in cache and forcing disk flush during checkpoints.
  3. Updating `conversations` immediately after the RPC duplicates row writes and causes HOT (Heap-Only Tuples) update failures because indexed columns are touched, creating row versions and bloating the table (demonstrated by the 46 MB relation size for only 12,624 rows!).
- **Status:** **CONFIRMED DEFECT** (Directly verified by code and corroborated by `pg_stat_statements` recording 98,000 calls and 2.28 GiB dirtied).
- **Recommended Optimization:**
  - Consolidate conversation `last_message` updating directly inside `log_outbound_message` so that a message log produces exactly one database operation instead of two.
  - Implement bulk logging: collect sent message results from the campaign worker chunk (e.g., 50 items) and perform a single batch `INSERT INTO messages` instead of 50 individual RPC roundtrips.
- **Potential Risks:** Low risk. Application logic remains identical.

---

### Finding Q-02: Status Update Fanout and Redundant Campaign Queue Updates (P0)
- **File & Lines:** [`wazpp-server/routes/webhook.js:900-948`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/webhook.js#L900-L948) and [`wazpp-server/routes/webhook.js:1182-1188`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/webhook.js#L1182-L1188).
- **Calling Context:** WhatsApp Webhook status receipt (`sent`, `delivered`, `read`, `failed`).
- **Database Tables Involved:** `messages`, `campaign_queue`, `campaigns`.
- **Expected Frequency:** Up to 3 to 4 status webhooks per outgoing message (10,000 sent messages = 30,000 to 40,000 status webhooks).
- **Queries Executed per Status:**
  1. `messages.select('*').in('whatsapp_message_id', waIds)` (Line 876)
  2. `rpcAsService('update_message_status', { p_whatsapp_message_id, p_status })` (Line 900)
  3. **Race Condition Retry:** If row wasn't found, waits 1.5s and calls `update_message_status` again (Line 912).
  4. If `!updatedMsg`: `messages.select('*').eq('whatsapp_message_id', ...).order('created_at', { ascending: false }).limit(1)` (Line 941).
  5. `campaign_queue.update(updatePayload).eq('message_id', waMessageId).select('campaign_id, campaigns (title, template_name)')` (Line 1183).
- **Why It Increases Disk I/O:**
  - Notice item 5: For **every status webhook**—including personal 1-on-1 chats and inbound messages that have nothing to do with campaigns—the code unconditionally fires an `UPDATE campaign_queue` matching on `message_id` with a relation join `select('campaign_id, campaigns(title, template_name)')`!
  - If `campaign_queue.message_id` is searched without an index or on non-existent rows, Postgres performs index scans or sequential scans on `campaign_queue` and writes empty update transaction locks.
  - `update_message_status` has been called **~182,000 times**, generating 48 MiB written blocks.
- **Status:** **CONFIRMED DEFECT**.
- **Recommended Optimization:**
  - Update `update_message_status` to only touch rows when `status` actually transitions forward (e.g., ignore `delivered` if already marked `read`).
  - Only execute the `campaign_queue` update if the message is known to be from a campaign (or filter `status.biz_opaque_callback_data` / prefix check on ID).
- **Potential Risks:** Minimal; ensures non-campaign messages don't perform unnecessary writes on campaign tables.

---

### Finding Q-03: Campaign Details 8x `COUNT(*)` Polling Fanout (P1)
- **File & Lines:** [`wazpp-server/routes/campaigns.js:410-422`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/campaigns.js#L410-L422), triggered every 5s by [`wazpp-client/src/components/Campaigns/CampaignDetails.tsx:70-75`](file:///e:/code-from-haris/Wazpp/new/wazpp-client/src/components/Campaigns/CampaignDetails.tsx#L70-L75).
- **Calling Context:** Frontend `GET /api/campaigns/:id` polling while user views campaign progress.
- **Database Tables Involved:** `campaign_queue`.
- **Expected Frequency:** Every 5 seconds per viewing user.
- **Queries Executed per Request:**
  ```javascript
  const [total, sent, delivered, read, failed, processing] = await Promise.all([
      supabase.from('campaign_queue').select('*', { count: 'exact', head: true }).eq('campaign_id', req.params.id),
      supabase.from('campaign_queue').select('*', { count: 'exact', head: true }).eq('campaign_id', req.params.id).ilike('status', '%SENT%'),
      supabase.from('campaign_queue').select('*', { count: 'exact', head: true }).eq('campaign_id', req.params.id).ilike('status', '%DELIVERED%'),
      supabase.from('campaign_queue').select('*', { count: 'exact', head: true }).eq('campaign_id', req.params.id).ilike('status', '%READ%'),
      supabase.from('campaign_queue').select('*', { count: 'exact', head: true }).eq('campaign_id', req.params.id).ilike('status', '%FAILED%'),
      supabase.from('campaign_queue').select('*', { count: 'exact', head: true }).eq('campaign_id', req.params.id).in('status', ['PROCESSING', 'PENDING'])
  ]);
  const pending = await supabase.from('campaign_queue').select('*', { count: 'exact', head: true }).eq('campaign_id', req.params.id).eq('status', 'PENDING');
  const processingCount = await supabase.from('campaign_queue').select('*', { count: 'exact', head: true }).eq('campaign_id', req.params.id).eq('status', 'PROCESSING');
  ```
- **Why It Increases Disk I/O:**
  1. **8 individual `COUNT(*)` queries per request**!
  2. Four of the count queries use `.ilike('status', '%STATUS%')` with leading and trailing wildcards. Postgres cannot use a standard B-tree index on `(campaign_id, status)` for `%STATUS%` wildcards, forcing Postgres to scan candidate rows and evaluate regex/ilike on each row.
  3. If 2 users keep the campaign tab open, they generate **96 count queries per minute** against `campaign_queue`.
- **Status:** **CONFIRMED DEFECT**.
- **Recommended Optimization:**
  - Replace the 8 separate count queries with a single aggregation query:
    ```sql
    SELECT status, COUNT(*) 
    FROM campaign_queue 
    WHERE campaign_id = $1 
    GROUP BY status;
    ```
  - Cache this aggregate result in Redis for 3–5 seconds so rapid polls from multiple browser tabs share a single cached count.
- **Potential Risks:** None. Response payload structure to the frontend remains identical.

---

### Finding Q-04: TOAST & Temp-File Thrashing in `branding_settings` (P1)
- **File & Lines:** [`wazpp-server/routes/settings.js:540-547`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/settings.js#L540-L547).
- **Calling Context:** User theme/branding saving (`PUT /api/settings/branding`).
- **Database Tables Involved:** `branding_settings`.
- **Telemetry Evidence:**
  - Table Size: **16 MB Total Size** for only **18 rows** (Table data is 8,192 bytes = 1 page; TOAST table is ~16 MB).
  - `pg_stat_statements`: **126 MiB written** and **42 MiB read in temp blocks** across only 72 calls!
- **Why It Increases Disk I/O:**
  - `theme_config` accepts arbitrary JSON from `req.body` without size validation or payload stripping.
  - When users paste or upload custom logos or favicon data URLs (base64 image strings) into the theme builder, a multi-megabyte string is embedded directly in the `theme_config` JSONB column.
  - PostgreSQL TOAST chunks any field larger than 2 KB into the `pg_toast` table. Every `UPSERT` forces Postgres to compress, slice, and write thousands of TOAST chunks, dirtying shared buffers and generating temp files during sorting/indexing.
- **Status:** **CONFIRMED DEFECT / ANOMALY**.
- **Recommended Optimization:**
  - Reject inline base64 image strings in `theme_config`. Any logo or image must be uploaded to Supabase Storage first (`/api/upload`) and saved as an HTTP URL in `theme_config`.
  - Vacuum and re-index `branding_settings` in Supabase to reclaim bloated TOAST space.
- **Potential Risks:** Frontend theme builder must ensure images are uploaded before sending theme payload.

---

### Finding Q-05: Exact Duplicate Indexes on High-Write Tables (P1)
- **Database Tables Involved:** `messages`, `contacts`.
- **Telemetry Evidence (`Findings.md`):**
  1. `messages`:
     - `idx_messages_conversation_created`: **0 scans** (Unused duplicate).
     - `idx_messages_conversation_id_created_at`: **10,964 scans** (Actively used).
  2. `contacts`:
     - `idx_contacts_user_phone`: **0 scans** (Unused duplicate).
     - `idx_contacts_user_phone_lookup`: **1,000,000+ scans** (Actively used).
- **Why It Increases Disk I/O:**
  - Every single `INSERT`, `UPDATE`, and `DELETE` on `messages` (140,850 rows, 125 MB) and `contacts` (25,618 rows, 22 MB) must write to **both** identical indexes.
  - Maintaining an index with 0 scans wastes disk write bandwidth, memory in `shared_buffers`, and doubles WAL volume on every message receipt and send.
- **Status:** **CONFIRMED DEFECT**.
- **Recommended Optimization:**
  - Safely drop the unused duplicates: `DROP INDEX CONCURRENTLY IF EXISTS idx_messages_conversation_created;` and `DROP INDEX CONCURRENTLY IF EXISTS idx_contacts_user_phone;`.
- **Potential Risks:** Zero risk if dropped concurrently; the identical active indexes continue serving all queries.

---

### Finding Q-06: Heavy Dashboard Query Fanout on Page Load (P2)
- **File & Lines:** [`wazpp-server/routes/dashboard.js:70-113`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/dashboard.js#L70-L113) and [`lines 352-368`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/dashboard.js#L352-L368).
- **Calling Context:** User navigates to Dashboard (`GET /api/dashboard/stats`).
- **Database Tables Involved:** `tasks`, `follow_ups`, `whatsapp_templates`, `user_integrations`, `campaigns`, `pipelines`, `pipeline_stages`, `leads`, `contacts`, `flows`, `contact_groups`, `conversations`, `messages`.
- **Queries Executed per Page Load:**
  - 12 parallel table queries:
    - `tasks.select('is_completed, due_date').limit(5000)`
    - `follow_ups.select('status, scheduled_at').limit(5000)`
    - `whatsapp_templates.select('status', { count: 'exact' }).limit(2000)`
    - `campaigns.select(..., { count: 'exact' }).limit(200)`
    - `leads.select(...).limit(10000)` (fallback when RPC is unavailable)
    - `contacts.select('*', { count: 'exact', head: true })`
    - `flows.select('*', { count: 'exact', head: true })`
    - `contact_groups.select('*', { count: 'exact', head: true })`
    - `conversations.select('id').limit(2000)` followed by `messages.select('created_at, status').in('conversation_id', cids).limit(2000)`
    - `leads.select('created_at').limit(5000)`
- **Why It Increases Disk I/O:**
  - Pulling thousands of records into Node.js memory just to compute sums and counts in JavaScript causes large buffer reads in Postgres, evicting frequently cached pages and generating disk read churn.
- **Status:** **CONFIRMED INEFFICIENCY**.
- **Recommended Optimization:**
  - Ensure `get_dashboard_stats` database function handles these aggregations directly in PostgreSQL, returning a single JSON summary row (~1 KB).
  - Cache the dashboard summary in Redis for 30–60 seconds.
- **Potential Risks:** Low; need to verify `get_dashboard_stats` RPC returns matching data.
