# T-058 — M1 Local Demo Report

**Date:** 2026-11-26  
**Result:** PASS — local M1 demo completed without extraction.

## Environment

- The API and Vue SPA ran locally on ports 5180 and 5173 against the dev Azure SQL, Blob Storage, and SignalR resources.
- EXT-001, EXT-003, EXT-005, and EXT-006 were PROVIDED; ARB-004 is SETTLED.
- API settings are in ignored `klesia_api/src/WebApi/appsettings.Local.json`; SPA settings are in ignored `klesia_webapp/.env.local`. The API and SPA PDF-size limits match.
- The dev database schema was built, reviewed, and published. The generated script had no table drops or row deletes; the five replaced check constraints were revalidated; the user seed only inserts missing users.

## Checklist

- **PASS — API health:** `GET /health` returned `200 {"status":"ok"}` with the dev database.
- **PASS — Vite startup:** the normal `npm run dev` script explicitly loads `vite.config.ts`; the Vue syntax overlay is gone.
- **PASS — Entra and initial list:** the user signed in through the SPA; the authenticated app loaded the empty-list state before creation.
- **PASS — Create and N-1:** three synthetic duplicate submissions used a generated test PDF held in the OS temp directory. The modal closed; observed POST responses were `201`; the authenticated list updated to three `pending` rows in all three open SPA tabs. On the instrumented submission, each tab made exactly one list GET returning `200`; the initiating GET began before the POST response, consistent with the N-1 notification rather than a POST-success reload.
- **PASS — Persistence:** the pending rows appeared through the API, and the authenticated document route returned `200 application/pdf`, confirming the stored document was readable through the API.
- **PASS — PDF viewer tab:** the authenticated route returned `200 application/pdf`; the user-provided screenshot confirms the synthetic PDF rendered in the browser viewer in a new tab.
- **PASS — No extraction:** the three demands remained `pending`; no extraction activity appeared in captured API console output. A source scan found no extraction worker, `IDocumentExtractor`, or Document Intelligence adapter wired in `klesia_api/src` at this milestone.
- **PASS — Secret/config hygiene:** local config files are Git-ignored; no identifiers, credentials, document content, GK values, or blob URLs are recorded in this report.

## Known limits

The three synthetic dev demands and their private blobs remain in `pending` state. When the extraction worker and recovery sweep are enabled by T-063 to T-065, they may be processed; coordinate authorized dev-data cleanup before enabling extraction.

## Verification

- DACPAC build and SqlPackage publish: passed.
- `dotnet build Klesia.sln`: passed.
- `dotnet test Klesia.sln`: 211 passed, 9 skipped because `KLESIA_TEST_SQL_CONNECTION` was unset.
- Webapp `type-check`, `lint`, `format:check`, `test:run`, `audit`, and `build`: passed; 209 frontend tests passed and the production dependency audit found 0 vulnerabilities.
- Contract files were unchanged.

## Remaining

The M1 demo is complete. The temporary synthetic PDF fixture was removed after upload. The three synthetic pending demands and their private blobs remain in the dev environment; coordinate authorized cleanup before enabling the future extraction worker and recovery sweep.
