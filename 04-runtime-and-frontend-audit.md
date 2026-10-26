# 04. Runtime, Background Jobs & Frontend Request Audit

## 1. Multi-Process Background Worker Execution (P0)

### 1.1 The Cluster Architecture Defect
The application is deployed using PM2 in cluster mode as defined in [`ecosystem.config.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/ecosystem.config.js):
```javascript
apps: [
  {
    name: 'wazpp-api-0',
    script: 'server.js',
    exec_mode: 'cluster',
    instances: 2,
    max_memory_restart: '1200M'
  }
]
```
In [`wazpp-server/server.js:472-522`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/server.js#L472-L522), when each process boots up, it executes:
- `campaignWorker.start()` (Line 482)
- `scheduler.start()` (Line 488)
- `dripEngine.start()` (Line 485)
- `setInterval(runSheetJobs, 60000)` (Line 521)

This means **every background worker is duplicated across both PM2 processes**.

---

### 1.2 The `leaderLock.js` Fail-Open Vulnerability
To coordinate between the multiple instances, the codebase introduced a Redis-based leader election in [`utils/leaderLock.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/utils/leaderLock.js):

```javascript
// Lines 34-45 in utils/leaderLock.js
async function acquire(key, ttlMs) {
    if (!redisClient.isReady) {
        // Redis not ready or not configured -> FAIL OPEN
        return true; 
    }
    try {
        const result = await redisClient.set(key, instanceId, {
            PX: ttlMs,
            NX: true
        });
        return result === 'OK';
    } catch (err) {
        console.error('[LeaderLock] Error acquiring lock:', err.message);
        return true; // ← FAILS OPEN ON ANY ERROR!
    }
}
```

#### Why This Depletes Supabase Disk I/O:
1. **Unconditional Fail-Open:** If `REDIS_URL` is omitted, misconfigured, restarting, flapping, or under high connection load, `leaderLock.acquire()` returns `true` for **both processes**.
2. **Double Query Polling:**
   - Both PM2 processes simultaneously query `campaign_queue` every 2–5 seconds.
   - Both processes query `tasks` every 15–30 seconds.
   - Both processes query `sheet_automations` every 60 seconds.
   - Both processes execute `fetch_and_lock_drip_jobs` RPC every 60 seconds.
3. **Optimistic Locking Contention:**
   - When both worker processes find the same candidate rows in `campaign_queue` and attempt to lock them via `UPDATE campaign_queue SET status = 'SENDING' WHERE id IN (...)`, one process succeeds while the other process encounters a lock race condition (logged as `LOCK_FAILED_RACE` at line 1093 in `campaignWorker.js`).
   - This doubles the database write and transaction volume on `campaign_queue`, generating lock wait queues, deadlocks, and duplicate WAL traffic.

---

## 2. Background Polling Workload Audit

| Background Task | Implementation File | Frequency | Database Access Pattern | Risk & I/O Impact |
| :--- | :--- | :--- | :--- | :--- |
| **Campaign Queue Consumer** | [`workers/campaignWorker.js:435-442`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/workers/campaignWorker.js#L435-L442) | Every 2s–5s when active, 60s idle | `SELECT` candidates from `campaign_queue`, `UPDATE` to `SENDING`, individual `UPDATE` to `SENT`. | **Critical (P0):** Generates tens of thousands of individual writes during campaign execution. |
| **Scheduled Campaign Maintenance** | [`workers/campaignWorker.js:474-510`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/workers/campaignWorker.js#L474-L510) | Every 30s | `SELECT campaigns WHERE status = 'SCHEDULED'` and `count('exact')` on `campaign_queue`. | **Low (P3):** Lightweight index check. |
| **Task Reminder Scheduler** | [`workers/scheduler.js:151-186`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/workers/scheduler.js#L151-L186) | Every 15s–60s adaptive | Scans `tasks` for due, upcoming, and overdue tasks. Queries: `.is('metadata->>pre_alarm_5m_sent', null).limit(1000)`. | **Medium (P2):** Evaluates unindexed JSON operator `metadata->>` on every cycle. |
| **Drip Campaign Engine** | [`workers/dripProcessor.js:195-200`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/workers/dripProcessor.js#L195-L200) | Every 60s (100ms when backlogged) | Calls stored procedure `fetch_and_lock_drip_jobs(batch_size: 100)`. | **Low–Medium (P2):** Safe stored procedure with row locking, but duplicates if leader lock fails open. |
| **Sheet Automation Sync** | [`services/SheetAutomationService.js:246-260`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/services/SheetAutomationService.js#L246-L260) | Every 60s | Queries `sheet_automations WHERE status = 'active'` and updates `last_run_at`. | **Low (P3):** Low table row count; minimal I/O impact unless large sheets are constantly synced. |

---

## 3. Frontend Polling & Request Frequency Audit

### 3.1 `CampaignDetails.tsx` Aggressive Polling Loop (P1)
In [`wazpp-client/src/components/Campaigns/CampaignDetails.tsx:68-83`](file:///e:/code-from-haris/Wazpp/new/wazpp-client/src/components/Campaigns/CampaignDetails.tsx#L68-L83):
```typescript
const startPolling = () => {
    stopPolling();
    pollInterval.current = setInterval(() => {
        if (!document.hidden) {
            fetchCampaignDetails(true);
            fetchQueue(true);
        }
    }, 5000); // Polls every 5 seconds!
};
```
- **What happens every 5 seconds:**
  1. `fetchCampaignDetails()` calls `GET /api/campaigns/:id`, which executes **8 separate `COUNT(*)` queries** against `campaign_queue` using wildcard `ILIKE '%STATUS%'` (as documented in `02-database-query-audit.md`).
  2. `fetchQueue()` calls `GET /api/campaigns/:id/queue`, which executes a paginated query with `{ count: 'exact' }` and joins `contacts(name)`.
- **Workload Impact:** A single user watching a live campaign progress bar issues **108 database queries per minute**. If multiple operators have this screen open, it causes constant sequential table scans and buffer eviction on `campaign_queue`.

### 3.2 Inbox Polling & Real-Time Sync
- In [`wazpp-client/src/components/Inbox.tsx`](file:///e:/code-from-haris/Wazpp/new/wazpp-client/src/components/Inbox.tsx), the inbox relies primarily on Socket.IO events (`new_message`, `message_updated`, `conversation_updated`) rather than aggressive polling.
- However, when selecting conversations, `GET /api/inbox/messages/:conversationId` is fetched, which executes `.select('*')` with `.limit(50)`. Telemetry confirms `messages` has an index on `(conversation_id, created_at DESC)` that is actively used (10,964 scans), meaning inbox reads are **not** the primary driver of Disk I/O.
