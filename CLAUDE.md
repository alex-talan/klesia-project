# Klesia System

This workspace contains two applications:
- `klesia-api`: backend REST API (.NET)
- `klesia-webapp`: Vue frontend

## System boundary

The canonical HTTP contract is:

`contracts/openapi.yaml`

The normative architecture reference is:

`architecture-plan.md`

Agents MUST read `architecture-plan.md` before any cross-application or structural change (HTTP boundary, authentication, data model, hosting, real-time, new external dependency). It sets the architecture principles (AP-01 to AP-14) and the change sequences (section 15); it references the specifications and does not replace the contract.

Never introduce or modify an API request/response contract without updating the canonical contract.

## Cross-project changes

For features affecting the HTTP boundary:

1. inspect both applications;
2. update the contract first;
3. modify the backend implementation;
4. update the frontend API layer (hand-written types in `klesia_webapp/src/models/demand.ts`);
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
   - Update the hand-written frontend types (`klesia_webapp/src/models/demand.ts`) so that they mirror the contract exactly.
   - Update frontend API consumption.
   - Run backend, frontend, and integration tests.
5. Changes to authentication, pagination, validation, error handling, and response formats must be reflected in the contract.
6. Avoid breaking API changes unless explicitly approved.
7. The frontend types are **hand-written** (webapp specification §6.1, MODEL-01 to MODEL-03) and must stay identical to the contract schemas; `check-contract` verifies it. If a generated client is ever introduced, this rule changes first and generated files are never edited by hand.

### Responsibilities

**klesia-api**:
- Implements the OpenAPI contract.
- Validates requests and responses.
- Maintains contract tests.

**klesia-webapp**:
- Consumes the OpenAPI contract.
- Mirrors the contract schemas in hand-written TypeScript types (`src/models/demand.ts`), the only place where backend DTOs are represented.
- Does not redefine backend DTOs anywhere else or independently of the contract.