# 01. Codebase Inventory & Architecture Map

## 1. Overview & System Components

The Wazpp codebase is organized into two primary applications:
1. **`wazpp-server`**: Node.js / Express backend API with embedded background workers, queue consumers, webhook listeners, and Redis integration.
2. **`wazpp-client`**: React (TypeScript) + Vite single-page application with Tailwind CSS and Supabase JS SDK.

---

## 2. Server Architecture (`wazpp-server`)

### 2.1 Process & Deployment Configuration
- **Entry Point:** [`server.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/server.js)
- **Process Manager:** [`ecosystem.config.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/ecosystem.config.js)
  - Configured with 2 cluster instances: `wazpp-api-0` and `wazpp-api-1`.
  - Runs with `exec_mode: 'cluster'`, `instances: 2`, `max_memory_restart: '1200M'`.
- **Background Worker Spawning:**
  - `server.js` (lines 472–522) instantiates and starts:
    1. `CampaignWorker` (loops every 2–60s)
    2. `Scheduler` (loops every 15–60s)
    3. `dripEngine` (loops every 60s)
    4. `SheetAutomationService` (loops every 60s)
  - **Critical Architectural Risk:** All four background workers are spawned inside the Express web server process. Because PM2 runs 2 cluster instances, both instances spawn their own internal copy of all four background workers.
- **Leader Election:** [`utils/leaderLock.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/utils/leaderLock.js)
  - Uses Redis keys (`SET key val NX PX ttl`).
  - **Fail-Open Defect:** If Redis is disconnected, not configured, or experiencing network latency, `leaderLock.acquire()` catches errors and returns `true` (`if (!redisClient.isReady) return true; catch { return true; }`). Under this condition, **both** PM2 processes run scheduled database scans simultaneously.

### 2.2 Supabase & Database Access Modules
- **Client Factory:** [`lib/supabaseClient.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/lib/supabaseClient.js)
  - Exports `supabaseAdmin` (bypasses RLS using `SUPABASE_SERVICE_ROLE_KEY` / `SUPABASE_SERVICE_KEY`).
  - Exports `supabase` (anonymous client).
  - Exports `getAuthenticatedSupabase(req)` (creates a client scoped to the caller's JWT).
  - Exports `rpcAsService(fnName, args)` (executes PostgreSQL RPCs with automatic 1-retry handling on transient `fetch failed` / `ECONNRESET`).
- **Configuration Cache:** [`lib/waConfigCache.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/lib/waConfigCache.js)
  - Caches `whatsapp_config` per user/phone in-memory (15s TTL) to prevent repeated queries to `whatsapp_config` on high-throughput webhook events.
- **Redis Connection:** [`lib/redisClient.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/lib/redisClient.js)
  - Manages standalone Redis client with 2.5s command timeouts. Used for leader locks, rate limiting, and cache invalidation.

### 2.3 API Routes & Database Interaction Map

| Route File | Base Path | Core Tables Accessed | Primary Operations |
| :--- | :--- | :--- | :--- |
| [`routes/webhook.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/webhook.js) | `/api/webhook` | `webhook_logs`, `messages`, `conversations`, `contacts`, `campaign_queue`, `leads`, `whatsapp_flow_submissions`, `flows` | Inbound message handling, media downloads, status update RPCs (`update_message_status`), queue updates, logging RPCs (`log_webhook_event`, `update_webhook_log`). |
| [`routes/inbox.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/inbox.js) | `/api/inbox` | `conversations`, `messages`, `contacts`, `leads`, `conversations_with_leads` | Conversation listing with exact counts, message history fetching, message sending, chat clearing. |
| [`routes/campaigns.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/campaigns.js) | `/api/campaigns` | `campaigns`, `campaign_queue`, `contacts`, `wallet_transactions`, `user_wallets` | Campaign creation, recipient staging, details polling with 8 status counts, queue pagination, report exports. |
| [`routes/dashboard.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/dashboard.js) | `/api/dashboard` | `tasks`, `follow_ups`, `whatsapp_templates`, `user_integrations`, `campaigns`, `pipelines`, `pipeline_stages`, `leads`, `contacts`, `flows`, `conversations`, `messages` | Massive dashboard aggregation pulling up to 10k rows across 12 tables in parallel on load. |
| [`routes/contacts.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/contacts.js) | `/api/contacts` | `contacts`, `contact_groups`, `leads`, `campaign_queue`, `whatsapp_flow_submissions` | CRUD, contact searches (ILIKE with trigram), chunked bulk import, chunked cascade deletion. |
| [`routes/settings.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/settings.js) | `/api/settings` | `branding_settings`, `whatsapp_config`, `profiles` | User preferences, branding configuration (TOAST table upsert), WhatsApp credentials. |
| [`routes/upload.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/routes/upload.js) | `/api/upload` | Supabase Storage (`campaign-media` bucket) | Media upload with Sharp JPEG compression for images; video validation for MP4. |

### 2.4 Background Workers & Schedulers

| Worker / Service | File Path | Polling Interval | Primary Database Operations |
| :--- | :--- | :--- | :--- |
| **Campaign Worker** | [`workers/campaignWorker.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/workers/campaignWorker.js) | 2s – 60s adaptive | `SELECT` candidates from `campaign_queue`, optimistic locking (`UPDATE status = 'SENDING'`), individual `UPDATE status = 'SENT'` per recipient, `log_outbound_message` RPC per recipient, `increment_campaign_success` RPC per recipient. |
| **Task Scheduler** | [`workers/scheduler.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/workers/scheduler.js) | 15s – 60s adaptive | Scans `tasks` for due/upcoming/overdue items and 5-min pre-alarms filtering `metadata->>pre_alarm_5m_sent`, updates `tasks.metadata`. |
| **Drip Engine** | [`workers/dripProcessor.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/workers/dripProcessor.js) | 60s (or 100ms when backlogged) | Calls RPC `fetch_and_lock_drip_jobs`, inserts into `drip_logs`, sends WhatsApp/Email. |
| **Sheet Automation** | [`services/SheetAutomationService.js`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/services/SheetAutomationService.js) | 60s interval | Queries `sheet_automations` where `status = 'active'`, syncs rows, inserts into `sheet_automation_history`. |

---

## 3. Frontend Architecture (`wazpp-client`)

### 3.1 Technology & Data Flow
- **Framework:** React 18 with Vite.
- **State & Data Access:** Primary data fetching is performed via standard `fetch()` and `axios` calls to backend `/api/*` endpoints. Direct client calls to Supabase PostgREST are restricted to only 2 instances (`whatsapp_config` lookup in `SettingsPanel.tsx:391` and `contact_groups` lookup in `WhatsAppApiSettings.tsx:207`).
- **Real-Time Layer:** Utilizes Socket.IO (`services/socketService.ts`) connected to Node.js backend. The Node.js backend broadcasts database updates via `realtimeSync.js` using Redis pub/sub.

### 3.2 Key Polling & Refetching Locations
- [`components/Campaigns/CampaignDetails.tsx:70-75`](file:///e:/code-from-haris/Wazpp/new/wazpp-client/src/components/Campaigns/CampaignDetails.tsx#L70-L75): Runs `setInterval` every **5,000 ms** (5 seconds) while visible, calling both `/api/campaigns/:id` and `/api/campaigns/:id/queue`.
- [`layouts/admin/AdminLayout.tsx:94`](file:///e:/code-from-haris/Wazpp/new/wazpp-client/src/layouts/admin/AdminLayout.tsx#L94): Runs `setInterval` every 60 seconds to refresh admin master stats.
- [`utils/reminderManager.ts:31`](file:///e:/code-from-haris/Wazpp/new/wazpp-client/src/utils/reminderManager.ts#L31): Runs `setInterval` every 5 seconds to check local notifications.

---

## 4. Database Schema & Migration Inventory

Located in [`wazpp-server/migrations/`](file:///e:/code-from-haris/Wazpp/new/wazpp-server/migrations/):
- `20261020_precheck_rpc_permissions.sql`: Permissions check script inspecting 29 public stored procedures (`log_outbound_message`, `update_message_status`, `log_webhook_event`, `update_webhook_log`, `fetch_and_lock_drip_jobs`, `adjust_wallet_balance`, etc.).
- `20261020_revoke_anon_rpc_permissions.sql`: Revokes public execution rights from `anon` on sensitive financial and message-logging procedures.
- `20261020_rollback_rpc_permissions.sql`: Rollback script for RPC privileges.
