# 06. Prioritized Remediation Plan

## Remediation Guidelines & Constraints
- **Minimal, Targeted Changes:** Address the exact write amplification, concurrency flaws, and query fanouts without rewriting the application architecture.
- **Zero Downtime:** Schema updates (such as index removals) must use `CONCURRENTLY`.
- **Pre-requisite:** Verify against current telemetry and staging before applying to production.

---

## Priority Matrix

| ID | Priority | Confidence | Title | Impact on Disk I/O |
| :---: | :---: | :---: | :--- | :--- |
| **FIX-01** | **P0** | High | Consolidate and batch message logging & conversation updates | **Massive:** Eliminates up to 50% of outbound message writes and HOT update failures. |
| **FIX-02** | **P0** | High | Prevent duplicate worker execution: Fix `leaderLock` fail-open | **High:** Halves background polling query volume across PM2 cluster instances. |
| **FIX-03** | **P0** | High | Webhook atomic deduplication via Redis and status update batching | **High:** Eliminates duplicate message inserts and unneeded status write churn. |
| **FIX-04** | **P1** | High | Optimize Campaign Details Polling: Replace 8 count queries with single aggregate | **High:** Reduces 96 unindexed wildcard count queries/min to 1 cached aggregate query. |
| **FIX-05** | **P1** | High | Safely drop exact duplicate indexes on `messages` and `contacts` | **Medium-High:** Reduces write I/O and WAL volume on every single message and contact write. |
| **FIX-06** | **P1** | High | Resolve `branding_settings` TOAST explosion: Reject inline base64 images | **High:** Eliminates 126 MiB temp-file writes and TOAST bloat on branding updates. |
| **FIX-07** | **P1** | High | Implement rolling retention / pruning on `webhook_logs` | **Medium:** Prevents `webhook_logs` from growing past 103 MB and generating vacuum I/O. |
| **FIX-08** | **P2** | Medium | Cache dashboard stats in Redis & eliminate unneeded row scanning | **Medium:** Protects database cache from being evicted by 10k-row scans on dashboard load. |
| **FIX-09** | **P2** | Medium | Standardize Supabase Storage media Cache-Control headers | **Network/Storage:** Prevents repetitive re-validation of inbound media attachments. |

---

## Detailed Remediation Steps

### FIX-01: Consolidate Outbound Message Logging & Eliminate Redundant Updates (P0)
- **Problem:** [`whatsappService.js:545-573`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/services/whatsappService.js#L545-L573) calls `log_outbound_message` RPC, then immediately performs a second `.update()` query on `conversations` setting `last_message`. During campaigns, every single recipient generates two individual database writes, resulting in 98,000 calls and 2.28 GiB of dirtied blocks.
- **Remediation Action:**
  1. Modify `log_outbound_message` stored procedure in PostgreSQL to update `conversations.last_message` and `conversations.updated_at` internally in the same transaction as the message insertion.
  2. In `whatsappService.js` (lines 560–576), remove the separate Node.js `.update({ last_message })` query.
- **Files Modified:** `wazpp-server/services/whatsappService.js`, PostgreSQL procedure `log_outbound_message`.
- **Migration Required:** Yes, 1 idempotent SQL patch replacing `log_outbound_message` procedure.
- **Testing:** Verify sending a template message logs to `messages` and updates `conversations.last_message` in the Inbox UI.
- **Rollback Plan:** Re-add the second update in `whatsappService.js` and restore previous procedure definition.

---

### FIX-02: Fix `leaderLock` Fail-Open in PM2 Cluster Mode (P0)
- **Problem:** [`utils/leaderLock.js:34-45`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/utils/leaderLock.js#L34-L45) returns `true` if Redis is not ready or on error. With PM2 running 2 cluster instances, both instances run all workers simultaneously, doubling polling queries and causing lock contention.
- **Remediation Action:**
  1. Change `leaderLock.js` to **fail-closed** (`return false`) when Redis is not ready or throws an error.
  2. Designate PM2 instance 0 (`NODE_APP_INSTANCE === '0'`) as the fallback worker process if Redis is unavailable:
     ```javascript
     if (!redisClient.isReady) {
         return process.env.NODE_APP_INSTANCE === '0' || !process.env.NODE_APP_INSTANCE;
     }
     ```
- **Files Modified:** `wazpp-server/utils/leaderLock.js`.
- **Migration Required:** No.
- **Testing:** Start PM2 cluster with 2 instances; verify in logs that only one instance logs `[Worker]` or `[Scheduler]` loop messages.
- **Rollback Plan:** Revert `leaderLock.js`.

---

### FIX-03: Webhook Atomic Deduplication & Filtered Queue Updates (P0)
- **Problem:** `routes/webhook.js` uses an in-memory `Set` that is not shared across PM2 cluster instances and is cleared every 10 minutes. Furthermore, lines 1182–1188 run `campaign_queue.update()` for non-campaign status updates.
- **Remediation Action:**
  1. Implement Redis-based deduplication with 24-hour TTL: `await redisClient.set('dedup:wa:' + messageId, '1', { NX: true, EX: 86400 })`.
  2. In `webhook.js` (line 1182), only execute the `campaign_queue` update if `status.biz_opaque_callback_data` or message metadata indicates it belongs to a campaign.
- **Files Modified:** `wazpp-server/routes/webhook.js`.
- **Migration Required:** No.
- **Testing:** Send duplicate webhook payloads to `/api/webhook`; verify only the first executes database writes.
- **Rollback Plan:** Revert `routes/webhook.js`.

---

### FIX-04: Optimize Campaign Details Polling (P1)
- **Problem:** `routes/campaigns.js:410-422` runs 8 separate `COUNT(*)` queries with `ilike` wildcards on every request, polled every 5s by `CampaignDetails.tsx`.
- **Remediation Action:**
  1. Replace the 8 queries with a single aggregation query:
     ```javascript
     const { data: statusCounts } = await supabase.rpc('get_campaign_status_summary', { p_campaign_id: req.params.id });
     ```
     Or single SQL query: `SELECT status, COUNT(*) FROM campaign_queue WHERE campaign_id = $1 GROUP BY status`.
  2. Cache the computed statistics in Redis for 3 seconds (`campaign:stats:${campaignId}`).
- **Files Modified:** `wazpp-server/routes/campaigns.js`.
- **Migration Required:** Optional (can use raw query or simple RPC).
- **Testing:** Open Campaign Details in the frontend and verify progress counters match database numbers accurately.
- **Rollback Plan:** Revert `routes/campaigns.js`.

---

### FIX-05: Drop Exact Duplicate Indexes on `messages` and `contacts` (P1)
- **Problem:** Exact duplicate indexes identified by `Findings.md`:
  - `idx_messages_conversation_created` (0 scans, 100% duplicate of `idx_messages_conversation_id_created_at`).
  - `idx_contacts_user_phone` (0 scans, 100% duplicate of `idx_contacts_user_phone_lookup`).
- **Remediation Action:**
  Execute in Supabase SQL Editor:
  ```sql
  DROP INDEX CONCURRENTLY IF EXISTS idx_messages_conversation_created;
  DROP INDEX CONCURRENTLY IF EXISTS idx_contacts_user_phone;
  ```
- **Migration Required:** Yes, 1 safe DDL migration script.
- **Testing:** Verify `idx_messages_conversation_id_created_at` and `idx_contacts_user_phone_lookup` continue handling queries via `EXPLAIN`.
- **Rollback Plan:** Re-create the dropped index concurrently if needed.

---

### FIX-06: Resolve `branding_settings` TOAST Explosion (P1)
- **Problem:** `branding_settings` total size is 16 MB for only 18 rows due to inline base64 image strings stored in `theme_config`, generating 126 MiB of temp writes.
- **Remediation Action:**
  1. In `wazpp-server/routes/settings.js:540`, validate `req.body.theme_config`. If any string starts with `data:image/`, reject the request with `400 Bad Request: Images must be uploaded via /api/upload`.
  2. Run `VACUUM FULL branding_settings;` or `REINDEX TABLE branding_settings;` in Supabase to reclaim bloated disk space.
- **Files Modified:** `wazpp-server/routes/settings.js`.
- **Migration Required:** No schema migration needed; run vacuum in SQL Editor.
- **Testing:** Save theme settings with normal URLs; verify theme applies properly and table size shrinks.
- **Rollback Plan:** Revert `routes/settings.js`.

---

### FIX-07: Implement Rolling Retention on `webhook_logs` (P1)
- **Problem:** `webhook_logs` has grown to 103 MB (~43,076 rows) with no cleanup, generating high background autovacuum and I/O.
- **Remediation Action:**
  1. Create a lightweight daily maintenance job or `pg_cron` schedule to delete logs older than 14 days:
     ```sql
     DELETE FROM webhook_logs WHERE created_at < NOW() - INTERVAL '14 days';
     ```
  2. Or truncate old logs in staging/production during a low-traffic maintenance window.
- **Files Modified:** Backend scheduler or SQL `pg_cron` task.
- **Migration Required:** 1 SQL migration.
- **Testing:** Verify logs older than retention period are pruned without locking active transactions.
- **Rollback Plan:** Disable the retention job.
