# 07. Validation Checklist & Acceptance Criteria

## 1. Before-and-After Measurement Protocol

To confirm that the Disk I/O root causes have been successfully eliminated, execute the following measurement sequence before and after applying any fix:

### 1.1 Baseline Capture (Before Remediation)
1. In Supabase SQL Editor, snapshot the current top dirtied and written blocks:
   ```sql
   SELECT query, calls, shared_blks_written, shared_blks_dirtied 
   FROM pg_stat_statements 
   ORDER BY shared_blks_dirtied DESC 
   LIMIT 10;
   ```
2. In Supabase Dashboard, record:
   - Current Disk I/O Budget percentage.
   - 24-hour Disk Read IOPS and Disk Write IOPS charts.
   - Cache Hit Rate percentage.
3. In PM2, check worker process count:
   ```bash
   pm2 list
   ```

### 1.2 Remediation Verification (After Applying Fixes)
1. **Reset `pg_stat_statements` (Optional, during maintenance window):**
   ```sql
   SELECT pg_stat_statements_reset();
   ```
2. **Execute a Simulated or Real 500-Message Broadcast Campaign:**
   - Observe worker behavior in PM2 logs.
   - Measure time taken and total queries executed.
3. **Capture Post-Remediation Telemetry:**
   - Verify `shared_blks_dirtied` per 1,000 sent messages has decreased by at least **50%**.
   - Verify `shared_blks_written` has decreased significantly.
   - Verify Disk I/O Budget consumption rate has stabilized or ceased declining.

---

## 2. Regression Testing Checklist

| Test Scenario | Module Under Test | Expected Behavior | Pass / Fail |
| :--- | :--- | :--- | :---: |
| **1. Inbound WhatsApp Message** | `routes/webhook.js`, `messages`, `conversations` | Message appears in Inbox immediately; conversation `last_message` and `unread_count` update accurately; Socket.IO emits event. | [ ] |
| **2. Inbound Media Download** | `routes/webhook.js`, Supabase Storage | Image/Audio/Video downloaded via curl and uploaded to `campaign-media`; public URL rendered in chat; no duplicate files created. | [ ] |
| **3. Duplicate Webhook Resend** | `routes/webhook.js` | Resending identical Meta webhook payload is skipped by Redis deduplication; zero database writes triggered. | [ ] |
| **4. Broadcast Campaign Send** | `campaignWorker.js`, `whatsappService.js` | All recipients receive message; `campaign_queue` marks `SENT`; `messages` row logged with single write; wallet balance updated correctly. | [ ] |
| **5. Status Updates (Delivered / Read)** | `routes/webhook.js` | Ticks turn double-gray (delivered) and blue (read) in UI; out-of-order updates ignored; no useless updates to `campaign_queue`. | [ ] |
| **6. Campaign Details Polling** | `CampaignDetails.tsx`, `routes/campaigns.js` | Counters (Total, Sent, Delivered, Read, Failed) update accurately in real-time; DB receives 1 aggregated query instead of 8 unindexed `ilike` counts. | [ ] |
| **7. PM2 Worker Single Execution** | `ecosystem.config.js`, `leaderLock.js` | Only one PM2 worker process acts as leader; no `LOCK_FAILED_RACE` messages logged in PM2 output. | [ ] |
| **8. Theme / Branding Update** | `routes/settings.js`, `branding_settings` | Theme settings update smoothly without embedding multi-megabyte base64 image strings. | [ ] |

---

## 3. Quantitative Acceptance Criteria

The remediation will be deemed successful when all of the following conditions are met:

1. **Disk I/O Budget:** The Supabase Disk I/O budget stops depleting and stabilizes. Daily baseline consumption drops below the burst replenishment rate.
2. **Write Amplification Reduction:**
   - Message logging write operations are reduced from **2 writes per message** to **1 atomic write per message**.
   - Webhook status updates no longer issue empty `UPDATE campaign_queue` queries for standard chat messages.
3. **Query Workload Reduction:**
   - Campaign Details screen generates at most **1 database query per polling interval** (instead of 8 wildcard `COUNT(*)` queries).
   - Duplicate worker polling across PM2 cluster instances is completely eliminated.
4. **Index Efficiency:**
   - Unused duplicate indexes (`idx_messages_conversation_created` and `idx_contacts_user_phone`) are removed, reducing write overhead on `messages` (140k rows) and `contacts` (25k rows).
5. **Zero Functional Regressions:**
   - All inbound/outbound messaging, flow triggers, CRM contact syncing, campaigns, and staff permissions continue operating without error.
