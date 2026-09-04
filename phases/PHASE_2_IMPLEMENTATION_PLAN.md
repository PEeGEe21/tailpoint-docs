# Phase 2 Implementation Plan

## Delivery decision

Phase 2 starts with Custom Fields. Forms, reusable templates, and every intake channel need a stable typed-field contract, while Custom Workflows needs a centralized transition service before Forms can safely select a destination status. Features remain default-off and are enabled only through the existing entitlement service.

## Delivery order

1. **2.1 Custom Fields** — typed field definitions and task values.
2. **2.2 Custom Workflows** — versioned workflow definitions and centralized transitions.
3. **2.5 Milestones** — dedicated milestone records required by project updates and approvals.
4. **2.3 Forms and Request Intake** — versioned forms mapped through the field and workflow contracts.
5. **2.4 Reusable Templates** — versioned task, checklist, and project snapshots with workflow remapping.
6. **2.6 Basic Approvals** — polymorphic requests for tasks, documents, and milestones.
7. **2.7 Universal Intake Expansion** — normalized intake events shared by CSV, webhook, email, and SDK channels.
8. **2.8 AI Intake Assistance** — optional, review-only suggestions after normalized intake is stable.

Milestones may be developed in parallel with Custom Workflows once the Custom Fields contract is stable. AI Intake Assistance does not block completion of the non-AI Phase 2 goal.

## 2.1 Custom Fields — MVP contract

### Authorization

- Viewer, Contributor: read active definitions and values on tasks they can view.
- Contributor: set values only while creating a task or editing a task they are permitted to change.
- Editor, Owner: create, reorder, update, and archive field definitions; set values on any editable task.
- Archived definitions and their historical values remain readable but cannot receive new values.
- Person values must reference active members of the same organization; every definition, task, option, and value must belong to the same organization and project.

### Schema

- `custom_field_definitions`: organization, project, stable key, name, description, type, required flag, position, default value, archived timestamp, creator, timestamps.
- `custom_field_options`: definition, stable key, label, color, position, archived timestamp, timestamps.
- `task_custom_field_values`: task, definition, one normalized JSON value, timestamps, unique `(task_id, definition_id)`.

The supported types are `text`, `number`, `date`, `single_select`, `multi_select`, `checkbox`, `person`, and `url`. JSON storage is an envelope only; the service validates and normalizes values by definition type before every write. Select values store stable option identifiers, not labels. Defaults use the same validator as task values.

### API tickets

- **CF-01 — Capability and contract**: register `custom_fields` as default-off across backend, workspace, and admin; document authorization and type rules. **Status: complete.**
- **CF-02 — Persistence**: add entities, foreign keys, tenant/project consistency constraints where supported, indexes, and a reversible migration. **Status: complete.**
- **CF-03 — Definition API**: list/create/update/reorder/archive definitions and options; reject destructive type changes and option removal when values exist. **Status: complete.**
- **CF-04 — Value API**: serialize values with tasks, validate required/default values on create and update, and provide an atomic bulk-value endpoint. **Status: complete.**
- **CF-05 — Filters and export**: typed filtering with server pagination and inclusion in task export. **Status: complete.**
- **CF-06 — Workspace UI**: project settings editor plus reusable task form/detail controls with loading, empty, error, and retry states. **Status: complete.**
- **CF-07 — Validation and pilot**: permission, tenant isolation, archive/history, option compatibility, filter, and large-project tests; admin pilot enablement. **Status: complete. Development migration verified and audited admin pilot controls are available; organization activation remains an operator rollout decision.**

### API shape

- `GET /projects/:projectId/custom-fields?includeArchived=false`
- `POST /projects/:projectId/custom-fields`
- `PATCH /projects/:projectId/custom-fields/:fieldId`
- `PUT /projects/:projectId/custom-fields/order`
- `POST /projects/:projectId/custom-fields/:fieldId/archive`
- `PUT /tasks/:taskId/custom-fields` with `{ values: [{ fieldId, value }] }`

Task reads return `customFields` as definition/value pairs so clients do not have to join option labels themselves. Task create and edit accept a `customFields` array and commit task plus field values transactionally.

## 2.2 Custom Workflows — MVP contract

### Boundary and compatibility

Custom Workflows governs the lifecycle of tasks inside a project. It is distinct from the existing reusable workflow-template library, which copies a reusable set of tasks into a project. Existing `status` rows remain the project-visible columns and receive stable workflow identifiers; workflow versions define which status-to-status transitions are allowed.

Projects without the `custom_workflows` entitlement retain today's unrestricted status changes. When the capability is enabled, every status mutation—including task edit, board drag, bulk/API updates, ingestion, forms, and future automation—must call the same transition service. Existing projects receive a compatible published workflow that permits every transition between their active statuses, avoiding behavior changes at enablement.

### Authorization and transition rules

- Viewer: read the published workflow and transition history for tasks they may view.
- Contributor: perform published transitions allowed to the contributor role on tasks they may edit.
- Editor: perform editor/contributor transitions and edit workflow drafts.
- Owner: publish workflow versions, migrate statuses, and perform owner transitions.
- A transition may require standard task fields and active Custom Field definitions. Requirements are validated server-side in the same transaction as the status change.
- Published versions are immutable. Editing creates or updates a draft; publishing validates and snapshots the complete graph.
- In-use statuses cannot be removed from a published workflow without an explicit mapping to another active status.

### Schema

- `project_workflows`: organization, project, current published version, draft version, timestamps, unique project ownership.
- `project_workflow_versions`: workflow, version number, state (`draft`, `published`, `retired`), immutable definition snapshot, creator/publisher, timestamps.
- `workflow_statuses`: version, stable status key, linked project status, position, initial/terminal flags.
- `workflow_transitions`: version, stable key, source and destination workflow statuses, permitted project roles, required standard/custom fields.
- `task_transition_history`: organization, project, task, workflow version, transition key, source/destination status snapshots, actor, validated-field snapshot, timestamp.

### API tickets

- **WF-01 — Capability and contract**: register `custom_workflows` as default-off across backend, workspace, and admin; document compatibility, authorization, and transition invariants. **Status: complete.**
- **WF-02 — Versioned persistence and backfill**: add workflow/version/status/transition/history entities and a reversible migration; generate compatible default workflows for existing projects. **Status: complete. Development migration and compatibility backfill verified.**
- **WF-03 — Draft and publish API**: read the published workflow, create/update a draft, validate graphs, publish immutable versions, and require status migration mappings. **Status: complete.**
- **WF-04 — Central transition service**: validate role, edge, entitlement, tenant/project ownership, required fields, and destination status in one transaction; record history. **Status: complete.**
- **WF-05 — Route integration**: route task edit, board/status updates, bulk changes, ingestion, and internal task creators through the transition service; add bypass-regression tests. **Status: complete. Task edits, attachment edits, board moves, ingestion reopen, and legacy status deletion are covered; task creation validates its project-scoped initial status through existing creation contracts.**
- **WF-06 — Workspace editor and transition UX**: add a safe project workflow editor, publish validation, migration mapping, permitted-action controls, and transition-history UI with recovery states. **Status: complete. Project settings expose draft graph, role, standard/custom-field requirement, validation and publish controls; task editing consumes authoritative allowed transitions and task details expose immutable history.**
- **WF-07 — Validation and pilot**: permission, tenant isolation, graph validity, required-field, concurrent publish, compatibility, history, and large-workflow tests; admin pilot enablement. **Status: complete. Focused permission, cross-project destination, graph, role, required-field, compatibility, history, owner-publish, ingestion, bypass, and 50-status workflow tests pass; development migration/backfill and audited admin pilot controls are verified. Organization activation remains an operator rollout decision.**

### Acceptance test matrix

- Direct task update endpoints cannot bypass the published workflow.
- Cross-project status IDs, transitions, custom fields, and workflow versions are rejected without leaking record existence.
- Draft changes do not affect task transitions until successfully published.
- Invalid graphs—including duplicate edges, missing status links, invalid roles, and unreachable required states—cannot be published.
- Removing an in-use status requires an explicit valid destination mapping and migrates tasks transactionally.
- Required standard and custom fields are validated against the task's resulting state before the status changes.
- Transition history retains immutable status labels, workflow version, actor, and rule key after later workflow edits.
- Enabling the capability on an existing project initially preserves every status change that was previously possible.

## 2.5 Milestones — MVP contract

### Boundary and progress rules

- A milestone is a dedicated project record, never a special task or task flag.
- Viewer and Contributor may read active milestones in projects they can view. Editor and Owner may create and edit milestones and task links. Owner may archive milestones; completion follows normal edit permission but is audited.
- Owners must be active members of the same organization and project. Linked tasks must belong to the same organization and project as the milestone.
- Progress is deterministic: only linked tasks with `counts_toward_progress=true` are eligible; completion is determined by the linked task status `isTerminal` flag. No eligible tasks yields 0%, otherwise progress is `terminal eligible tasks / eligible tasks`, rounded to the nearest whole percent.
- Setting a milestone to completed records `achieved_at`. If eligible linked tasks remain open, a non-empty manual completion reason is required. Reopening clears `achieved_at` but retains activity history.
- Archiving hides a milestone from normal lists while retaining its task links, update references, and activity history.

### Schema

- `milestones`: organization, project, title, description, completion criteria, target date, status, health, owner, creator, achieved timestamp, completion reason, archive timestamp, and timestamps.
- `milestone_tasks`: milestone, task, progress-eligibility flag, timestamp, unique `(milestone_id, task_id)`.
- Milestone activity uses the existing project activity stream with immutable action metadata snapshots.

### API tickets

- **MS-01 — Capability and contract**: register `milestones` as default-off across backend, workspace, and admin; document authorization, progress, completion, and archive invariants. **Status: complete.**
- **MS-02 — Persistence**: add milestone and milestone-task entities, project/organization ownership fields, indexes, foreign keys, and a reversible migration. **Status: complete; development migration applied and verified.**
- **MS-03 — CRUD and task links API**: list/read/create/update/archive milestones; atomically replace project-scoped task links and validate owners. **Status: complete. Tenant/project-scoped endpoints, active project-member owner checks, legacy-compatible project ownership checks, atomic task-link replacement, archive safeguards, filtering, pagination, and focused boundary tests are implemented.**
- **MS-04 — Progress and completion service**: calculate progress from eligible linked task terminal states, enforce manual completion reasons, and expose stable calculation details. **Status: complete. Responses expose percent/eligible/completed/open counts, status changes use a pessimistic transaction lock, completion timestamps are server-owned, open eligible tasks require a reason, and reopening clears completion state.**
- **MS-05 — Update references and activity**: authorize milestone references from Structured Project Updates and record milestone lifecycle activity snapshots. **Status: complete. Active project-scoped milestones are searchable update references with immutable title snapshots; milestone create, update, link, status, and archive mutations emit project activity metadata.**
- **MS-06 — Workspace timeline**: add project milestone list/timeline, create/edit/detail flows, task-link controls, progress and health display, plus loading/empty/error/retry states. **Status: complete. The capability-gated Milestones tab provides ordered timeline cards, progress/health/status display, create/edit owner and task-link controls, guarded completion/reopen, owner archive confirmation, and recovery states; project updates can attach milestones.**
- **MS-07 — Validation and pilot**: permission, tenant isolation, cross-project link, progress, manual completion, archive/history, update-reference, and large-project tests; admin pilot enablement. **Status: complete. Focused milestone/update suites cover project and organization scoping, owner membership, duplicate/cross-project links, progress eligibility, guarded completion, activity snapshots, authorized update references, and a 500-task milestone. Backend/workspace/admin builds pass, runtime module/route initialization is verified, and audited admin pilot controls are available; organization activation remains an operator decision.**

## 2.3 Forms and Request Intake — MVP contract

### Versioning and submission boundary

- A request form belongs to exactly one organization and project and has a non-guessable public key. Public keys identify forms but do not grant workspace access.
- Form configuration lives in immutable versions. Editors work on one draft; owner publication retires the previous published version atomically. Existing submissions always retain the exact published version and original answers they used.
- Visibility is either `public` or `organization`. Organization forms require an active organization member who can view the destination project. Public submitters never receive project, member, task, custom-field, or workflow data beyond the published form snapshot and confirmation text.
- Each field maps to a supported standard task field, an active project Custom Field, or submission-only data. Conditional rules may only reference earlier stable field keys and are evaluated identically on client and server; server evaluation is authoritative.
- Submission validation and task creation are one normalized transaction. Destination status must belong to the form project and be a valid initial creation status under the published workflow contract. Custom Field values use the existing typed validator.
- A submission stores original answers and validation outcomes even if task creation fails. Accepted submissions link to the created task; retries use the submission identifier as an idempotency key and cannot create duplicate tasks.
- Public endpoints use dedicated rate limits, a honeypot/timing signal, bounded payloads, neutral error responses, hashed source IPs, and observable rejection/quarantine states. Attachments are size/type/count limited and remain quarantined until validation/scanning succeeds.

### Schema

- `request_forms`: organization, project, opaque public key, display name, creator, archive timestamp, and timestamps.
- `request_form_versions`: form, version number, state, immutable title/description/visibility, destination status, confirmation text, creator/publisher, and publish timestamp.
- `request_form_fields`: version, stable key, label/description, input type, mapping target, standard/custom target, required/order flags, option snapshot, conditional rules, and input configuration.
- `request_form_submissions`: organization, project, form/version, created task, optional authenticated submitter/email, state, original answer and validation snapshots, privacy-safe request metadata, failure reason, and timestamp.
- `request_form_submission_attachments`: submission, original name, opaque storage key, MIME type, size, digest, quarantine state, and timestamp.

### API tickets

- **RF-01 — Capability and contract**: register `request_forms` as default-off across backend, workspace, and admin; document versioning, visibility, mapping, security, and idempotency invariants. **Status: complete.**
- **RF-02 — Versioned persistence**: add form/version/field/submission/attachment entities, ownership fields, indexes, foreign keys, immutable submission links, and a reversible migration. **Status: complete; development migration applied and all five tables verified.**
- **RF-03 — Builder and publish API**: list/read/create/archive forms, create/update drafts, validate conditional graphs and mappings, preview, and owner-publish immutable versions. **Status: complete.** The project-scoped API now enforces editor/owner permissions, immutable published versions, transactional replacement of published definitions, mapping/type/condition validation, custom-option snapshots, activity records, and large-form validation coverage.
- **RF-04 — Normalized submission pipeline**: render published snapshots, validate conditional/required/typed answers, create tasks and Custom Field values transactionally, enforce workflow destination rules, retain failures, and provide idempotent submission history. **Status: complete.** Published snapshots and project-scoped submission history are exposed through capability-gated endpoints; server-authoritative condition/type validation retains rejected attempts, task and typed Custom Field writes share one transaction, published workflow initial-state rules are enforced, failed attempts retain diagnostics and can be retried against their immutable version, and pessimistically locked client submission IDs prevent duplicate tasks.
- **RF-05 — Public security and attachments**: add public/org access boundaries, dedicated throttling, spam/timing controls, neutral errors, IP hashing, upload restrictions, quarantine/scanning hooks, and observable rejection states. **Status: complete.** Public-key routes disclose respondent-safe snapshots only, verify the organization entitlement, use dedicated read/submit/upload limits, retain honeypot and timing rejections, HMAC source addresses, return neutral failures, and restrict uploads by count, size, and MIME type before storing them under opaque keys in quarantine. Workspace history exposes submission and attachment quarantine/rejection states without storage keys.
- **RF-06 — Workspace and respondent UI**: add project form management, safe builder/preview/publish flows, submission/task traceability, and responsive public/private form experiences with confirmation and recovery states. **Status: complete.** The capability-gated Request Forms project tab provides loading/empty/error/retry states, draft creation/editing, server preview validation, owner publication/archive, public-link launch, and submission/task/attachment traceability. The responsive respondent route evaluates conditions, renders typed controls and constrained uploads, prevents duplicate resubmission with client UUIDs, and provides sending, confirmation, unavailable, validation, and attachment-recovery states.
- **RF-07 — Validation and pilot**: permission, tenant isolation, immutable version, condition, mapping, workflow, custom-field, idempotency, abuse, attachment, failure-retention, and large-form tests; admin pilot enablement. **Status: complete.** Focused request-form coverage exercises mapping and tenant-scoped definition validation, owner-only publication and immutable versions, conditional/typed/file validation, respondent snapshot redaction, privacy-safe IP hashing, option snapshots, and 100-field definitions. Backend and both frontend type checks pass, the backend production build passes, and the audited default-off admin override is available for pilot activation; enabling an organization remains an operator decision.

### Acceptance test matrix

- Direct API calls cannot bypass entitlement or project-role checks.
- Required fields reject missing and type-invalid values server-side.
- Archived fields preserve values and appear only when historical fields are requested.
- Archiving or relabeling an option preserves old values; deleting an option referenced by a task is rejected.
- Person values cannot reference outsiders or inactive organization members.
- Project A definitions cannot be assigned to Project B tasks, including within the same organization.
- Filters use typed comparison semantics and return the same result/count population.
- Exports include stable field keys, display names, types, and normalized values.

## 2.4 Reusable Templates — implementation result

- **RT-01 Capability and immutable contract: complete.** `reusable_templates` is default-off across backend, workspace, and admin. Templates are organization-scoped and every edit creates an immutable numbered snapshot.
- **RT-02 Task/checklist/project snapshots: complete.** Task templates capture task defaults, checklist templates capture ordered task items, and project templates capture project/status/task definitions without retaining live-object references.
- **RT-03 Compatibility and instantiation: complete.** Preview reports missing workflow status mappings; mappings are verified against the target project, instantiation resolves mapped statuses and relative due dates, and all project/task objects are created in one transaction.
- **RT-04 Workspace and pilot: complete.** The capability-gated Templates project tab supports creating, listing, visible snapshot previews, explicit status remapping, immutable new-version editing, relative-date configuration, and instantiation with recovery states; audited admin pilot controls are available.

## 2.6 Basic Approvals — implementation result

- **AP-01 Capability and polymorphic contract: complete.** `basic_approvals` is default-off and approval requests target tenant/project-scoped tasks, documents, or milestones with immutable subject snapshots.
- **AP-02 Requests and responses: complete.** One or more active project reviewers can be assigned; requesters cannot review their own requests, responses are unique and immutable, any rejection rejects the request, and unanimous approval resolves it.
- **AP-03 Invalidation and audit: complete.** Subject revision changes invalidate pending approval before a response is recorded. Creation and responses write audit records, optional/required rejection comments are supported, and resolved snapshots remain durable.
- **AP-04 Due dates and workspace: complete.** Due-soon pending requests enqueue idempotent reviewer reminders. The capability-gated Approvals project tab provides named peer and subject selectors, due-date and required-rejection-comment settings, subject snapshots, named reviewer history, and approve/reject actions.
- **AP-05 Validation and pilot: complete.** Focused tests cover reviewer eligibility controls, immutable response affordances, the transactional response insert/status-update regression, template snapshot validation, and stable workflow-key extraction. Backend/workspace/admin type checks and production builds pass; pilot activation remains an operator decision.

## 2.7 Universal Intake Expansion — MVP contract

### Normalized boundary and lifecycle

- Every intake request becomes one durable, organization- and project-scoped normalized event before task creation. Supported channels are `api`, `sdk`, `csv`, `excel`, `webhook`, `email`, and `form`.
- A channel adapter may authenticate, parse, sanitize, and enrich its source payload, but it may not create a task directly. It submits the same normalized task input, source identity, idempotency key, and attachment references to the common pipeline.
- Idempotency is scoped to organization, channel, source identity, and key. Concurrent delivery and later retries return the original event/task outcome and cannot create a second task. Provider identifiers and imported row identifiers become channel idempotency keys rather than task deduplication keys.
- Event states are `received`, `validated`, `accepted`, `rejected`, `quarantined`, and `failed`. Validation failures are durable and immutable; operational failures retain retryable attempts. Rejected or quarantined events never create tasks.
- Validation covers entitlement, tenant/project ownership, destination status and workflow initial-state rules, normalized standard and Custom Field values, bounded content, and attachment policy. Task creation and the accepted event/task link commit in one transaction.
- Task-level occurrence deduplication remains a separate optional rule after event idempotency. Existing ingestion dedupe keys may group repeated alerts into one task without weakening exactly-once event processing.
- Raw secrets, authorization headers, untrusted storage keys, and full source IP addresses are never retained in event snapshots. Sender or webhook identity is attribution only and never grants organization or project access.
- Each processing attempt records its trigger, start/end time, outcome, privacy-safe diagnostics, and retry relationship. Operators can retry failed events; accepted, rejected, and quarantined events require an explicit reprocessing action governed by project permissions.

### Channel rules

- CSV and Excel imports require an editor or owner, use a previewed header-to-field mapping, validate every row independently, and expose row-level outcomes. Retrying an import preserves stable row keys and does not duplicate accepted rows.
- Authenticated webhooks use per-source secrets, signature and timestamp verification, replay protection, bounded bodies, mapping configuration, and secret rotation. Authentication failures receive neutral responses and do not expose project existence.
- Email projects use opaque, revocable recipient tokens. Provider signatures and message identifiers are required; HTML is sanitized, spam signals are evaluated, and supported attachments remain quarantined until validation/scanning succeeds.
- API and SDK intake preserve the current public capture contract during migration. SDK retries reuse a stable idempotency key and surface the durable event identifier and validation outcome.
- Request Forms may migrate to the common pipeline after compatibility coverage proves their respondent-safe response and immutable-version behavior are unchanged.

### Persistence

- `intake_events`: UUID, organization, project, optional task, channel, source identity, idempotency key, state, normalized payload, validation snapshot, optional task dedupe key, failure code/message, retryability, occurrence timestamps, and timestamps; unique `(organization_id, channel, source_key, idempotency_key)`.
- `intake_event_attempts`: event, attempt number, trigger (`initial`, `automatic_retry`, `manual_retry`, `reprocess`), state, diagnostic snapshot, start/completion timestamps, and unique `(event_id, attempt_number)`.
- Later channel tickets add import batches/rows, webhook sources, inbound addresses, and attachment records while retaining `intake_events` as the authoritative processing history.

### API tickets

- **UI-01 — Contract and capability**: register `universal_intake` as default-off across backend, workspace, and admin; document channel authorization, normalized input, lifecycle, idempotency, security, and retry invariants. **Status: complete.**
- **UI-02 — Normalized persistence**: add intake event and attempt entities, tenant/project ownership, scoped idempotency uniqueness, validation/failure snapshots, indexes, foreign keys, and a reversible migration. **Status: complete; the development migration was applied successfully.**
- **UI-03 — Common processing pipeline**: normalize and validate channel input, enforce workflow and typed-field rules, create tasks transactionally, retain failures, and support safe retries/reprocessing. **Status: complete. Durable receive/process, scoped idempotency, attempt history, accepted-event replay, failure retention, permission-gated operator retry/reprocessing APIs, and typed Custom Field mapping all use the shared transactional pipeline. Focused unit and ingestion lifecycle integration tests, backend type checks, and the production build pass.**
- **UI-04 — SDK/API migration**: route the existing capture endpoint through the common pipeline without breaking clients; add stable SDK idempotency and event outcome fields. **Status: complete. Existing capture responses remain compatible with additive event fields, and SDK HTTP retries reuse one generated or caller-supplied idempotency key.**
- **UI-05 — CSV/Excel import**: add upload, sheet/header detection, field mapping, preview, row validation, bounded batch processing, downloadable error reporting, and idempotent retry. **Status: complete. Permission-gated CSV/XLSX preview persists durable batches and stable row identities; quoted/multiline CSV and first-sheet Excel parsing feed validated standard and typed Custom Field mappings through the normalized pipeline. Processing is bounded to 5,000 rows and 10 MB, accepted rows are idempotently skipped on retry, row outcomes are retained, and correction-ready CSV error reports are downloadable. The reversible development migration is applied; focused tests, type checks, and the production build pass.**
- **UI-06 — Authenticated webhooks**: add tenant-safe source configuration, signature/timestamp verification, replay protection, payload mapping, secret rotation, rate limits, and neutral failure responses. **Status: complete. Project editors can manage tenant-scoped webhook sources with one-time secrets, safe dotted-path standard/Custom Field mappings, revocation, and encrypted secret rotation with a bounded overlap window. The public endpoint verifies the exact raw body with HMAC-SHA256, enforces a five-minute timestamp window, scopes delivery identifiers through normalized intake idempotency, applies a dedicated rate limit and bounded JSON body, and returns neutral authentication failures. The reversible development migration is applied; focused security/intake tests, type checks, and the production build pass.**
- **UI-07 — Email-to-task**: add opaque project addresses, provider webhook verification, recipient routing, HTML sanitization, spam controls, attachment quarantine/scanning hooks, message deduplication, and revocation. **Status: complete. Editors can create, revoke, and rotate opaque project addresses. The SendGrid Inbound Parse adapter uses the provider security-policy bearer/OAuth boundary, neutral recipient failures, bounded multipart input, message-ID idempotency, sanitized text/HTML, attribution-only senders, configurable spam quarantine, and MIME/size/digest attachment quarantine ready for scanning. The reversible development migration is applied; 25 focused intake/security tests, type checks, and the production build pass. A live provider/DNS smoke test remains an operator deployment check.**
- **UI-08 — Workspace operations**: add project intake settings, import wizard, channel controls, event history, validation/failure diagnostics, retry actions, and loading/empty/error/recovery states. **Status: complete. The capability-gated Intake Operations project tab preserves API-key/default-status settings and adds durable event history with state filtering, attempt/failure diagnostics, retry/reprocessing controls, CSV/XLSX preview and standard/Custom Field mapping, authenticated error-report downloads, import outcomes, webhook creation/revocation/secret rotation with one-time secret display, and opaque email-address creation/copy/revocation/rotation. Backend event/import history APIs are tenant-scoped and bounded. Loading, empty, error, retry, permission, and capability-disabled states are covered; backend focused tests/type/build and workspace type/production builds pass.**
- **UI-09 — Validation and pilot**: cover permissions, tenant isolation, concurrent idempotency, workflow/custom fields, malicious content, replay, large imports, retry compatibility, builds, migration verification, and audited admin pilot controls. **Status: complete. The release suite covers permission/tenant-scoped operator queries, concurrent scoped-idempotency collision recovery, durable workflow/typed-field validation rejection, exact 5,000-row import bounds, unsafe mapping and HTML handling, stale/replayed provider deliveries, secret-rotation overlap, message-ID deduplication, attachment quarantine, accepted-event replay, and manual retry attempt history. Thirty-five focused backend tests and three ingestion HTTP integration tests pass. Backend, workspace, and admin production builds pass, the compiled Nest application context initializes successfully, and the development database reports no pending migrations. The default-off Universal Intake capability is exposed through the existing audited admin pilot override, and organization activation remains an operator decision.**

### Acceptance test matrix

- No channel adapter or legacy endpoint can bypass the normalized event pipeline once migrated.
- Concurrent requests with the same scoped idempotency identity produce one event and at most one task.
- The same provider identifier or row key may be reused safely by a different organization, channel, or configured source.
- Cross-project statuses, Custom Fields, users, tasks, source configurations, and retry identifiers are rejected without leaking record existence.
- Validation snapshots explain field-level rejection without retaining secrets or unsafe raw content.
- Retrying an operational failure preserves event identity and attempt history; retrying an accepted event returns its existing task.
- Email senders and webhook principals never become workspace actors or receive access through intake.
- Malicious HTML, invalid signatures, replayed timestamps, spam, and unsafe attachments are rejected or quarantined before task creation.
- Large imports are bounded and observable, and partial failures can be corrected and retried without duplicating accepted rows.
- Existing API/SDK clients retain compatible capture behavior throughout migration.

## 2.8 AI Intake Assistance — review-only contract

AI Intake Assistance enriches normalized intake without creating, routing, assigning, or merging work autonomously. Every suggestion is tenant-scoped, generated from authorized context, stores its reason and confidence, and requires an explicit user decision. Applying a suggestion must revalidate its referenced project resources and reject stale input.

- **AI-01 — Suggestion contract and lifecycle**: add durable, versioned intake suggestions with `pending`, `applied`, `dismissed`, and `stale` states; bind them to an intake event and payload fingerprint; store only structured proposed changes, confidence, reasons, governance correlation ID, and reviewer metadata. **Status: complete. The tenant-scoped schema, allowlisted structured contract, stable payload fingerprint, prompt/audit provenance, reviewer metadata, list endpoint, and concurrency-safe dismissal endpoint are implemented. Both AI Assistance and Universal Intake entitlements plus project permissions are enforced in the service. Focused lifecycle and existing AI governance tests pass, the backend typecheck/build succeeds, and the development migration is applied.**
- **AI-02 — Governed suggestion generation**: add a permission- and entitlement-protected endpoint that uses the existing redaction, quota reservation, provider, prompt versioning, and audit pipeline. Parse model output through a strict allowlist and treat all intake content as untrusted data. **Status: complete. The dedicated event-scoped generation endpoint sends only bounded normalized title, description, severity, priority, and channel context through the existing redaction/governance pipeline. The versioned prompt forbids following intake instructions and currently permits cleaned title, category, and priority suggestions only. Output is strictly parsed, type-checked, bounded, explained, confidence-scored, and persisted through the AI-01 contract; malformed or unsupported output is rejected and marks the audit as `invalid_structured_output`. The generic assistance endpoint cannot invoke this prompt. Fifteen focused lifecycle, generation, and governance tests pass, and backend typecheck/build succeeds.**
- **AI-03 — Authorized matching context**: suggest duplicates, assignees, and destination projects only from bounded organization-scoped candidates the actor may view. Never send secrets, raw attachments, unrestricted user directories, or cross-tenant identifiers to the provider. **Status: complete. A server-side context assembler uses the actor's organization and confirmed project access to select at most 12 projects, 20 members per project, and 12 likely duplicate tasks from a 100-task recent scan. It sends IDs and bounded display labels only—never member email addresses, task descriptions, attachments, or unrestricted directories—and excludes the task already created by the intake event. Prompt version 2 permits candidate IDs only, and returned destination, duplicate, and assignee IDs are checked against the exact authorized candidate set, including destination-project membership. Sixteen focused AI tests pass, and backend typecheck/build succeeds.**
- **AI-04 — Review, apply, and dismiss**: expose suggestions beside staged imports and normalized events with before/after values, reasons, and confidence. Apply only selected fields after server-side resource and payload-fingerprint validation; merging and routing always require separate confirmation. **Status: complete. Accepted event rows expose an entitlement-aware AI review panel with current/proposed values, field-level reasons and confidence, selective ordinary-field application, dismissal, regeneration, and dedicated routing/duplicate confirmations. The apply endpoint locks the suggestion transactionally, rechecks the source payload fingerprint, project permissions, tenant ownership, destination default status, category availability, duplicate visibility, and current project membership. Stale and concurrent reviews fail safely. Confirmed duplicate merges rebind the intake event before removing the redundant intake-created task; confirmed routing moves the task to the authorized project's default intake status. Nineteen focused AI tests pass, backend typecheck/build succeeds, and the workspace production build passes.**
- **AI-05 — Validation and pilot**: cover tenant isolation, permissions, prompt injection, malformed output, stale suggestions, concurrent decisions, quota/audit outcomes, provider failures, redaction, and default-off pilot controls across backend, workspace, and admin surfaces. **Status: complete. The release suite covers both configured providers, redaction, bounded tenant/project context, dual-capability and project permission enforcement, prompt injection as untrusted data, strict/normalized structured output, empty recommendations, stale fingerprints, concurrent review decisions, locked quota reservation, post-processing audit failures, and safe provider error responses. Apply and dismiss decisions now emit content-free central audit records, and provider exceptions log only correlation/error codes rather than raw error details. AI Assistance remains default-off and is exposed as an audited Pilot override in admin. Forty focused AI/entitlement tests pass; backend, workspace, and admin production builds pass; the compiled Nest application context initializes; and the development database has no pending migrations.**

## Cross-surface requirements

- `track-a-project-backend`: authoritative validation, authorization, entitlement, activity events, and typed filtering/export.
- `track-a-project`: project settings and reusable task field controls; no client-only enforcement.
- `tracker-admin`: existing entitlement override and audit views expose each Phase 2 capability; no direct field mutation controls.

## Phase 2 release gate

Each epic must have its own stable capability key, backend permission and tenant-isolation tests, activity/audit coverage for important mutations, frontend recovery states, migrations verified in a development environment, and explicit pilot enablement. An epic moves to Validation only after its vertical slice passes across every applicable surface.
