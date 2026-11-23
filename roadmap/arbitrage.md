# KLESIA — Arbitrages (decisions and open questions)

**Created:** 2026-11-21
**Companion:** [`roadmap.md`](roadmap.md)

This document lists every decision that must be settled and every open question that must be answered for the implementation of KLESIA. Each arbitrage has a file `ARB-xxx.md` in this folder. Most of them come from the `OPEN-*` and `INT-*` items and from the `ASSUMPTION`s of the specifications; the others were found while preparing the roadmap.

## 1. Legend

| Column | Meaning |
|---|---|
| Arbitrage number | `ARB-xxx`, link to its file. |
| Subject | Short title. |
| Description | What must be decided or answered. |
| Blocking | `YES`: at least one task lists it in its *Dependencies* and cannot start before it is `SETTLED`. `NO`: a documented provisional default (usually a specification `ASSUMPTION`) lets every concerned task proceed; the decision is still required before production (T-090) and, if it differs from the default, it creates follow-up tasks. |
| Concerning tasks | Tasks affected by the decision (gating or not). `ALL` = the whole roadmap; empty = none yet. |
| Status | `OPEN` (to decide) or `SETTLED` (decided, recorded in the *Decision* section of the file). |
| To be decided by | Who is expected to decide (informative). |

## 2. Source of truth and synchronisation

- The header table of each `ARB-xxx.md` is **authoritative** (fields `Number`, `Subject`, `Description`, `Blocking`, `Concerning tasks`, `Status`). The table below is a synchronised copy.
- To settle an arbitrage, run **`/settle-arbitrage ARB-xxx`** (`.claude/skills/settle-arbitrage`): it asks the questions, then fills the *Decision* section of its file (decision, date, decided by, follow-up), sets `Status` to `SETTLED` in the header, (1) copies the header into the table below, (2) in [`roadmap.md`](roadmap.md), moves each concerned task from `BLOCKED` to `TODO` when all its other dependencies allow it.
- New arbitrages raised while implementing a task (`/implement-task T-xxx`) are appended at the end of the table below, each with its `ARB-xxx.md` file.
- If the decision differs from the documented default, add the follow-up tasks to the roadmap (new numbers at the end of the table, with their dependencies) and list them in the *Decision* section.
- A decision that changes the HTTP boundary follows the contract-first rule: owning specification, then `contracts/openapi.yaml`, in the same change (root `CLAUDE.md`, GOV-01).

## 3. Arbitrage table

| Arbitrage number | Subject | Description | Blocking | Concerning tasks | Status | To be decided by |
|---|---|---|---|---|---|---|
| [ARB-001](ARB-001.md) | Work-tracking source of truth | Confirm that `roadmap/` replaces `klesia_webapp/docs/TASKS.md` (referenced but missing) | YES | `T-002` | SETTLED | Project owner |
| [ARB-002](ARB-002.md) | Maximum PDF size and request limits | Real PDF limit aligned across SPA, API, App Service host, platform timeout and Document Intelligence (INT-05, OPEN-08f, OPEN-09d) | NO | `T-009`, `T-032`, `T-036`, `T-041`, `T-044`, `T-058`, `T-083`, `T-090` | OPEN | Client + DSI |
| [ARB-003](ARB-003.md) | JSON-schema library for backend contract tests | Which package validates API responses against `contracts/openapi.yaml` in xUnit tests | YES | `T-023`, `T-066` | SETTLED | Tech lead |
| [ARB-004](ARB-004.md) | Server-side Azure SignalR integration | Management SDK or raw REST, Azure role of the identity, access keys in dev or managed identity only | YES | `T-030`, `T-031`, `T-058`, `T-079`, `T-080` | SETTLED | Architect TALAN |
| [ARB-005](ARB-005.md) | Document Intelligence model and answer envelope | Model id, API version, SDK, and whether the answer is EXT-04 JSON as is or an `AnalyzeResult` to map (OPEN-09a/b) | YES | `T-059`, `T-065` | OPEN | Document Intelligence owners |
| [ARB-006](ARB-006.md) | Required extracted fields | Which EXT-04 fields the model always returns (drives `NOT NULL` columns and EXT-05) (OPEN-09c) | NO | `T-004`, `T-060`, `T-065` | OPEN | Document Intelligence owners + client |
| [ARB-007](ARB-007.md) | Document Intelligence limits, quota and timeout | Page/size limits, throttling quota, call timeout, worker concurrency (OPEN-09d/e) | NO | `T-009`, `T-063`, `T-065` | OPEN | Document Intelligence owners |
| [ARB-008](ARB-008.md) | Durability of background work | Keep the in-process queue + recovery sweep, or Azure Service Bus behind `IExtractionQueue` (OPEN-10) | NO | `T-034`, `T-063`, `T-064` | OPEN | DSI + tech lead |
| [ARB-009](ARB-009.md) | Environments, subscriptions and naming | List of environments (dev, staging, prod), subscriptions, resource groups, regions, naming convention | YES | `T-079`, `T-080`, `T-081`, `T-082`, `T-083`, `T-084`, `T-086`, `T-087` | OPEN | DSI |
| [ARB-010](ARB-010.md) | Infrastructure-as-code ownership | Who writes and deploys the Bicep (DSI or these repositories), where it lives, who assigns roles (OPEN-08g) | YES | `T-079`, `T-080`, `T-081`, `T-082`, `T-083`, `T-084` | OPEN | DSI |
| [ARB-011](ARB-011.md) | App Service hosting settings | Plan, OS, tier, region, Always On, slots, instances, autoscale, Health check reaction (OPEN-08a/d/e) | YES | `T-083`, `T-086`, `T-088` | OPEN | DSI |
| [ARB-012](ARB-012.md) | Network topology and inbound exposure | Public + firewall or VNet + private endpoints for SQL and storage; inbound exposure, domain, certificate (OPEN-07b, OPEN-08b/c, OPEN-12a) | YES | `T-081`, `T-082`, `T-083` | OPEN | DSI |
| [ARB-013](ARB-013.md) | Azure SQL Database settings | Server, database, tier, region, Entra admin, Entra-only auth, database users, TLS (OPEN-07) | YES | `T-082`, `T-086` | OPEN | DSI |
| [ARB-014](ARB-014.md) | Storage account settings | Redundancy, soft delete, retention, lifecycle, one account per environment (OPEN-12c/d) | YES | `T-081` | OPEN | DSI |
| [ARB-015](ARB-015.md) | Managed identity and role assignments | One user-assigned identity and its roles on SQL, storage, Document Intelligence, SignalR (OPEN-11, HOST-04) | NO | `T-080`, `T-088` | OPEN | DSI |
| [ARB-016](ARB-016.md) | Observability backend | Log Analytics workspace, Application Insights or console + platform logs only (OPEN-08h, HOST-09) | NO | `T-014`, `T-083` | OPEN | DSI |
| [ARB-017](ARB-017.md) | CI/CD authentication and governance | GitHub OIDC, environments, approvals, deployment principals, branch protection (OPEN-08i) | YES | `T-086`, `T-087` | OPEN | DSI + tech lead |
| [ARB-018](ARB-018.md) | Static Web Apps hosting and deployment | SWA plan, region, custom domain, deployment authentication, per-environment builds and origins | YES | `T-069`, `T-084`, `T-087` | OPEN | DSI |
| [ARB-019](ARB-019.md) | Allowlist population per environment | Which users are seeded in each environment and how `02_User.sql` varies by environment | YES | `T-085`, `T-090` | OPEN | Client + project owner |
| [ARB-020](ARB-020.md) | Data retention and personal data (GDPR) | Retention of PDFs and extracted data, deletion procedure, legal basis | NO | `T-081`, `T-090` | OPEN | Client (DPO) |
| [ARB-021](ARB-021.md) | CSP compatibility with the PDF viewer | Whether the strict CSP lets browsers display a PDF object URL in a new tab (AC-57, DEP-05) | NO | `T-069`, `T-089` | OPEN | Tech lead (after test) |
| [ARB-022](ARB-022.md) | End-to-end test scope | Automated E2E smoke test (Playwright) with an Entra test tenant, or manual live test only | NO | `T-058`, `T-078` | OPEN | Tech lead + client |
| [ARB-023](ARB-023.md) | Azure App Configuration and Key Vault adoption | Keep or remove the speculative `Azure:*` sections of `appsettings.json` | NO | `T-007` | OPEN | Tech lead |
| [ARB-024](ARB-024.md) | Node.js version for builds | Node 26 LTS as specified vs the version installed locally and on CI/SWA build images | NO | `T-037`, `T-038`, `T-087` | OPEN | Tech lead |
| [ARB-025](ARB-025.md) | Development-only OpenAPI document route vs the contract surface | Decide how the anonymous OpenAPI document served in Development relates to the contract, which declares `/health` as the only anonymous route | NO | `T-010`, `T-075` | OPEN | Tech lead |
| [ARB-026](ARB-026.md) | Application-layer logging for demand creation | Decide how demand creation logs a best-effort blob cleanup failure without exposing the blob URL or adding an unapproved dependency | YES | `T-035`, `T-093` | SETTLED | Architect TALAN |

## 4. Change log

| Date | Change |
|---|---|
| 2026-11-21 | 24 arbitrages created, all `OPEN`. |
| 2026-11-22 | ARB-025 created by T-001 (baseline audit, finding N-01). |
| 2026-11-22 | ARB-001 settled: `roadmap/` is the single work tracker. |
| 2026-11-23 | ARB-003 settled: approve `JsonSchema.Net` + `YamlDotNet` for backend contract tests; T-023 unblocked. |
| 2026-11-23 | ARB-004 settled: Management SDK + `DefaultAzureCredential`, SignalR Service Owner, developer Entra identity locally; T-030 unblocked. |
| 2026-11-23 | ARB-026 created during T-035: decide the Application-layer logging boundary for sanitized cleanup-failure logging. |
| 2026-11-23 | ARB-026 settled: approve `Microsoft.Extensions.Logging.Abstractions` for Application; T-093 created as the prerequisite for T-035. |
