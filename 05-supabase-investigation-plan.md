# 05. Supabase Production Investigation Plan & Safe Verification Queries

## 1. Objectives & Safety Constraints

This investigation plan outlines safe, **read-only** verification steps to execute within the Supabase SQL Editor and Supabase Dashboard.
- **Rule 1:** Only read-only catalog views (`pg_stat_statements`, `pg_stat_user_tables`, `pg_stat_user_indexes`, `pg_stat_bgwriter`, `pg_class`) may be queried.
- **Rule 2:** Never execute destructive DDL, `TRUNCATE`, `DROP`, or `UPDATE` statements during the investigation phase.
- **Rule 3:** Do not run `EXPLAIN (ANALYZE)` in production on write queries or unbounded scans because `ANALYZE` executes the statement. Use `EXPLAIN` without `ANALYZE` to inspect query plans safely.

---

## 2. Production Metrics Checklist (Supabase Dashboard)

Before running SQL queries, record the following charts from the Supabase Dashboard (`Reports` / `Database` / `Storage`):
1. **Disk I/O Budget:** Check the percentage of Disk I/O budget remaining and note the exact hours when the budget drops most steeply.
2. **Disk IOPS & Disk Throughput:** Compare read IOPS vs. write IOPS. (Telemetry indicates write IOPS will dominate).
3. **RAM & Cache Hit Ratio:** Check whether the database buffer cache hit ratio is above 99% or drops below 95% during campaign bursts.
4. **Storage & Cached Egress:** Track daily Cached Egress (which spiked to 15.8 GB on Oct 22 and remains at 6–12 GB daily).
5. **Connection Count & Pooling:** Check Supavisor active connections vs client connections to verify whether connection spikes coincide with PM2 worker polling cycles.

---

## 3. Safe Read-Only Verification SQL Queries

Run each of the following queries in the Supabase SQL Editor to obtain real-time diagnostic evidence.

### Query 1: Top 10 High-Write & High-I/O Queries (`pg_stat_statements`)
This query captures cumulative write, dirtied, and temporary block activity to confirm which procedures produce physical disk writes:
```sql
SELECT 
    substring(query, 1, 100) AS query_preview,
    calls,
    round(total_exec_time::numeric, 2) AS total_time_ms,
    round(mean_exec_time::numeric, 2) AS mean_time_ms,
    shared_blks_written,
    shared_blks_dirtied,
    temp_blks_written,
    temp_blks_read
FROM pg_stat_statements
ORDER BY shared_blks_dirtied DESC
LIMIT 10;
```
*Expected Result:* Confirms `log_outbound_message`, `update_message_status`, and `log_webhook_event` as the leading dirtied block producers.

---

### Query 2: Inspecting Temp File & TOAST Bloat on `branding_settings`
Determines the exact size distribution between heap data, indexes, and TOAST storage for `branding_settings`:
```sql
SELECT
    relname AS relation_name,
    pg_size_pretty(pg_relation_size(c.oid)) AS heap_size,
    pg_size_pretty(pg_total_relation_size(c.oid) - pg_relation_size(c.oid)) AS toast_and_indexes_size,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname = 'public' 
  AND c.relname = 'branding_settings';
```
*Expected Result:* Validates that `branding_settings` holds ~8 KB of heap data and ~16 MB of TOAST/index storage, confirming oversized JSONB payloads.

---

### Query 3: Verifying Duplicate & Unused Indexes
Inspects the exact scan counts of candidate duplicate indexes on `messages` and `contacts`:
```sql
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan AS number_of_scans,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE relname IN ('messages', 'contacts')
ORDER BY relname, indexrelname;
```
*Expected Verification:*
- `idx_messages_conversation_created`: Confirmed 0 scans (vs `idx_messages_conversation_id_created_at` with >10,000 scans).
- `idx_contacts_user_phone`: Confirmed 0 scans (vs `idx_contacts_user_phone_lookup` with >1,000,000 scans).

---

### Query 4: Checkpoint & Background Writer Activity (`pg_stat_bgwriter`)
Checks whether PostgreSQL is frequently forced to flush dirty buffers to disk due to checkpoint pressure from high write volume:
```sql
SELECT 
    checkpoints_timed,
    checkpoints_req AS checkpoints_requested_early,
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint,
    buffers_clean,
    buffers_backend AS buffers_written_directly_by_backends
FROM pg_stat_bgwriter;
```
*Interpretation:* If `checkpoints_req` is high compared to `checkpoints_timed`, the volume of incoming writes from WhatsApp logging and campaign processing is filling `max_wal_size` faster than normal scheduled checkpoints, forcing aggressive disk flushes and consuming the Disk I/O budget.

---

### Query 5: Safe Query Plan Check for Campaign Queue Polling
Checks the execution plan of the campaign details count query **without executing it**:
```sql
EXPLAIN
SELECT count(*) 
FROM campaign_queue 
WHERE campaign_id = '00000000-0000-0000-0000-000000000000' 
  AND status ILIKE '%SENT%';
```
*Expected Verification:* Inspects whether Postgres is forced to do a sequential scan or filter every row due to the leading wildcard `%SENT%`.

---

## 4. How to Correlate High Disk I/O with Application Events

To determine which feature or operational event triggers a Disk I/O spike:
1. **Correlate with Broadcast Campaigns:**
   - Note the timestamp of campaign launches from `campaigns.created_at` or `campaigns.scheduled_at`.
   - Check if Disk I/O spikes precisely when `campaignWorker.js` begins processing the queue and sending messages.
2. **Correlate with High-Volume Inbound Webhook Waves:**
   - Compare the Supabase Dashboard API Request chart against `webhook_logs` timestamps.
   - Look for spikes in `update_message_status` and `log_webhook_event` calls matching Meta webhook delivery windows.
3. **Correlate with Background Worker Restarts:**
   - Check PM2 logs (`pm2 logs`) for restarts. When PM2 cluster processes restart or lose Redis connection, `leaderLock` fails open, causing duplicate worker execution and an immediate surge in polling queries.
