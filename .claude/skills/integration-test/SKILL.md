---
name: integration-test
description: Plan, write and run the tests that show klesia-webapp and klesia-api work together over contracts/openapi.yaml - backend HTTP-pipeline tests, frontend service tests against the contract, and an optional live wire test through a contract proxy. Use for a cross-application feature, before merging a change to the HTTP boundary, or when asked to write or run integration or contract tests.
argument-hint: "[backend|frontend|live] (default: backend and frontend)"
---

# Integration test

Goal: prove that what the API sends is what the webapp expects, using the contract as the referee. The contract is `contracts/openapi.yaml`; tests never invent a request or response shape of their own. Run commands from each application's own folder (`klesia_api/`, `klesia_webapp/`); do not guess script names - read `package.json`, `Klesia.sln` and the test projects first, and report only commands you actually ran.

**No phasing.** The contract describes the whole application (five operations). Write tests for the code that exists and report a case whose code is not written yet as "not written", never as a pass. Run `check-contract` first; a failing contract makes every other result meaningless.

## Level 1 - Backend HTTP pipeline against the contract

Where: `klesia_api/tests/IntegrationTests` (xUnit, FluentAssertions, NSubstitute; `WebApplicationFactory`, signed test tokens, in-memory repositories - see `klesia_api/CLAUDE.md`). No real Entra tenant, no database.

- Load the contract in the test and validate every response body against its schema (status, `Content-Type`, required fields, GK pattern, status enum). If no JSON-schema package is already in `Directory.Packages.props`, propose one to the user instead of adding it silently.
- Read and authorization cases (API specification section 10): AT-01 to AT-11 (401/403 matrix, empty list, several demands, `bae_link` equal to the API path, no cross-user leakage, ignored `user_id`), AT-12, AT-20, AT-21 (every `401`/`403`/`5xx` is `application/problem+json` with `traceId`, plus `X-Correlation-Id`), AT-22 (`/health`), AT-13, AT-14 and AT-59 (CORS preflight allowed and refused, document route included).
- Creation cases: AT-23 to AT-30, AT-39 (`201` returned without waiting for the extraction, `status: pending`, exactly one N-1 after the commit), AT-48 (storage unavailable, then a failing insert after a successful upload), AT-50 (a client-supplied `user_id` is refused) and AT-56 (two submissions give two blob names). The blob store is faked behind `IBaeDocumentStore`; assert that the blob reference written to the database is never returned and that `bae_link` is the API path.
- Document cases: AT-51 (`200`, `application/pdf`, exact bytes, `Content-Disposition: inline`, `Cache-Control: private, no-store`, `X-Content-Type-Options: nosniff`), AT-52 (`404` identical for another user's demand, an unknown id and `abc`; the blob store is not called for the first two), AT-53 (401/403), AT-54 (`500` for a missing blob, `503` for unavailable storage), AT-55 (every `bae_link` is exactly `/api/v1/demands/{demand_id}/bae`), AT-58 (streamed, cancellation stops the read) and AT-19 (an invalid stored blob reference is a sanitized `500`).
- Real-time cases: AT-18, AT-31 (negotiate returns `{url, accessToken}`, `userId` = `oid`), AT-32 (preflight with `X-Requested-With` and `X-SignalR-User-Agent`, no credentials header), AT-37 (the notifier sends `demandUpdated` with no arguments to the owner's `oid` only, SignalR client faked) and AT-38 (a failing SignalR service never throws into the caller). N-1 comes from `POST /api/v1/demands`, N-2 from the extraction worker: at the boundary AT-40 and AT-41 only require that the next `GET` shows `ok` or `fail`; the extraction rules (AT-42 to AT-47) belong to the API's own tests.
- Hosting cases (API specification section 6.3): AT-36 (a request forwarded over HTTPS by the App Service front end: no redirect loop, treated as HTTPS; test host), AT-34 (`/health` answers `503` while the database is unreachable), AT-35 (production App Service settings: Always On, HTTPS Only, TLS 1.2, Health check, `AZURE_CLIENT_ID`) and AT-33 (deployed dev app with the managed identity). AT-33 and AT-35 need the dev Azure environment: report them as not run otherwise, and never claim a deployment was verified without running it.
- Both `403` causes (unknown user, inactive user, missing scope) return the identical status and body.
- Database and storage constraint tests: AT-15, AT-16, AT-17 and AT-49 need a real, disposable Azure SQL database published from `database/Klesia.Database` (DACPAC): run only when `KLESIA_TEST_SQL_CONNECTION` is set, else report them as not run (include the case-sensitive GK and status checks, for example `gk260902-144743112` must be rejected). AT-57 (private container, shared-key access refused) needs a disposable dev storage account.

Run: `dotnet test` on the solution or the integration project (add a filter when the project defines one).

## Level 2 - Frontend services against the contract

Where: `klesia_webapp/tests/` (Vitest and Vue Test Utils; mock MSAL and HTTP at the service boundary, Appendix A of the webapp specification gives the file names).

- Build mock responses from the contract examples, or validate hand-written fixtures against the contract schemas (a small test helper reading `../contracts/openapi.yaml`), so a mock can never drift from the API. Also test `src/models/demand.ts` (`parseUserDemands`, MODEL-02) with the contract examples (valid) and mutated copies (missing field, wrong type, `demand_id` 0 or not an integer; an unknown status and an extra property are accepted).
- List, authorization and models: AC-02 (GET with the API access token), AC-24 (malformed `200` body keeps the previous list), AC-19 and AC-25 (`403` access denied; `401` one recovery, no loop), AC-03 and AC-04 (known and unknown status), AC-31 and AC-47 (`bae_link` other than the exact API path of its own demand), AC-49 (token sent only to the API origin, PDF route included, and to negotiate). Verify that no test uses a field name absent from the contract.
- Opening a document: AC-05 (blank tab, one authenticated `GET {VITE_API_BASE_URL}{bae_link}`, object URL revoked), AC-55 (401, 403, 404, 500, 503, network error or a content type other than `application/pdf`: tab closed, French message, no retry beyond one 401 recovery) and AC-56 (blocked tab). AC-57 (the PDF viewer under the production CSP) is manual.
- Real time and creation: AC-36 to AC-40 (negotiate with `withCredentials: false`, reconnection, ignored payload), AC-52 to AC-54 (creation scenario: no list GET after a POST `2xx`; N-1 and N-2 each trigger exactly one reload), AC-14 (multipart has only `gk_number` and `bae_file`, no `user_id`, no manual `Content-Type`), AC-15, AC-16 and AC-26 to AC-28 (`201`, `400/422` keyed by `errors`, `413`, `5xx`, no automatic retry).

Run, from `klesia_webapp/`: `npm run type-check`, `npm run test:run` (and `npm run lint` when it exists).

## Level 3 - Live wire test (environment-dependent, optional)

Needs a dev API connected to a dev Azure SQL database (the developer's Entra account must be a database user and the client IP allowed by the server firewall), a real Entra test registration and a test user. Never run it against production, and never commit or log a token. The in-memory tests above do not need it; report Level 3 as "not run" when the prerequisites are missing.

1. Start the API in Development (`klesia_api/docs/local-development.md`; `http://localhost:5180`), or target the deployed dev App Service (HTTPS; its `Cors:AllowedOrigins` must include the webapp's dev origin).
2. Put a contract proxy between the webapp and the API so every request and response is validated against the contract. With Stoplight Prism available: `npx @stoplight/prism-cli proxy contracts/openapi.yaml http://localhost:5180` (check `--help` for the exact flags first, in particular the one that turns violations into errors).
3. Start the webapp (`npm run dev`) with `VITE_API_BASE_URL` pointing to the proxy, sign in with the test user, and exercise the flow (list loads, a PDF opens through the API, a demand is created and the two notifications refresh the list, `403` for an unknown user, CORS from `http://localhost:5173`). The dev simulator is not a substitute: it does not validate tokens.
4. Optionally run `schemathesis run contracts/openapi.yaml --url http://localhost:5180` with a test token to fuzz the API against the contract, if the tool is installed.
5. Collect the proxy's violation report; every violation is a finding.

## Rules for test data

Use invented values only: GK numbers like `GK260902-144743112`, example users, `/api/v1/demands/1/bae` paths and `https://contoso.blob.core.windows.net/...` blob references (internal, never returned by the API). No real GK number, document, tenant id, client id or token in a test, fixture, log or commit.

## Report

List per level: what was written, commands run with their outcome, the AT and AC identifiers covered and not covered, and anything that could not run (missing database, tenant, tool) with the reason. A contract change needed to make a test pass is a proposal for the user's approval, never a silent edit.
