---
name: integration-test
description: Plan, write and run the tests that show klesia-webapp and klesia-api work together over contracts/openapi.yaml - backend HTTP-pipeline tests, frontend service tests against the contract, and an optional live wire test through a contract proxy. Use for a cross-application feature, before merging a change to the HTTP boundary, or when asked to write or run integration or contract tests.
argument-hint: "[backend|frontend|live] (default: backend and frontend)"
---

# Integration test

Goal: prove that what the API sends is what the webapp expects, using the contract as the referee. The contract is `contracts/openapi.yaml`; tests never invent a request or response shape of their own. Run commands from each application's own folder (`klesia_api/`, `klesia_webapp/`); do not guess script names - read `package.json`, `Klesia.sln` and the test projects first, and report only commands you actually ran.

**Lot scoping.** Lot 1 = `GET /api/v1/demands`, `GET /health`, Entra authentication, CORS. `POST /api/v1/demands`, the "Nouvelle demande" modal and `POST /api/v1/notifications/negotiate` are not built in Lot 1: write their tests only when their code exists (their cases are listed below as Lot 2). Run `check-contract` first; a failing contract makes every other result meaningless.

## Level 1 - Backend HTTP pipeline against the contract

Where: `klesia_api/tests/IntegrationTests` (xUnit, FluentAssertions, NSubstitute; `WebApplicationFactory`, signed test tokens, in-memory repositories - see `klesia_api/CLAUDE.md`). No real Entra tenant, no database.

- Load the contract in the test and validate every response body against its schema (status, `Content-Type`, required fields, GK pattern, status enum). If no JSON-schema package is already in `Directory.Packages.props`, propose one to the user instead of adding it silently.
- Lot 1 cases (API specification section 9): AT-01 to AT-11 (401/403 matrix, empty list, several demands, no cross-user leakage, ignored `user_id`), AT-13 and AT-14 (CORS preflight allowed and refused), AT-12 and AT-19 (database down, stored non-HTTPS `bae_link`: sanitized `5xx`), AT-20 (no `oid`), AT-21 (every `401`/`403`/`5xx` is `application/problem+json` with `traceId`, plus `X-Correlation-Id`), AT-22 (`/health`).
- Both `403` causes (unknown user, inactive user, missing scope) return the identical status and body.
- Lot 2 cases: AT-23 to AT-30 (`POST /api/v1/demands`: `201` and body, `400`, `422`, `413`, `415`, `401/403`, duplicates allowed, nothing stored on failure) and AT-31, AT-32 (negotiate, preflight with `X-Requested-With` and `X-SignalR-User-Agent`, no credentials header). The SharePoint store is faked behind `IBaeDocumentStore`; also assert that the link written to the database is the one `GET` returns.
- Database constraint tests (AT-15, AT-16) need a real, disposable Azure Database for MySQL 8.4 database already migrated by `database/Klesia.Database`: run only when `KLESIA_TEST_DB_CONNECTION` is set, else report them as not run. Include the case-sensitive GK and status checks (for example `gk260902-144743112` must be rejected).

Run: `dotnet test` on the solution or the integration project (add a filter when the project defines one).

## Level 2 - Frontend services against the contract

Where: `klesia_webapp/tests/` (Vitest and Vue Test Utils; mock MSAL and HTTP at the service boundary, Appendix A of the webapp specification gives the file names).

- Build mock responses from the contract examples, or validate hand-written fixtures against the contract schemas (a small test helper reading `../contracts/openapi.yaml`), so a mock can never drift from the API.
- Lot 1 cases: AC-02 (GET with the API access token), AC-24 (malformed `200` body keeps the previous list), AC-19 and AC-25 (`403` access denied; `401` one recovery, no loop), AC-03/AC-04 (known and unknown status), AC-31 and AC-47 (unsafe `bae_link`), AC-49 (token sent only to the API origin and negotiate), AC-36 to AC-39 (SignalR: negotiate with `withCredentials: false`, reconnection, `demandUpdated` ignored payload). Verify that no test uses a field name absent from the contract.
- Lot 2 cases: AC-14 (multipart has only `gk_number` and `bae_file`, no `user_id`, no manual `Content-Type`), AC-15, AC-16, AC-26 to AC-28 (`201`, `400/422` keyed by `errors`, `413`, `5xx`, no automatic retry).

Run, from `klesia_webapp/`: `npm run type-check`, `npm run test:run` (and `npm run lint` when it exists).

## Level 3 - Live wire test (environment-dependent, optional)

Needs a dev API connected to a dev Azure Database for MySQL database (the developer's Entra account must be a database user and the client IP allowed by the server firewall), a real Entra test registration and a test user. Never run it against production, and never commit or log a token. The in-memory tests above do not need it; report Level 3 as "not run" when the prerequisites are missing.

1. Start the API in Development (`klesia_api/docs/local-development.md`; `http://localhost:5180`).
2. Put a contract proxy between the webapp and the API so every request and response is validated against the contract. With Stoplight Prism available: `npx @stoplight/prism-cli proxy contracts/openapi.yaml http://localhost:5180` (check `--help` for the exact flags first, in particular the one that turns violations into errors).
3. Start the webapp (`npm run dev`) with `VITE_API_BASE_URL` pointing to the proxy, sign in with the test user, and exercise the Lot 1 flow (list loads, `403` for an unknown user, CORS from `http://localhost:5173`). The dev simulator is not a substitute: it does not validate tokens.
4. Optionally run `schemathesis run contracts/openapi.yaml --url http://localhost:5180` with a test token to fuzz the API against the contract, if the tool is installed.
5. Collect the proxy's violation report; every violation is a finding.

## Rules for test data

Use invented values only: GK numbers like `GK260902-144743112`, example users, `https://contoso.sharepoint.com/...` links. No real GK number, document, tenant id, client id or token in a test, fixture, log or commit.

## Report

List per level: what was written, commands run with their outcome, the AT and AC identifiers covered and not covered, and anything that could not run (missing database, tenant, tool) with the reason. A contract change needed to make a test pass is a proposal for the user's approval, never a silent edit.
