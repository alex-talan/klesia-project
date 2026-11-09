# Klesia System

This workspace contains two applications:
- `klesia-api`: backend REST API (.NET)
- `klesia-webapp`: Vue frontend

## System boundary

The canonical HTTP contract is:

`contracts/openapi.yaml`

Never introduce or modify an API request/response contract without updating the canonical contract.

## Cross-project changes

For features affecting the HTTP boundary:

1. inspect both applications;
2. update the contract first;
3. modify the backend implementation;
4. update/regenerate the frontend API layer;
5. run backend tests;
6. run frontend tests;
7. run integration/contract tests.

## Project instructions

Follow the more specific `CLAUDE.md` found inside each application when modifying that application.

## Scope

Do not duplicate frontend or backend implementation rules here. Those belong to the child projects.

## API Contract Governance

The canonical API specification is: `contracts/openapi.yaml`

### Mandatory rules

1. The OpenAPI specification is the single source of truth for HTTP API contracts.
2. Never introduce, remove, or modify an API endpoint without updating the OpenAPI specification.
3. Never independently invent API request or response structures in either application.
4. When implementing a cross-application feature:
   - Read the existing OpenAPI contract.
   - Update the contract first.
   - Implement the backend changes.
   - Regenerate frontend TypeScript types.
   - Update frontend API consumption.
   - Run backend, frontend, and integration tests.
5. Changes to authentication, pagination, validation, error handling, and response formats must be reflected in the contract.
6. Avoid breaking API changes unless explicitly approved.
7. Do not manually modify generated API client files.

### Responsibilities

**klesia-api**:
- Implements the OpenAPI contract.
- Validates requests and responses.
- Maintains contract tests.

**klesia-webapp**:
- Consumes the OpenAPI contract.
- Uses generated TypeScript types.
- Does not redefine backend DTOs independently.