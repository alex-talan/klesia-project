# T-001 — Baseline contract and repository audit

**Date:** 2026-11-22
**Skill:** `check-contract` (report-only), all steps, with `--skip-missing` semantics (no application code exists yet)
**Scope:** `contracts/openapi.yaml` v0.5.0, `klesia_api/docs/specs/SPECIFICATIONS.md` v2.3, `klesia_webapp/docs/specs/SPECIFICATIONS.md` v2.1, `klesia_api/database/Klesia.Database`, both applications.

## Verdicts

| Step | Verdict | Notes |
|---|---|---|
| 1 — Contract | **PASS** | `validate_contract.py`: 5 operations, 0 errors, 0 warnings. `--model`: model file absent, skipped (NOTE). Redocly `lint` (recommended ruleset): valid, 6 warnings (see N-05). |
| 2 — Specifications vs contract | **PASS with DRIFT** | Every fact of the skill table agrees; wording drift in the webapp specification and webapp `CLAUDE.md` (D-02). |
| 3 — Database vs contract | **FAIL** | `dbo.Demand.CreatedAt` missing; `DemandInformation` and `DemandFormula` missing (B-01). Everything else agrees; DACPAC builds. |
| 4 — Backend vs contract | **NOT RUN (code absent)** | No C# source in `klesia_api/src` or `tests` (0 `.cs` files): every operation is *not implemented yet* (NOTE). Configuration drift found (D-01). |
| 5 — Frontend vs contract | **NOT RUN (code absent)** | `klesia_webapp/src` does not exist: every operation is *not implemented yet* (NOTE). |

## Commands run

| Command | Result |
|---|---|
| `py -3 …validate_contract.py` | Failed to start: the `py` launcher points to a missing Python 3.14 install. Re-run with `python` (3.13.15). |
| `python .claude/skills/check-contract/scripts/validate_contract.py` | `5 operations checked, 0 error(s), 0 warning(s)`, exit 0 |
| `python …validate_contract.py --model klesia_webapp/src/models/demand.ts` | `NOTE: model file not found, skipped`, exit 0 |
| `npx --yes @redocly/cli@latest lint contracts/openapi.yaml` | Valid; 6 warnings (`no-server-example.com` ×4, `info-license` ×1, `operation-4xx-response` ×1), exit 0 |
| `dotnet build database/Klesia.Database/Klesia.Database.sqlproj -c Debug` (in `klesia_api`) | Build succeeded, 0 warnings, 0 errors; `bin/Debug/Klesia.Database.dacpac` exists |
| Searches with `Select-String` on both specifications, webapp `CLAUDE.md`, `appsettings.json`, source folders | See findings |

## Findings

### BREAKING

| Id | Where | Contract / spec says | Other side says | Proposed fix | Contract change? | Mapped to |
|---|---|---|---|---|---|---|
| B-01 | `klesia_api/database/Klesia.Database/Tables/dbo/` | API spec SC-DB-01, DEC-23, §3.4: six tables; `dbo.Demand.CreatedAt DATETIME2(6)` with `DF_Demand_CreatedAt` (check-contract Step 3 requires it) | `Demand.sql` has no `CreatedAt`; `DemandInformation.sql` and `DemandFormula.sql` do not exist | Add the column and the two tables per §3.4 | No | [T-003](../T-003.md), [T-004](../T-004.md), [T-005](../T-005.md) |

(BREAKING for the extraction workflow and the recovery sweep; nothing on the wire breaks today because no code exists.)

### DRIFT

| Id | Where | Says | Should say | Proposed fix | Contract change? | Mapped to |
|---|---|---|---|---|---|---|
| D-01 | `klesia_api/src/WebApi/appsettings.json` lines 14–24 | `Azure:Storage:BlobServiceUri`, plus `Azure:AppConfiguration`, `Azure:KeyVault`, `Azure:ServiceBus` | API spec §6.1: `Storage:BlobServiceUri`, `Storage:ContainerName`; no App Configuration, Key Vault or Service Bus key | Align with §6.1 | No | [T-007](../T-007.md), [ARB-023](../ARB-023.md) |
| D-02 | `klesia_webapp/docs/specs/SPECIFICATIONS.md` line 163 (AUTH-03); `klesia_webapp/CLAUDE.md` line 36 (rule 1) and line 9 | "Bearer on **both** protected REST operations" / "access token to **both** API calls"; "Appendix A (section 13)" | The contract has **three** protected REST operations (`listDemands`, `createDemand`, `getDemandDocument`; DEM-03 already requires the Bearer on the third); Appendix A is section 12 | Fix the wording (no behaviour change) | No | [T-090](../T-090.md) (new) |
| D-03 | `klesia_webapp/docs/TASKS.md` | Referenced as the work tracker by `architecture-plan.md` §1.2/Appendix A and `klesia_webapp/CLAUDE.md` | Does not exist | Point references to `roadmap/` | No | [ARB-001](../ARB-001.md), [T-002](../T-002.md) |

### NOTE

| Id | Observation | Mapped to |
|---|---|---|
| N-01 | API spec §4.1 allows an anonymous OpenAPI document in Development; the contract lists `/health` as the only anonymous route and calls itself the complete HTTP surface; the `check-contract` skill says code for a route not in the contract is always a finding. Tooling route vs governance rule to clarify. | [ARB-025](../ARB-025.md) (new, non-blocking), [T-010](../T-010.md), [T-074](../T-074.md) |
| N-02 | Contract `ProblemDetails` examples show `traceId` in W3C `traceparent` format (`00-…-…-00`); the specifications only require `traceId` = `X-Correlation-Id`, with no format. T-014 may choose the format (string); no conflict. | [T-014](../T-014.md) |
| N-03 | `Scripts/02_User.sql` seeds one named person (Entra `oid` and e-mail) in every environment (DEC-05, explicitly requested). | [ARB-019](../ARB-019.md), [T-084](../T-084.md) |
| N-04 | No backend or frontend code exists (0 `.cs` files; no `klesia_webapp/src`); `klesia_webapp/package.json` `type-check` references a missing `tsconfig.app.json`. All five operations: *not implemented yet*. | Backend T-007…T-046, frontend [T-047](../T-047.md)… |
| N-05 | Redocly recommended-ruleset warnings: four `servers` on `localhost`/`example.com` (placeholders, replaced once hosts are known); no `info.license`; `/health` has no `4XX` response (by design, API spec §5.3: `200`/`503` only). Only the server hosts need action; the two others contradict no source and are accepted as is. | Server hosts: [ARB-009](../ARB-009.md) (question 4) |
| N-06 | The `py -3` launcher on this workstation points to a missing Python 3.14; the skill's Windows hint (`py -3`) fails here, `python` works. Environment only. | — |

## Database checks (Step 3) that passed

`CK_Demand_GKNumber` binary-collation pattern = contract pattern, `GKNumber CHAR(18)`; `CK_Status_Code` and `01_RefData.sql` codes = `DemandStatus` enum (`pending`, `ok`, `fail`, case-sensitive); `CK_Demand_BaeLink` `https://` prefix; `UserUniqueId UNIQUEIDENTIFIER`; primary keys `Id`; `IX_Demand_UserId (UserId, Id DESC)`; seeds idempotent (merge on `Code`, users insert-only) and referenced only from `PostDeployment.sql`; DACPAC builds.

## Not verified

Backend and frontend steps (no code); Step 4 `--actual` (no running API); `--model` (no model file); live behaviour of anything.
