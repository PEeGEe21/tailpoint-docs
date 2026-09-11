# Mobile API Contract Audit

**Status:** Complete — initial mobile client generated  
**Last updated:** 2026-09-04  
**Repositories reviewed:** `track-a-project-backend`, `tailpoint-mobile`

## Purpose

This audit establishes whether the backend's OpenAPI document is reliable enough to generate and maintain the mobile TypeScript client.

## Executive finding

The initial mobile surface now has a reproducible backend OpenAPI artifact covering authentication, organizations, projects, tasks, dependencies, approvals, notifications, search, and attachments. The mobile repository pins that artifact, generates typed paths with `openapi-typescript`, calls them through `openapi-fetch`, and rejects generated-output drift in CI. Breaking-change detection and the remaining compatibility review are still required before treating every backend route as mobile-stable.

## Evidence snapshot

| Measure | Finding |
| --- | ---: |
| Controllers | 54 |
| Controllers with `@ApiTags` | 2 |
| Controllers with `@ApiOperation` | 2 |
| Controllers with explicit success-response decorators | 0 |
| DTO files under `src/**/dto` | 72 |
| DTO files with `@ApiProperty` metadata | 2 |
| DTO files with class-validator metadata | 50 |

Swagger is configured in `src/main.ts` at `/api/docs`, with JSON available through Nest's generated `/api/docs-json` route. The API uses the `/api/` global prefix and bearer authentication. The title and description currently contain legacy spelling and naming.

## Mobile-critical coverage matrix

| Domain | Routes reviewed | Current contract quality | Main gaps | Priority |
| --- | ---: | --- | --- | --- |
| Authentication | 17 | Strong baseline | Mobile routes now have request/response/error schemas and body-based refresh; token rotation/revocation semantics remain | P0 |
| Organizations and invitations | 15 | Strong baseline | Requests, responses, pagination, authentication, and organization-header requirements are explicit; authorization consistency still requires runtime review | P0 |
| Projects | 32 | Strong baseline | Every core project operation now has tenant/auth requirements, a summary, and a success schema; specialized nested project modules remain separate P1 contracts | P0 |
| Tasks | 14 | Strong baseline | Core create/update/status/priority bodies and list/detail responses are typed; custom fields, discussions, and dependencies remain separate P1 contracts | P0 |
| Task dependencies | 6 | Strong baseline | Requests, dependency results, warnings, scheduling previews, authentication, and errors are explicit | P1 |
| Approvals | 7 | Strong baseline | Staged requests, decisions, delegation, inbox/detail/options responses, authentication, and errors are explicit | P1 |
| Notifications | 10 | Strong baseline | Notification lists, push subscription requests, responses, authentication, and optional organization context are explicit | P1 |
| Global search | 1 | Strong baseline | Query constraints and typed cross-feature result metadata are explicit | P1 |
| Resources/attachments | 11 | Strong baseline | Resource CRUD, multipart upload, binary download, preview requests, authentication, and errors are explicit | P1 |

## Cross-cutting contract gaps

1. Initial mobile operations declare reusable standard error schemas; legacy/non-mobile operations still need alignment.
2. Initial tenant-scoped mobile operations declare the organization header; remaining operations need alignment.
3. Initial protected mobile operations declare bearer authentication; remaining operations need alignment.
4. Pagination names and response metadata are inconsistent or implicit.
5. IDs mix integers, UUIDs, and untyped strings without reusable schemas.
6. Dates, enum values, nullable fields, and compatibility expectations are not consistently explicit.
7. Multipart uploads, binary downloads, and preview redirects/URLs lack media-type schemas.
8. A deterministic, infrastructure-free export and committed artifact now exist; CI verifies that the artifact remains current.
9. Generated-client drift checking is active; semantic lint and breaking-change detection remain.
10. Route naming includes legacy and ambiguous paths that should be preserved or versioned deliberately rather than normalized accidentally.

## Remediation sequence

### P0 — Safe client-generation baseline

- [x] Extract Swagger configuration into a reusable module and correct its metadata.
- [x] Add deterministic `openapi:export` and `openapi:check` commands that work without live infrastructure or an HTTP listener.
- [x] Define reusable schemas for API errors, pagination metadata, bearer auth, and `x-organization-id`.
- [x] Fully document core project and task operations and DTOs. Mobile authentication and organization operations are also documented.
- [x] Add mobile-safe `POST /api/auth/refresh` while retaining and deprecating the legacy query route during migration.
- [x] Validate the exported document in CI and fail on undocumented path parameters or invalid schemas.

### P1 — Initial mobile feature coverage

- [x] Document dependencies, approvals, notifications, search, and attachment operations.
- [x] Specify success and standard expected error outcomes for the initial mobile surface.
- Add examples only where they clarify enums, polymorphic results, or state transitions.

### P2 — Contract governance

- Compare the new artifact against the last mobile-compatible contract in CI.
- [x] Generate the mobile client/types with pinned versions of `openapi-typescript` and `openapi-fetch`.
- [x] Fail mobile CI when generated output differs from the committed contract.
- Record intentional breaking changes and a mobile compatibility window.

## Definition of ready for mobile generation

The backend contract is ready when:

- the export succeeds from a clean checkout;
- OpenAPI validation passes;
- all P0/P1 mobile routes have request, success, and standard error schemas;
- tenant requirements are explicit per operation;
- pagination, dates, IDs, enums, and nullable fields are consistent;
- the mobile client generates with no unresolved `any` response bodies;
- backend and mobile CI detect contract drift.

## Next implementation task

Add breaking-contract detection against the last mobile-compatible artifact, then finish refresh-token rotation and session semantics before building authentication screens.
