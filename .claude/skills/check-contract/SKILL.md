---
name: check-contract
description: Verify that contracts/openapi.yaml is valid and that both specifications, the database project and both applications still agree with it. Use after any change to an HTTP route, schema, status code, header, auth rule or CORS rule, before finishing a cross-application task, or when asked to check, validate or audit the API contract. Report-only: never edits the contract or the code.
argument-hint: "[contract|specs|database|backend|frontend] (default: all)"
---

# Check contract

`contracts/openapi.yaml` is the machine-readable form of the HTTP boundary and wins on anything observable on the wire (root `CLAUDE.md`, `architecture-plan.md` section 1.4); a disagreement between it and a specification is a defect (GOV-01). This skill **checks and reports; it does not fix**. Every incompatibility found is listed for the user's approval first. Never edit the contract, a specification or code to make a check pass.

The two applications live in nested repositories: `klesia_api/` and `klesia_webapp/` (each has its own `.git`). Their specifications are `klesia_api/docs/specs/SPECIFICATIONS.md` and `klesia_webapp/docs/specs/SPECIFICATIONS.md`. Run commands from the workspace root. `$ARGUMENTS` limits the scope; with no argument run every step.

**No phasing.** The contract and both specifications describe the whole application. The contract has five operations (`getHealth`, `listDemands`, `getDemandDocument`, `createDemand`, `negotiateNotifications`) and each must exist in the code of the side that implements it. While the application is under construction, an operation that has no code yet is a NOTE (not implemented yet), not BREAKING: run Step 4 with `--skip-missing`. Code for a route that is not in the contract is always a finding.

## Step 1 - The contract itself

```
python .claude/skills/check-contract/scripts/validate_contract.py
```

(`py -3` on Windows if `python` is missing; needs `pyyaml` and `jsonschema`.) With `--model klesia_webapp/src/models/demand.ts` it also compares the hand-written webapp types with the contract schemas (skipped with a note while the file does not exist). It checks: OpenAPI 3.1, every `$ref` resolves, unique `operationId`, no `x-implementation-status` leftover (the contract carries no phasing), only `/health` anonymous, `401`/`403`/`500` documented on every protected operation, all error responses `application/problem+json`, every example valid against its schema, snake_case property names (except `NegotiateResponse` and `ProblemDetails`), the single GK pattern `^GK[0-9]{6}-[0-9]{9}$`, scope `access_as_user`, servers without `/api/v1`, `bae_link` is the API-path pattern (never a storage URL), `getDemandDocument` answers `application/pdf` and documents `404`. It also warns when the contract still mentions SharePoint, Container Apps, a Sprint or a Lot.

If `npx` is available also run `npx @redocly/cli lint contracts/openapi.yaml`. If a tool is missing or offline, say so in the report; do not install anything silently.

## Step 2 - Specifications vs contract

For each fact, read the contract and then both specifications (use `Grep`; do not rely on memory). A fact stated differently in any of the three is a finding.

| Fact (contract) | API spec | Webapp spec |
|---|---|---|
| Routes and methods (five operations) | 5.1, 5.3, 5.4, 7.2, 8.1 | 1.1, 6.1, 6.5, 7.3 |
| `GET` response fields and types | 5.1 | 6.1 |
| `gk_number` format, uniqueness, no normalization on the API | 3.3, 7.2 | NEW-03, NEW-06 |
| Status codes per operation (`404` only on the document route) | 4.3, 5.2, 5.4, 7.2 | 8 |
| Error body `application/problem+json`, `errors` keys | 5.2 | 8 |
| Scope `access_as_user`, token type (never ID token) | 4.1, DEC-11 | AUTH-02, VITE_API_SCOPE |
| CORS methods and request headers, no credentials | 5.2 | BE-06, RT-01 (`withCredentials: false`) |
| Base URL without `/api/v1` | - | 3.1 |
| Status enum `pending` / `ok` / `fail` and French labels | 3.3 | DEM-02 |
| A demand leaves `pending` exactly once; `ok` and `fail` are final | 3.3, DEC-19, 7.4 | DEM-02 |
| `bae_link` is the API path `/api/v1/demands/{demand_id}/bae`; `getDemandDocument` streams the PDF to its owner; `404` for a missing or foreign demand | 5.1, 5.4, DEC-24 | DEM-03, INT-06 |
| Negotiate response `{url, accessToken}`, hub `demands`, event `demandUpdated` without payload | 8.1, 8.2 | 6.4, 6.5 |
| Two notifications, same `demandUpdated` event: N-1 *demand stored* (after the commit, sent by `POST /api/v1/demands`) and N-2 *extraction completed* (final status `ok` or `fail`, sent by the extraction worker) | 8.2 (NOT-01 to NOT-05), 7.2, 7.3 | BE-04, RT-05 |
| Upload limit `Upload:MaxPdfSizeMb` equals `maxPdfSizeMb` | 6.1, INT-05 | 3.2 |

## Step 3 - Database vs contract (`klesia_api/database/Klesia.Database`)

The database is Azure SQL Database (DEC-14 of the API specification): tables `sec.Role`, `sec.User`, `ref.Status`, `dbo.Demand` (plus the two extraction tables), declared by the SQL project (`Schemas/`, `Tables/`) and seeded by `Scripts/*.sql` through `PostDeployment.sql`.

- `CK_Demand_GKNumber` accepts exactly the contract pattern **case-sensitively** (the model collation is case-insensitive, so the check uses an explicit binary collation, `Latin1_General_100_BIN2`) and the length is 18; `GKNumber` is `CHAR(18)`.
- `ref.Status` codes (`CK_Status_Code`, `Scripts/01_RefData.sql`) equal the `DemandStatus` enum, matched case-sensitively.
- `Demand.BaeLink` holds the blob URL without SAS (`CK_Demand_BaeLink` requires the `https://` prefix) and is **internal**: no query, DTO or log returns it; `bae_link` is built by the API from `Demand.Id`. `dbo.Demand.CreatedAt` exists (DEC-23).
- `sec.User.UserUniqueId` is a `UNIQUEIDENTIFIER` holding the Entra `oid` that `user_id` returns.
- Primary keys are `Id` and queries alias them to the domain names; `demand_id` is `Demand.Id`; list order is `Id DESC`.
- No definition or script refers to a table or column that no longer exists (for example `RoleId` on `sec.Role`), the seed scripts stay idempotent (merge on the stable `Code`, insert-only users) and are referenced only from `PostDeployment.sql`, and the project builds into a DACPAC.

## Step 4 - Backend vs contract (`klesia_api/`)

If the API builds and runs in Development, save its OpenAPI document (route per the app configuration, `/openapi/v1.json` by default) to a scratchpad file and run:

```
python .claude/skills/check-contract/scripts/validate_contract.py --actual <file>
```

(add `--skip-missing` while the API is under construction: routes without code are then warnings). It reports missing or extra routes and status codes and field/enum/required differences. If the API cannot be run, review the code instead:

- Routes are exactly the contract routes; no extra route; `/health` anonymous; every other route protected by the fallback policy.
- Response JSON names are declared explicitly on dedicated contract records (snake_case); no domain or database object is serialized.
- Errors are `ProblemDetails` for `400/401/403/404/413/415/422/5xx`, including the JWT challenge (`401` with `WWW-Authenticate: Bearer`) and identical bodies for every `403` cause; no stack trace, SQL, token or storage URL.
- CORS: exact configured origins, methods `GET, POST, OPTIONS`, request headers `Authorization, Content-Type, X-Requested-With, X-SignalR-User-Agent`, exposed `X-Correlation-Id`, max age 600, no credentials.
- Identity comes only from the validated token (`oid`); no `user_id` in any input.
- Document route: the demand is read by `Demand.Id` **and** the caller's user in one query; a missing and a foreign demand give the identical `404`; the PDF is streamed (never buffered whole, never written to disk) with `Cache-Control: private, no-store` and `X-Content-Type-Options: nosniff`; no storage URL, blob name or SAS appears in any response or log.
- Hosting (Azure App Service, API specification section 6.3): Always On, HTTPS Only, minimum TLS 1.2, Health check on `/health`, `AZURE_CLIENT_ID` for the user-assigned identity; no `Dockerfile`, container ingress, probe rule or forwarded-header workaround (they belonged to the withdrawn Container Apps decision); the platform may return its own non-JSON `502/503/504`, which the contract allows.

## Step 5 - Frontend vs contract (`klesia_webapp/`)

- Request URL = `VITE_API_BASE_URL` + the contract path; the Bearer token is the API access token (never the ID token) and is sent only to the API origin.
- Types: the webapp types are hand-written (`src/models/demand.ts`, webapp specification MODEL-01 to MODEL-03). Run `validate_contract.py --model` and read the file against `Demand`, `DemandStatus` and `UserDemands`: same field names, `integer` as `number`, `string` as `string`, every field required, `KnownDemandStatus` equal to the contract enum, `Demand.status` kept as `string`, `parseUserDemands` rules. A field missing on either side is BREAKING; any other DTO redefined outside this file is DRIFT. If a generated client is introduced later, regenerate it to a scratchpad file and diff instead.
- Error handling follows the status code (`401` one recovery, `403` access denied, `5xx` and network generic); server `title`/`detail` is never rendered.
- Fixtures, mocks and the dev simulator use the contract's field names and examples. The simulator is not proof of backend conformity.
- SignalR: `VITE_SIGNALR_HUB_URL` ends before `/negotiate`; `withCredentials: false`; the `demandUpdated` payload is ignored.
- BAE PDF: `bae_link` is used only when it matches `^/api/v1/demands/[1-9][0-9]*/bae$` and its id equals `demand_id`; the PDF is fetched with the Bearer token and shown through an object URL, never through `<a href>` or `window.open(bae_link)`; the token goes to the API origin only; `404` shows "Document introuvable.".
- Creation: multipart with exactly `gk_number` and `bae_file`, no `user_id`, no manual `Content-Type`, no automatic retry, and no list reload after a `2xx` (the two notifications update the list).

## Report

Start with a one-line verdict per step: PASS, FAIL or NOT RUN (with the reason). Then list findings, most severe first:

- **BREAKING** - an application would fail or misbehave against the other (renamed field, different status, pattern mismatch).
- **DRIFT** - two documents disagree but nothing breaks yet.
- **NOTE** - risk or missing coverage.

For each: where (file and line or section), what the contract says, what the other side says, smallest proposed fix, and whether the fix changes the contract (then it needs the user's approval and the root rule applies: the owning specification and the contract change together, then the code). End with what was **not** verified. Never claim a check ran unless you ran it.
