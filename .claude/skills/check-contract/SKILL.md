---
name: check-contract
description: Verify that contracts/openapi.yaml is valid and that both specifications, the database project and both applications still agree with it. Use after any change to an HTTP route, schema, status code, header, auth rule or CORS rule, before finishing a cross-application task, or when asked to check, validate or audit the API contract. Report-only: never edits the contract or the code.
argument-hint: "[contract|specs|database|backend|frontend] (default: all)"
---

# Check contract

`contracts/openapi.yaml` is the single source of truth for the HTTP boundary (root `CLAUDE.md`). This skill **checks and reports; it does not fix**. Every incompatibility found is listed for the user's approval first. Never edit the contract, a specification or code to make a check pass.

The two applications live in nested repositories: `klesia_api/` and `klesia_webapp/` (each has its own `.git`). Their specifications are `klesia_api/docs/specs/SPECIFICATIONS.md` and `klesia_webapp/docs/specs/SPECIFICATIONS.md`. Run commands from the workspace root. `$ARGUMENTS` limits the scope; with no argument run every step.

**Lot scoping.** Each operation carries `x-implementation-status`. In Lot 1 only `sprint-1` operations (`getHealth`, `listDemands`) must exist in code. `adopted` operations (`createDemand`, `negotiateNotifications`) are checked in the contract, the specifications and the database, and in code only once code for them exists. Their absence from code in Lot 1 is **not** a finding; code for them in Lot 1 is (the webapp has no modal and no POST in Lot 1, and the backend adds no placeholder routes).

## Step 1 - The contract itself

```
python .claude/skills/check-contract/scripts/validate_contract.py
```

(`py -3` on Windows if `python` is missing; needs `pyyaml` and `jsonschema`.) It checks: OpenAPI 3.1, every `$ref` resolves, unique `operationId`, `x-implementation-status` on every operation, only `/health` anonymous, `401`/`403`/`500` documented on every protected operation, all error responses `application/problem+json`, every example valid against its schema, snake_case property names (except `NegotiateResponse` and `ProblemDetails`), the single GK pattern `^GK[0-9]{6}-[0-9]{9}$`, scope `access_as_user`, servers without `/api/v1`.

If `npx` is available also run `npx @redocly/cli lint contracts/openapi.yaml`. If a tool is missing or offline, say so in the report; do not install anything silently.

## Step 2 - Specifications vs contract

For each fact, read the contract and then both specifications (use `Grep`; do not rely on memory). A fact stated differently in any of the three is a finding.

| Fact (contract) | API spec | Webapp spec |
|---|---|---|
| Routes, methods, lot status | 5.1, 5.3, 7.1, 7.3 | 1.1, 6.1, 6.5, 7.3 |
| `GET` response fields and types | 5.1 | 6.1 |
| `gk_number` format, uniqueness, no normalization on the API | 3.3, 7.3 | NEW-03, NEW-06 |
| Status codes per operation | 4.3, 5.2, 7 | 8 |
| Error body `application/problem+json`, `errors` keys | 5.2 | 8 |
| Scope `access_as_user`, token type (never ID token) | 4.1, DEC-11 | AUTH-02, VITE_API_SCOPE |
| CORS methods and request headers, no credentials | 5.2 | BE-06, RT-01 (`withCredentials: false`) |
| Base URL without `/api/v1` | - | 3.1 |
| Status enum `pending` / `ok` / `fail` and French labels | 3.3 | DEM-02 |
| `bae_link` permanent SharePoint HTTPS URL | OPEN-05, DEC-13 | DEM-03, INT-06 |
| Negotiate response `{url, accessToken}`, hub `demands`, event `demandUpdated` without payload | 7.1, 7.2 | 6.4, 6.5 |
| Upload limit `Upload:MaxPdfSizeMb` equals `maxPdfSizeMb` | 6.1, INT-05 | 3.2 |

## Step 3 - Database vs contract (`klesia_api/database/Klesia.Database`)

The database is Azure Database for MySQL (DEC-14 of the API specification): tables `sec_Role`, `sec_User`, `ref_Status`, `Demand`, created by `Migrations/*.sql` and seeded by `Seeds/*.sql`.

- `CK_Demand_GKNumber` accepts exactly the contract pattern **case-sensitively** (the default collation `utf8mb4_0900_ai_ci` ignores case, so the regex needs the `'c'` match type) and the length is 18; `GKNumber` is `CHAR(18)`.
- `ref_Status` codes (`CK_Status_Code`, `Seeds/01_RefData.sql`) equal the `DemandStatus` enum, matched case-sensitively.
- `CK_Demand_BaeLink` requires `https://`, matching `bae_link`. In MySQL `LIKE` ignores case, so the API must still validate the URL itself.
- `sec_User.UserUniqueId` holds the lower-case canonical GUID that `user_id` returns.
- Primary keys are `Id` and queries alias them to the domain names; `demand_id` is `Demand.Id`; list order is `Id DESC`.
- No script refers to a table or column that no longer exists (T-SQL leftovers, `RoleId` on `sec_Role`, old `sec.`/`ref.`/`dbo.` names), and no SQL comment contains a semicolon (the DbUp migrator splits statements on it).

## Step 4 - Backend vs contract (`klesia_api/`)

If the API builds and runs in Development, save its OpenAPI document (route per the app configuration, `/openapi/v1.json` by default) to a scratchpad file and run:

```
python .claude/skills/check-contract/scripts/validate_contract.py --actual <file>
```

(add `--all` once `adopted` operations are implemented). It reports missing or extra routes and status codes and field/enum/required differences. If the API cannot be run, review the code instead:

- Routes are exactly the contract routes; no extra route; `/health` anonymous; every other route protected by the fallback policy.
- Response JSON names are declared explicitly on dedicated contract records (snake_case); no domain or database object is serialized.
- Errors are `ProblemDetails` for `400/401/403/413/415/422/5xx`, including the JWT challenge (`401` with `WWW-Authenticate: Bearer`) and identical bodies for every `403` cause; no stack trace, SQL, token or document URL.
- CORS: exact configured origins, methods `GET, POST, OPTIONS`, request headers `Authorization, Content-Type, X-Requested-With, X-SignalR-User-Agent`, exposed `X-Correlation-Id`, max age 600, no credentials.
- Identity comes only from the validated token (`oid`); no `user_id` in any input.

## Step 5 - Frontend vs contract (`klesia_webapp/`)

- Request URL = `VITE_API_BASE_URL` + the contract path; the Bearer token is the API access token (never the ID token) and is sent only to the API origin.
- Types: if generated types exist (for example with `openapi-typescript`), regenerate to a scratchpad file and diff against the committed file; generated files are never hand-edited. If models are hand-written, compare them field by field with the contract schemas. Flag any DTO redefined independently of the contract.
- Error handling follows the status code (`401` one recovery, `403` access denied, `5xx` and network generic); server `title`/`detail` is never rendered.
- Fixtures, mocks and the dev simulator use the contract's field names and examples. The simulator is not proof of backend conformity.
- SignalR: `VITE_SIGNALR_HUB_URL` ends before `/negotiate`; `withCredentials: false`; the `demandUpdated` payload is ignored.
- Lot 2 only (do not expect it in Lot 1): multipart with exactly `gk_number` and `bae_file`, no `user_id`, no manual `Content-Type`, no automatic retry.

## Report

Start with a one-line verdict per step: PASS, FAIL or NOT RUN (with the reason). Then list findings, most severe first:

- **BREAKING** - an application would fail or misbehave against the other (renamed field, different status, pattern mismatch).
- **DRIFT** - two documents disagree but nothing breaks yet.
- **NOTE** - risk or missing coverage.

For each: where (file and line or section), what the contract says, what the other side says, smallest proposed fix, and whether the fix changes the contract (then it needs the user's approval and the root rule applies: contract first, then the specifications, then code). End with what was **not** verified. Never claim a check ran unless you ran it.
