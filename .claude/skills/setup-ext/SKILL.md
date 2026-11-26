---
name: setup-ext
description: "Set up one roadmap external dependency EXT-XXX step by step. Use for EXT-001 through EXT-008, Azure resources, Entra configuration, SQL, Storage, SignalR, Document Intelligence, test accounts, or GitHub OIDC."
argument-hint: "EXT-XXX (for example EXT-001)"
user-invocable: true
disable-model-invocation: true
---

# Set up external dependency

Guide the setup of exactly one external dependency: `$ARGUMENTS`.

Files: `roadmap/roadmap.md` section 6 is the status source, `roadmap/T-*.md` records the dependent tasks, and any `ARB-*.md` referenced by the dependency or those tasks provides the settled decision or open constraint.

## Step 0: Parse and locate

- Accept `EXT-1`, `ext-001`, `1`, or `EXT-001`; normalize to `EXT-` plus three digits.
- Read `roadmap/roadmap.md` section 6 and find the normalized code.
- If the code has no row: answer that it does not exist and stop.
- Read the complete row: subject, provided by, needed by, and current status.
- If status is `PROVIDED`, report that it is already satisfied and stop.
- Read the referenced task files for the required operational context and the relevant specifications or local-development documents.

## Step 1: Identify the satisfaction checklist

Prepare a numbered checklist from the roadmap subject and project sources. Do not invent providers, values, role names, regions, or resource names.

Never ask for or store passwords, client secrets, access keys, connection strings with credentials, tokens, or private keys. Ask only for the location or owner of sensitive values and record the location, not the value.

Use these project-specific paths:

- `EXT-001`: Entra app registrations, tenant, API scope, SPA redirect, consent, and optional claims.
- `EXT-002`: Entra test accounts and allowlist identities.
- `EXT-003`: dev Azure SQL database, Entra database access, network/firewall access, and developer IP.
- `EXT-004`: disposable Azure SQL database for constraint tests.
- `EXT-005`: Azure Storage account, private container, and Storage Blob Data Contributor role.
- `EXT-006`: Azure SignalR Serverless resource, CORS, and the role selected by `ARB-004`.
- `EXT-007`: Document Intelligence endpoint/model, role, and anonymized test samples.
- `EXT-008`: GitHub-to-Azure OIDC federation and deployment/SQL principals.

## Step 2: Give the setup procedure

Read only the sources needed for the selected dependency. Use short phases and a verification command or portal check after each phase.

### EXT-001 — Entra app registrations

1. Confirm the tenant and tenant administrator.
2. Create or identify the API app registration for delegated access.
3. Define a scope named `access_as_user` under the API application ID URI.
4. Create or identify the SPA app registration.
5. Add the exact SPA redirect URI `http://localhost:5173/redirect.html` for development and the deployed `/redirect.html` URI for the target environment.
6. Add the API delegated scope as an authorized API permission and grant consent as required.
7. Configure optional claims `given_name` and `family_name` only if approved for the client flow.
8. Record these locations, not values: tenant ID, API client ID, SPA client ID, scope, redirect URIs.

Verify:
- `az ad app show --id <api-client-id>` returns the API registration and exposed scope.
- `az ad app show --id <spa-client-id>` returns the SPA redirect configuration.
- The webapp can use `/.env.local` values matching the registrations without embedding a secret.

### EXT-002 — Entra test accounts

1. Obtain one active allowlisted account, one inactive account, and one non-allowlisted account.
2. Capture only the stable Entra object IDs and account purpose, never passwords or tokens.
3. Add the allowlisted active user's `oid` to the database allowlist through the project-approved process.
4. Verify that each account can authenticate to the tenant and that application consent is available.

Verify:
- Active allowlisted account: API returns owned data.
- Inactive account: API returns the specified forbidden behavior.
- Non-allowlisted account: API returns the specified forbidden behavior.

### EXT-003 — dev Azure SQL

Read `klesia_api/docs/local-development.md` and follow its database-user, firewall, and publishing procedure.

1. Confirm the subscription, resource group, SQL server, and database.
2. Ensure Entra authentication and passwordless developer access are configured.
3. Add the developer IP rule or private connectivity required by the environment.
4. Map the developer Entra identity to the database principal with the grants in the local-development guide.
5. Test connectivity from the workstation.

Verify:
- `/health` can reach SQL and return `200` once the API is configured.
- The developer can publish the DACPAC and run local SQL tests without credentials in source files.

### EXT-004 — disposable Azure SQL

1. Identify or create a disposable database separate from demo data.
2. Configure the test-only `KLESIA_TEST_SQL_CONNECTION` value in the developer environment, never in tracked files.
3. Confirm cleanup/constraint-test isolation and network access.

Verify:
- Constraint tests run only when the environment variable is explicitly set.
- The disposable database can be rebuilt or discarded without affecting production/demo data.

### EXT-005 — Azure Storage

1. Confirm the subscription, storage account, and private Blob container.
2. Grant the developer or workload identity `Storage Blob Data Contributor` as required.
3. Ensure public blob access remains disabled.
4. Configure `Storage:BlobServiceUri` and `Storage:ContainerName` outside source control.

Verify:
- `az storage container show --name <container> --account-name <account> --auth-mode login` succeeds.
- The account can list/upload only through the intended private container and identity.

### EXT-006 — Azure SignalR

Read `roadmap/ARB-004.md` and follow the selected Serverless role/model.

1. Confirm or create the Azure SignalR resource in Serverless mode.
2. Configure CORS for `http://localhost:5173` for local development and the deployed SPA origin for deployment.
3. Assign the role required by `ARB-004` to the API's managed identity or developer identity.
4. Configure `AzureSignalR:Endpoint` outside source control.
5. Confirm local or managed identity authentication, with no connection string or access key in source.

Verify:
- Negotiate returns a URL and short-lived connection token.
- The hub `demands` accepts a user-targeted connection.
- `demandUpdated` reaches only the owning user.

### EXT-007 — Document Intelligence

Read the relevant OPEN-09 and T-059/T-065 materials.

1. Confirm the custom BAE model, endpoint, model ID, and required role.
2. Obtain a set of anonymized sample PDFs and expected answers.
3. Store endpoint/model identifiers in approved configuration locations.
4. Assign `Cognitive Services User` or the approved role to the required workload identity.

Verify:
- The adapter resolves the model and endpoint.
- An anonymized sample produces the expected answer without logging document contents.

### EXT-008 — GitHub OIDC

1. Confirm the Azure deployment principal and SQL deployment principal.
2. Create or update the GitHub-to-Azure federated credential for the approved repository/environment.
3. Assign only the deployment and SQL roles required by the deployment task.
4. Confirm secret-free GitHub Actions configuration and required environment names.

Verify:
- A credential-free workflow can authenticate to Azure OIDC.
- Deployment and SQL roles are sufficient and no secret is committed.

## Step 3: Stop and verify

- Ask the user or owner to perform portal/Azure CLI steps that require elevated or external access.
- Provide the next command/check after each successful phase.
- Do not guess resource names, secrets, IDs, or access results.
- At the end, summarize:
  - completed phases;
  - verification results;
  - unresolved blockers and who owns them;
  - which values were confirmed but are stored outside the repository;
  - which dependent tasks may become ready.

## Step 4: Mark provided

Do not change roadmap status automatically.

Only after the user explicitly confirms that every item for the selected EXT is satisfied:
1. Change the status in `roadmap/roadmap.md` section 6 from `PENDING` to `PROVIDED`.
2. Never put secret values or identifiers into the roadmap.
3. Add a short dated line to the roadmap change log §9 stating the dependency is now provided, at which location the team stores the values, and the tasks it unblocks.
4. Re-evaluate dependent tasks. A task with only satisfied dependencies becomes ready; a task that also depends on another PENDING EXT or OPEN ARB remains BLOCKED.
5. Report the tasks now ready and do not change their status.
