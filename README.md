# Wazpp Project — Supabase Disk I/O & Database Workload Root Cause Audit

## Executive Summary

- **Target System:** Wazpp Clean Workspace (`e:/code-from-haris/Wazpp/new`)
- **Supabase Project Reference:** `wsthuj safdesakmpvoez` (Pending configuration confirmation: `wsthuj...`)
- **Incident Trigger:** Supabase Disk I/O Budget Depletion Warning (Risk of database throttling, high latency, I/O wait, instance unresponsiveness).
- **Audit Methodology:** 100% Read-Only inspection of codebase, background workers, scheduled polling, webhook lifecycles, and production telemetry (`pg_stat_statements` extract, table relation sizes, and cached egress traffic).
- **Core Conclusion:** The Disk I/O exhaustion is **NOT** caused by high table read volume or missing search indexes; it is primarily driven by an **extreme volume of high-frequency unbatched writes**, **duplicate worker execution across PM2 cluster instances**, **per-status webhook write churn**, **unbounded webhook logging without retention**, and **heavy TOAST/temp-file activity on small configuration tables**.

---

## Key Metrics at a Glance (From Supabase Telemetry)

| Component / Relation | Metrics from Production (`pg_stat_statements` & Table Sizes) | Code Source & Root Cause Driver |
| :--- | :--- | :--- |
| **`public.log_outbound_message`** | **~98,000 calls**, 92 MiB blocks written, **2.28 GiB dirtied** | Single-row inserts into `messages` + redundant `conversations` updates per campaign send (`whatsappService.js:545-570`, `campaignWorker.js:1800`). |
| **`public.update_message_status`** | **~182,000 calls**, 48 MiB written | Webhook processes status updates individually (`webhook.js:900`). Double-calls on race conditions (`webhook.js:912`). |
| **`public.webhook_logs`** | **103 MB Total Size** (60 MB data, 43 MB indexes, ~43,076 rows) | Dual writes per webhook: `log_webhook_event` (48 MiB) + `update_webhook_log` (54 MiB) with no automated pruning. |
| **`public.conversations`** | **46 MB Total Size** (Table data is only 4.2 MB; **Indexes = 41.8 MB / 10x bloat**) | Every incoming/outgoing message executes an `UPDATE conversations` for `last_message`, causing index fragmentation and HOT update failure. |
| **`public.branding_settings`** | **16 MB Total Size** for only **18 rows** (Table data: 8 KB; TOAST: 16 MB; **126 MiB temp written**) | Huge JSONB / base64 strings stored in `theme_config` via single upserts (`settings.js:540`), causing Postgres TOAST churn and temporary file thrashing. |
| **Cached Egress Spike** | **15.811 GB** peak on Oct 22; sustained 6–12 GB daily | WhatsApp campaign media fetched repeatedly from `campaign-media` storage bucket by Meta CDN and recipients. |

---

## High-Level Findings by Severity

| Severity Level | Count | Summary of Key Drivers |
| :---: | :---: | :--- |
| **P0 — Critical** | 3 | **1.** Unbatched write storm in campaign/message logging (~98k outbound + ~182k status writes dirtied >2.28 GB).<br>**2.** PM2 Cluster Mode running background workers in duplicate when Redis leader lock fails open.<br>**3.** In-memory non-shared webhook deduplication clearing every 10 min without cross-process sync. |
| **P1 — High** | 4 | **4.** Dual synchronous logging writes per webhook in `webhook_logs` (103 MB bloat, ~100 MiB written).<br>**5.** Campaign Details polling (`CampaignDetails.tsx`): 8 unindexed `COUNT(*)` queries with `ILIKE` every 5 seconds.<br>**6.** Exact duplicate indexes on `messages` (`idx_messages_conversation_created` vs `idx_messages_conversation_id_created_at`) and `contacts` (`idx_contacts_user_phone` vs `idx_contacts_user_phone_lookup`).<br>**7.** TOAST temp-file thrashing on `branding_settings` (16 MB TOAST for 18 rows; 126 MiB temp writes in 72 calls). |
| **P2 — Medium** | 3 | **8.** Dashboard query fanout (12+ parallel table queries pulling up to 10,000 rows on every page view).<br>**9.** `tasks` reminder scheduler polling every 15–60s filtering unindexed JSON extraction (`metadata->>pre_alarm_5m_sent`).<br>**10.** Storage cache-control inconsistency on inbound media (`3600` vs `31536000`). |
| **P3 — Low** | 2 | **11.** Missing email trigram index (confirmed low-impact by `pg_stat_statements`).<br>**12.** Lack of structured query latency metric tagging in API logs. |

---

## Directory Structure of Audit Deliverables

The full audit report is divided into seven focused technical documents located under `docs/audits/supabase-disk-io/`:

1. [**`01-codebase-inventory.md`**](file:///e:/code-from-haris/Wazpp/new/docs/audits/supabase-disk-io/01-codebase-inventory.md) — Comprehensive inventory of applications, background workers, webhook handlers, schedulers, and database access modules.
2. [**`02-database-query-audit.md`**](file:///e:/code-from-haris/Wazpp/new/docs/audits/supabase-disk-io/02-database-query-audit.md) — Detailed code audit of high-frequency queries, unbatched loops, and `.count('exact')` fanouts.
3. [**`03-whatsapp-processing-audit.md`**](file:///e:/code-from-haris/Wazpp/new/docs/audits/supabase-disk-io/03-whatsapp-processing-audit.md) — Webhook lifecycle, idempotency gaps, status update write storms, and media storage egress.
4. [**`04-runtime-and-frontend-audit.md`**](file:///e:/code-from-haris/Wazpp/new/docs/audits/supabase-disk-io/04-runtime-and-frontend-audit.md) — PM2 cluster worker behavior, `leaderLock` fail-open vulnerability, and frontend polling loops.
5. [**`05-supabase-investigation-plan.md`**](file:///e:/code-from-haris/Wazpp/new/docs/audits/supabase-disk-io/05-supabase-investigation-plan.md) — Safe, read-only SQL diagnostic queries for production (`pg_stat_statements`, index hit ratios, TOAST analysis).
6. [**`06-prioritized-remediation-plan.md`**](file:///e:/code-from-haris/Wazpp/new/docs/audits/supabase-disk-io/06-prioritized-remediation-plan.md) — Step-by-step remediation guide ordered by P0–P3 with risk mitigation and zero downtime rollback plans.
7. [**`07-validation-checklist.md`**](file:///e:/code-from-haris/Wazpp/new/docs/audits/supabase-disk-io/07-validation-checklist.md) — Quantitative verification checklist, test scenarios, before-and-after baseline metrics.

---

## Confirmation of Audit Constraints

- **Read-Only:** No application code, database schemas, package manifests, or environment files were modified during this audit.
- **No Secrets Exposed:** All credentials, JWT keys, tokens, and database passwords were strictly excluded from all reports.
- **Evidence-Based:** Every finding is cross-referenced with exact source file lines and verified against the provided Supabase production telemetry.
