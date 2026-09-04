# Phase 3 Implementation Plan

## Delivery decision

Phase 3 starts with Rule-Based Automation because later recurring-work, approval-reminder, integration-delivery, and conversational-automation features need one durable execution contract. The engine must reuse the existing entitlement, project-role, workflow-transition, Custom Field, template, notification, activity, audit, Redis, and queue boundaries rather than creating privileged mutation paths.

All Phase 3 capabilities remain default-off and are enabled only through the centralized entitlement service and audited operator controls.

## Delivery order

1. **3.1 Rule-Based Automation Builder** — versioned rules, durable events and runs, authorized actions, observability, and loop protection.
2. **3.5 Audit Trail** — generalize the existing audit foundation before more automated writers are added.
3. **3.6 Reliable Integration Delivery** — publish signed, retryable outbound events from the durable audit/event boundary.
4. **3.3 Task Dependencies** — add the execution constraints needed by recurrence, approvals, and later planning.
5. **3.2 Advanced Recurring Work** — extend the existing recurrence engine using automation, dependencies, templates, and calendars.
6. **3.4 Advanced Approvals** — build staged policies on the Basic Approvals snapshot and automation/reminder contracts.
7. **3.8 Semantic Duplicate Suggestions** — optional, review-only assistance using the existing governed AI pipeline.
8. **3.7 Conversational Automation** — optional draft generation after the visual rule contract is stable.

Audit Trail and Reliable Integration Delivery may be developed in parallel after the automation event and run schemas are stable. Optional AI work does not block completion of the non-AI Phase 3 goal.

## 3.1 Rule-Based Automation Builder — MVP contract

### Authorization and execution boundary

- Viewer and Contributor may read enabled-rule summaries and execution results only where those results reference records they can view. They cannot inspect secrets or unrestricted rule configuration.
- Editor may create and edit drafts and run dry tests for projects they can edit.
- Owner may publish, enable, disable, and archive rules and may choose the stored project-role authorization policy used at execution time.
- A rule belongs to one organization and one project. Every referenced status, Custom Field, template, member, watcher, and task must belong to the same organization and be valid for the rule project unless an action contract explicitly supports another authorized project.
- Published rule versions are immutable. Editing creates or updates a draft; enabling always points to one published version.
- The synthetic `Tailpoint Automation` actor provides attribution only. Each action is authorized at execution time against the stored policy, current entitlement, active project/resources, and the same domain service used by human mutations.
- The creator or last material editor is retained as human attribution after membership changes, but their departure neither grants nor automatically removes execution authority. Disabling the rule or entitlement prevents new runs.
- Rule evaluation and execution are asynchronous. The source mutation commits before an automation event is processed; actions run with stable idempotency keys and never share an unbounded transaction with the source request.

### Initial trigger, condition, and action contract

- Triggers: `task.created`, `task.field_changed`, `task.status_changed`, `task.deadline_reached`, `task.ingested`, and `form.submitted`.
- Conditions: equality/inequality, set membership, presence, typed numeric/date comparisons, actor/project checks, and changed-from/changed-to checks where the trigger supplies a before/after value.
- Actions: assign a project member, update a supported standard or Custom Field, transition status through the central workflow service, add a watcher, notify authorized recipients, and create a task from a published reusable template.
- One rule has exactly one trigger, zero or more AND conditions in the MVP, and one or more ordered actions. OR groups, branching, delays, and cross-organization actions are out of scope.
- Contracts use stable keys and explicit schema versions. Unknown trigger fields, operators, action fields, and resource IDs are rejected at publish time and revalidated at execution time.

### Loop, idempotency, and failure rules

- Each source mutation emits one durable automation event with a stable event ID and correlation ID.
- A unique `(rule_id, rule_version_id, event_id)` identity creates at most one run, including concurrent consumers.
- Every action attempt has a stable idempotency key derived from the run and action key. Retries cannot repeat a completed action.
- Events caused by automation retain a causation chain. A rule may not process an event already containing its rule ID, and the engine enforces bounded chain depth and total-action limits.
- Conditions are evaluated from a bounded trigger snapshot. Actions re-read and revalidate mutable resources immediately before mutation.
- Retry only operational failures. Validation, authorization, disabled-entitlement, missing-resource, and policy failures are terminal and explain their safe error code.
- Disabling a rule prevents queued but unstarted runs. A run already executing may finish its current atomic action, then stops before the next action.

### Persistence

- `automation_rules`: organization, project, stable key, name, description, active/published/draft pointers, authorization policy, creator, last material editor, archive timestamp, timestamps.
- `automation_rule_versions`: rule, version number, state (`draft`, `published`, `retired`), schema version, immutable trigger/conditions/actions snapshot, creator/publisher, timestamps.
- `automation_events`: organization, project, event type, subject type/id, safe before/after snapshot, actor type/id, correlation/causation IDs, chain metadata, occurred/available timestamps, timestamps.
- `automation_runs`: organization, project, rule/version/event, state, condition trace, matched flag, attempt count, started/finished timestamps, safe failure code, timestamps, unique execution identity.
- `automation_action_attempts`: run, stable action key, idempotency key, state, attempt number, safe input/result snapshots, started/finished timestamps, failure code, unique action-attempt identity.
- Synthetic actor identity is organization-scoped and provisioned through an idempotent service; it is not an interactive user and cannot authenticate.

### API tickets

- **AU-01 — Capability and contract**: register `rule_based_automation` as default-off across backend, workspace, and admin; document authorization, versioning, execution, idempotency, loop, attribution, and redaction invariants. **Status: complete.**
- **AU-02 — Versioned persistence**: add rule/version/event/run/action-attempt entities, constraints, indexes, synthetic actor identity, and a reversible migration. **Status: complete. Six persistence entities are registered in runtime and CLI data sources; development migration up/down/up is verified. Rule-version, run-execution, action-attempt, idempotency, actor, tenant/project, correlation, and operational indexes are enforced.**
- **AU-03 — Builder and publish API**: list/read/create/archive rules, update drafts, validate contracts and referenced resources, publish immutable versions, and enable/disable published rules. **Status: complete. Capability-gated project APIs enforce editor draft access and owner-only publish/enable/disable/archive operations; viewers receive enabled summaries only. Stable versioned trigger/condition/action contracts reject unknown keys and cross-project or stale status, Custom Field, member, form, and template references. Publishing retires the prior immutable version transactionally, activation revalidates the published definition, synthetic actors are provisioned idempotently, and lifecycle mutations emit safe project activity. Focused service/guard tests, typecheck, build, and compiled runtime initialization pass.**
- **AU-04 — Durable event capture**: emit transactionally safe events for the initial triggers and deadline scheduler without delaying source requests or leaking unsafe payloads. **Status: complete. A Nest-registered TypeORM subscriber captures task creation plus standard and Custom Field/status changes through the mutation transaction regardless of source channel; accepted form submissions emit a deduplicated form event beside their task/submission commit. Capability-disabled organizations retain no automation events. Snapshots are allowlisted, depth/count/string bounded, and project scoped. A non-overlapping minute scheduler captures non-terminal deadlines from active published rules with a bounded one-day recovery window and deterministic event identities. The reversible dedupe migration, 26 focused automation/form tests, typecheck, build, and runtime subscriber registration are verified.**
- **AU-05 — Execution engine**: enqueue matched rules, evaluate typed conditions, enforce run/action idempotency and loop bounds, revalidate authorization/resources, and execute through domain services. **Status: complete. A durable database-backed matcher creates concurrency-safe rule/version/event runs only for rules active when the event occurred. A bounded non-overlapping worker claims runs, recovers expired leases, retries operational failures up to three times, and records terminal failures without retry. Typed equality, membership, presence, numeric/date comparison, and changed-from/to conditions are supported. Stable action identities make assignment, standard/Custom Field updates, workflow transitions, watchers, notifications, and task-template creation retry-safe; each mutation and its success marker commit atomically. Current capability, project, synthetic actor, membership, status, workflow edge/role/requirements, Custom Field, and task-template resources are revalidated. Async execution context propagates correlation, causation, ancestry, depth, and action count into generated events; self/mutual recursion is bounded at 10 levels and 50 actions. Task watchers and forward-only rule activation have verified reversible migrations. Focused tests, typecheck, build, and compiled runtime engine initialization pass.**
- **AU-06 — Dry run and observability API**: evaluate a draft against an authorized sample or synthetic payload without mutation; expose bounded run, condition, action, retry, and failure details. **Status: complete. Draft dry runs accept only project-scoped persisted events or bounded, redacted synthetic snapshots, reuse definition/resource validation and condition semantics, return explicit non-mutation and would-run action results, and never create events, runs, attempts, or domain mutations. Tenant-scoped history and detail APIs provide bounded condition traces, action attempts, retry counts, safe failure codes, correlation metadata, timestamps, filters, and pagination.**
- **AU-07 — Workspace builder and run history**: add capability-gated rule list, guided trigger/condition/action editor, validation, publish/enable controls, dry-run results, and execution history with loading, empty, error, and retry states. **Status: complete. The project workspace now includes a capability-gated Automations surface with rule status/list selection, guided trigger/condition/action authoring, server validation, owner-only publish/enable/disable controls, synthetic dry-run previews, execution summaries, and explicit loading, empty, error, and retry states.**
- **AU-08 — Validation and pilot**: cover permissions, tenant isolation, stale resources, workflow/custom-field/template integration, concurrency, retries, disable races, recursive chains, attribution, redaction, queue fallback, migration, builds, and audited admin pilot controls. **Status: complete. Existing capability, authorization, resource, engine, event, idempotency, recursion, retry, race, attribution, and fallback coverage remains green; the new endpoints preserve project/organization guards and bounded redaction. The focused automation suite (26 tests), backend typecheck/build, frontend typecheck, and diff checks pass. The production frontend compile reaches only the pre-existing external Google Fonts fetch and is blocked in the offline environment; no local TypeScript or automation compile error remains. Pilot rollout continues through the existing default-off, audited organization capability controls.**

### Automation follow-up backlog

- [x] Complete watcher behavior with downstream task-change notifications and watcher management.
- [x] Preserve human attribution across every task mutation channel.
- [x] Capture task-created snapshots after initial assignees and Custom Fields are committed.
- [x] Support multiple actions in the visual builder.
- [x] Add Custom Field, published-form, and missing typed-operator selectors to the builder.
- [x] Route automation notifications through the full preference and push-delivery pipeline.
- [x] Expand run details and add controlled retry controls.
- [x] Add execution-history pagination in the workspace, using the existing bounded server `limit`/`offset` contract with loading, empty-page, previous/next, and total-result states.
- [x] Emit a deduplicated `task.ingested` trigger for accepted API, SDK, import, webhook, and email intake outcomes, with source/outcome conditions and a preference-gated owner/assignee notification.

### Acceptance test matrix

- Direct and queued execution cannot bypass project roles, workflow transitions, typed Custom Field validation, template compatibility, active membership, or entitlement checks.
- Cross-organization and cross-project IDs are rejected without leaking record existence.
- Draft edits never affect enabled execution until a valid immutable version is published and selected.
- Concurrent delivery of one event creates one run and each completed action occurs at most once.
- Self-triggering and mutually triggering rules terminate through ancestry and depth/action bounds, with an observable reason.
- Disabling a rule or capability stops unstarted executions; stale references fail safely without partial unauthorized work.
- Dry runs perform no writes or notifications and clearly distinguish matched conditions from proposed actions.
- Logs retain safe attribution, correlation, condition outcomes, and action results without secrets, tokens, message bodies, attachment contents, or unrestricted personal data.
- Automation-created mutations remain visible in normal project activity and identify both the synthetic actor and the responsible human rule editor.
- The engine behaves correctly with inline development execution and the supported Redis-backed production queue strategy.

## 3.5 Advanced Audit Trail — MVP contract

### Audit boundary and invariants

- `audit_logs` becomes the canonical, append-only organization event ledger. Existing administrative entries are migrated in place; the implementation must not introduce a second audit table or silently discard legacy rows.
- Every new entry has a stable event ID, schema version, organization, action, subject type/ID, actor attribution, occurrence timestamp, and request correlation ID. Project, source, causation, safe before/after changes, and supplementary metadata are nullable only when the event contract does not provide them.
- Audit writes for a successful mutation occur in the same database transaction as that mutation. Failed or rolled-back mutations do not leave success entries. Security-relevant denied attempts may be recorded separately with an explicit outcome and no leaked resource details.
- Rows are immutable through application code: there is no update endpoint, correction creates a new linked event, and retention is the only supported deletion path. Database enforcement must reject mutation outside the narrowly scoped retention worker.
- Audit event capture is synchronous database work only. It must not call webhooks or queues in the mutation transaction. Phase 3.6 consumes committed event IDs through a checkpointed publisher and preserves the original event identity.
- Actor and subject display data are bounded snapshots so history remains understandable after rename or deletion. Snapshot labels are not authorization sources and must not contain unrestricted personal data.

### Event and actor contract

- Actor types are `human`, `automation`, `system`, and `admin`. Human events store the organization user ID; automation events store the organization-scoped synthetic actor and responsible rule editor/publisher where available; admin events store the platform administrator; system events use a stable service key rather than a fabricated user.
- Actor attribution includes `actor_type`, nullable `actor_id`, a bounded safe label, and nullable `responsible_user_id`. Impersonated actions identify both the effective human actor and the initiating administrator. Deleted users remain attributable through immutable IDs and safe snapshots.
- Actions use stable, namespaced past-tense keys such as `task.updated` and `automation_rule.enabled`; display text is a client concern. Subject types use an allowlisted stable registry rather than entity or table names supplied by callers.
- Before/after payloads contain changed allowlisted fields only. The central sanitizer applies field allowlists plus recursive key denial, depth, item-count, and string-size bounds. Passwords, tokens, secrets, cookies, authorization headers, raw request bodies, message/file contents, signed URLs, provider payloads, and unrestricted Custom Field values are never recorded.
- Values requiring audit usefulness but carrying elevated sensitivity are represented by safe summaries (for example `changed`, counts, authorized IDs, or irreversible fingerprints), never reversible masking.
- Correlation and causation IDs flow from human requests, automation execution context, scheduled/system jobs, and later integration deliveries. A nullable idempotency/source key prevents duplicate entries where a producer can retry.

### Persistence and migration

- Evolve `audit_logs` with `schema_version`, `project_id`, `actor_type`, `actor_id`, `actor_label`, `responsible_user_id`, `subject_type`, `subject_id`, `subject_label`, `source`, `outcome`, `before_changes`, `after_changes`, `request_id`, `correlation_id`, `causation_id`, `source_event_key`, `occurred_at`, and `retention_expires_at`, while retaining compatible `metadata` and legacy `admin_id`, `target_user_id`, `action`, and `created_at` during migration.
- Backfill legacy rows deterministically as version-1 admin/system events without inventing unavailable subjects or actors. Reads normalize legacy and current rows through one response contract; writers use only the generalized service after migration.
- Index organization/time first, then supported filters: project/time, action/time, subject/time, actor/time, correlation, and retention expiry. Enforce a unique nullable `(organization_id, source, source_event_key)` identity for retrying producers.
- Foreign keys that would erase history use `SET NULL` only where an immutable scalar ID/snapshot remains. Organization deletion follows the product data-lifecycle policy; ordinary project, user, task, rule, form, workflow, template, and entitlement changes cannot cascade-delete audit entries.
- Retention is organization-scoped, defaults to the product policy, cannot go below the platform minimum, and changes apply prospectively unless an authorized operator explicitly schedules a bounded purge. Purges are chunked, resumable, observable, and record aggregate proof outside the deleted range without copying deleted payloads.

### Authorization, API, and export contract

- The capability is default-off and enforced server-side. Organization admins may view the organization ledger and manage retention/export. Project Owners may view entries for projects they currently own, but cannot export or infer organization-wide events. Other project roles have no Advanced Audit Trail API access in the MVP.
- Platform administrators use a separate admin route and explicit support permission. They receive event headers and safe metadata by default; before/after details require an audited break-glass reason. Impersonation alone never grants broader audit access.
- List APIs require organization scope, use deterministic cursor pagination (`occurred_at`, `id`), and support bounded date range plus project, action, actor type/ID, subject type/ID, source, outcome, and correlation filters. Invalid or cross-tenant identifiers return a non-enumerating response.
- Detail responses return only fields authorized at read time. A project-scoped reader cannot follow correlations into inaccessible projects or organization-only events, and deleted/inaccessible subjects do not cause raw payload disclosure.
- Exports reuse the same authorization and filters, execute asynchronously from a frozen filter/time watermark, stream bounded CSV or JSONL artifacts, expire automatically, and expose status/download only to the requesting organization admin. Export creation, completion, download, expiry, and cancellation are audited without recursively exporting export-control events unless explicitly filtered.
- APIs cap ranges, page sizes, filter counts, and export size. CSV output is protected against formula injection; JSONL/CSV serialization cannot reintroduce redacted fields.

### Coverage contract

- Initial domain coverage includes create/update/archive/delete and permission-relevant lifecycle events for projects, tasks, workflows/status transitions, request forms/submissions, reusable templates, entitlement overrides, and automation rules/runs/manual retries.
- Domain services emit audit records through one typed audit writer and explicit event definitions. Controllers, TypeORM subscribers, and generic entity diffs are not the primary write boundary because they cannot reliably preserve authorization, transaction, or semantic context.
- Task and automation activity feeds remain user-facing collaboration history; they may reference audit event IDs but do not replace or duplicate the security ledger contract.

### Delivery tickets

- **AT-01 — Capability, registry, and redaction contract**: register `advanced_audit_trail` as default-off across backend, workspace, and admin; add the typed action/subject/actor registry, central allowlist sanitizer, payload bounds, and the authorization/retention/export contract. **Status: complete. The default-off capability is served by the centralized backend catalog to the dynamic admin entitlement controls and is registered in the workspace capability contract. Typed action, subject, actor, source, outcome, export-format, retention, and payload-limit contracts are centralized. Subject-specific allowlists, recursive denied-key filtering, and depth, count, string, and metadata-size bounds are covered by focused tests.**
- **AT-02 — Generalized append-only persistence**: evolve `audit_logs`, add constraints/indexes/immutability enforcement and the typed audit writer, migrate legacy admin rows compatibly, and verify reversible schema migration plus forward data preservation. **Status: complete. The existing ledger is extended in place, legacy rows receive deterministic version-1 attribution, supported filter and retry-deduplication indexes are defined, project deletion preserves scalar history, and database triggers reject application updates and non-retention deletes. The version-2 writer requires the caller's active transaction, sanitizes every payload, calculates retention expiry, and resolves retry identities idempotently. A configured MySQL up/down/up rehearsal preserved the generalized ledger and exposed an organization-foreign-key index-order defect in the down path; the migration now installs a temporary supporting index before removing generalized indexes and replaces it only after the forward organization/time index exists. Focused migration/writer tests, backend typecheck, and build pass.**
- **AT-03 — Core domain capture**: transactionally cover project and task create/update/archive/delete, assignments, Custom Fields, workflow/status transitions, and actor/correlation propagation without duplicating activity entries. **Status: complete for the currently supported project/task lifecycle. Project create/update/delete and task create/update/delete/status-transition/priority events use the typed writer in the domain mutation transaction and are default-off through the centralized entitlement. General and attachment-based task updates commit standard fields, assignment changes, Custom Field summaries, workflow transitions, and their audit events atomically; notifications remain post-commit. Task creation commits initial assignees before capturing its final safe snapshot. HTTP request identity propagates through async context so related events share bounded request/correlation IDs; non-HTTP work receives a generated correlation. Existing activity feeds remain separate, and the unused non-audited assignment mutation path was removed. Project/task archive operations do not exist in the current domain model; automation/system attribution is covered by AT-04.**
- **AT-04 — Extended domain and automation capture**: cover workflows, forms/submissions, templates, entitlement overrides, automation rule lifecycle, runs, retries, system schedulers, and admin/impersonated actions with human/automation/system/admin attribution. **Status: complete. Automation rule lifecycle, run completion/skip/failure, and manual retries now use transactionally coupled typed events with human or organization-automation attribution, propagated correlation/causation, and retry-stable source identities. Entitlement overrides, workflow lifecycle, reusable templates, and request-form lifecycle are covered without retaining definitions, snapshots, answers, attachments, or other content. Accepted, pending-review, and rejected submissions record only status, form ID, and created-task ID with human or stable public-intake system attribution. Approval requests, responses, invalidations, and scheduled reminders exclude messages, reviewer identities, comments, and subject snapshots. Subscription changes, project-member role changes, and impersonation use typed admin/system events; impersonation preserves both effective and responsible actor identity. AI suggestion application and dismissal record safe status/count/task summaries and retain suggestion correlation without prompt, note, or generated content. The extended-domain writer sweep found no remaining direct generalized `AuditLog` writers outside the canonical writer; the separate AI-governance request telemetry remains purpose-specific and is not a second domain ledger.**
- **AT-05 — Filtered audit API**: add permission-gated organization/project list and detail endpoints with deterministic cursor pagination, bounded filters, correlation lookup, read-time subject authorization, and non-enumerating tenant isolation. **Status: complete. The capability-gated `audit-events` API normalizes legacy and version-2 rows through one response, bounds pages to 100 and date windows to 90 days, supports project/action/actor/subject/source/outcome/correlation filters, and orders by occurrence time plus event ID with opaque stable cursors. Organization administrators receive tenant scope; other members must supply a currently owned project and are restricted to it for both list and detail reads. Missing, cross-tenant, and unauthorized entries share a non-enumerating not-found response, and stored metadata is recursively sanitized again at read time.**
- **AT-06 — Workspace audit history**: add a capability-gated organization-admin audit surface plus project-owner scoped entry point, filters, pagination, actor/subject presentation, redacted before/after detail views, and loading/empty/error/retry states. **Status: complete. A responsive audit history surface is available from workspace admin settings and from project settings for current owners. It uses the same filtered API, presents stable actor/subject/source/outcome attribution, supports action, actor, source, date, and correlation filters, expands safe before/after changes, and provides deterministic load-more pagination plus explicit capability-disabled, loading, empty, error, and retry states. Project owners enter through the neutral dashboard route so the admin layout cannot broaden or block their project-scoped authorization.**
- **AT-07 — Admin review, export, and retention**: add separate support-review controls, audited break-glass detail access, asynchronous safe CSV/JSONL export lifecycle, organization retention settings, and bounded resumable purge operations. **Status: complete. Separate super-admin support routes expose headers only until a bounded reason is supplied, then transactionally record break-glass access. Organization-admin exports freeze filters and a time watermark, bind status/download to the requester and tenant, exclude export-control recursion, sanitize again through the reader, neutralize CSV formulas, support JSONL, cancellation, and 24-hour expiry. Retention is prospective by default; explicit application to existing data schedules observable, resumable 500-row purge chunks through the database-enforced retention-worker path. Workspace controls expose retention and export creation/status. Configured MySQL rehearsal and authenticated lifecycle checks pass.**
- **AT-08 — Validation and pilot**: cover permission matrices, tenant/project/correlation isolation, allowlists and recursive redaction, all four actor types, impersonation, transaction rollback, retry deduplication, concurrent writes, immutability, migration/backfill/down safety, cursor stability, export injection/isolation/expiry, retention races, builds, and audited pilot controls. **Status: complete for pilot. The focused audit suite covers canonical writer transactions/deduplication, recursive redaction bounds, tenant/project reads, cursor ordering, migration preservation, request correlation, and actor attribution. The configured MySQL up/down/up rehearsal passes with ledger data preserved and pre-existing export records restored; live database checks confirm update/delete immutability. Export and purge workers use conditional state claims so concurrent workers cannot process the same export or purge chunk. Authenticated HTTP checks verify organization-admin list/retention access, project-owner scoped reads, contributor/viewer denial, project-owner export denial, cross-organization blocking, requester-bound downloads, cancellation, expiry and expiry auditing, CSV formula neutralization, prospective retention changes, and audited super-admin break-glass detail access. Backend typecheck/build and both production frontend builds pass; the focused audit gate is 19/19. The broader backend suite has 305 passing tests and 30 unrelated legacy scaffold failures caused by omitted Nest test providers. Pilot rollout remains default-off and must be enabled only through audited organization capability controls.**

### Acceptance test matrix

- A domain mutation and its audit entry commit or roll back together; concurrent mutations produce distinct ordered events, while retrying one source identity produces one event.
- Human, automation, system, admin, and impersonated actions retain accurate effective/responsible attribution after users, rules, projects, or subjects are renamed or deleted.
- Secrets and denied keys are absent at rest, in list/detail responses, logs, exports, fixtures, and error messages; nested or oversized payloads are safely bounded.
- Organization admins never read another tenant's events. Project Owners see only currently authorized project events, including through subject, actor, and correlation filters.
- Cursor pagination is stable when new events arrive between pages and uses a deterministic `occurred_at`/ID order without duplicates or omissions inside the captured window.
- Existing administrative audit rows remain readable after up migration; new rows survive supported down/up rehearsal without silent data loss or semantic reassignment.
- Application users cannot update or delete audit rows. Retention purges only eligible tenant rows in bounded chunks and cannot race into newly extended or held data.
- Export contents equal the frozen authorized filter set, neutralize spreadsheet formulas, expire as configured, and cannot be downloaded across users or organizations.
- Disabling the capability closes UI/API access and new premium capture behavior as defined by rollout policy without weakening mandatory security audit records.
- Committed events expose the stable IDs and checkpoints required by Phase 3.6 without making outbound delivery part of the source transaction.

## 3.6 Reliable Integration Delivery — MVP contract

### Delivery boundary and security

- Organization administrators configure organization-scoped HTTPS endpoints and an explicit allowlist of supported audit actions. Project-scoped endpoints may additionally restrict delivery to one project. Private, loopback, link-local, credential-bearing, and non-HTTPS destinations are rejected and revalidated before every request.
- A checkpointed publisher consumes committed version-2 audit rows after their source transaction commits. It creates one immutable delivery per `(endpoint_id, audit_event_id)` and never performs network work in the domain mutation transaction.
- Payloads use a versioned public envelope containing the original audit event ID, occurrence time, organization/project scope, action, outcome, safe actor/subject attribution, correlation/causation IDs, and the already-sanitized metadata/change summaries. Legacy audit rows and audit/export/integration-control events are not published.
- Each request includes a stable delivery ID, event ID, timestamp, payload version, and HMAC-SHA256 signature. Secrets are encrypted at rest, shown only once when created or rotated, never logged/audited/exported, and support a bounded overlap window during rotation.

### Reliability and replay

- Delivery is at-least-once. HTTP 2xx succeeds; 408, 425, 429, and 5xx responses retry on a bounded exponential schedule with jitter and `Retry-After` support; other 4xx responses fail permanently. Network timeouts and response bodies are bounded.
- Workers claim queued attempts with leases, recover abandoned leases, and use conditional state transitions so concurrent workers cannot send the same attempt. Endpoint disablement stops unstarted work without altering completed history.
- Exhausted deliveries enter `dead_letter`. Authorized replay creates a new delivery generation for the same endpoint and original event ID; it never forges a new audit event. Manual replay requires a bounded reason and is itself audited.
- Delivery history stores timing, attempt number, status code, bounded response/error codes, and next-attempt time, but never response bodies, secrets, tokens, or full request payloads. Retention follows bounded platform policy independently of the audit ledger.

### API and product surfaces

- Organization admins may create, list, update, disable, rotate, test, inspect delivery history, and replay dead letters. Project Owners may view delivery status for their owned project but cannot reveal endpoints/secrets, change configuration, or replay organization deliveries. Other roles have no access.
- Test delivery uses a synthetic, clearly marked payload and does not create an audit event. It follows the same destination validation, signing, timeout, and observability rules as production delivery.
- Workspace settings provide endpoint configuration, one-time secret handling, event selection, health/history, retry countdowns, dead-letter replay with confirmation/reason, and loading/empty/error/recovery states. Admin provides entitlement rollout plus safe aggregate health/failure summaries only.

### Delivery tickets

- **RI-01 — Capability and public contract**: register `reliable_integration_delivery` as default-off; define endpoint, envelope, signature, retry, replay, authorization, SSRF, redaction, and retention contracts. **Status: complete. The capability is registered across backend and workspace contracts, depends on Advanced Audit Trail at every API/publisher/send boundary, and the public envelope, HMAC verification, retry, replay, and receiver-deduplication contract is documented.**
- **RI-02 — Persistence and migration**: add endpoints, publisher checkpoints, deliveries, attempts, uniqueness/lease indexes, encrypted secret versions, and a reversible migration. **Status: complete. Four tenant-scoped tables preserve endpoint configuration, a locked audit cursor, stable event/generation identities, worker leases, and content-free attempt history. Secrets use authenticated encryption. The development up/down/up rehearsal passes and the migration is applied.**
- **RI-03 — Post-commit publisher**: checkpoint eligible committed audit events, apply endpoint/action/project filters, and create deduplicated deliveries without delaying source mutations. **Status: complete. A transactionally locked cursor consumes only committed version-2 audit rows, advances deterministically by occurrence time/event ID, rechecks both capabilities, excludes audit/integration control recursion, applies tenant/project/action filters, and upserts generation-one deliveries by stable identity.**
- **RI-04 — Signed delivery worker**: implement destination revalidation, bounded requests, versioned HMAC signing, retry classification/backoff, leases, disable races, and dead-letter transitions. **Status: complete. The database worker conditionally claims due work, recovers expired leases, rechecks endpoint/capability state, validates all DNS answers, pins a public address to prevent rebinding, rejects redirects, bounds time/body sizes, signs exact bytes with current/overlap secrets, honors bounded Retry-After, and classifies retryable, permanent, and exhausted outcomes.**
- **RI-05 — Management and observability API**: implement scoped endpoint CRUD/rotation/test operations, delivery filters/detail, safe attempt history, and reason-gated replay preserving original event identity. **Status: complete. Organization-admin mutation APIs are tenant-scoped and audited without secrets or URLs; project Owners receive non-enumerating read-only project history. Synthetic tests use the production signing/network boundary without creating source events. Dead-letter replay requires a reason and creates a new generation for the original event ID.**
- **RI-06 — Workspace integrations surface**: add organization configuration and project-owner status views with one-time secrets, filters, health, history, test, rotation, disable, and replay recovery UX. **Status: complete. Workspace settings expose endpoint setup, one-time copyable secrets, event selection, test/rotate/disable operations, delivery status and dead-letter replay; project settings link Owners to a read-only scoped history with loading, empty, error, refresh, and failure states.**
- **RI-07 — Admin operations surface**: expose audited pilot controls and tenant-safe aggregate delivery health/failure summaries without endpoint URLs, secrets, or payload contents. **Status: complete. The existing dynamic entitlement controls expose default-off rollout, and a protected support view reports only per-organization endpoint and delivery-state counts.**
- **RI-08 — Validation and pilot**: cover transaction separation, deduplication, ordering, signatures/rotation, SSRF/DNS rebinding, tenant/project authorization, retry/dead-letter/replay, concurrent leases, disable races, redaction, migrations, builds, and default-off pilot activation. **Status: complete for development pilot. The focused integration/audit gate passes 34 tests covering public/private address classification, mixed DNS rejection, exact-byte signatures, rotation overlap, bounded Retry-After, HTTP outcome classification, capability disablement, project-owner scoping, replay identity, conditional lease recovery, checkpointed publication, schema constraints, sanitizer/writer behavior, and migration compatibility. The development migration passes up/down/up and is applied; backend typecheck/build/runtime initialization and both frontend typechecks/production builds pass. Pilot activation remains an audited operator decision.**

### Acceptance test matrix

- Rolling back a domain mutation creates neither a publishable event nor a delivery; committed eligible events eventually create exactly one delivery per endpoint generation.
- Payloads and operational records contain no secrets, credentials, denied metadata, unrestricted content, or receiver response bodies.
- Signatures verify against the documented canonical bytes during current/previous-secret overlap and fail after overlap expiry.
- Unsafe destinations are rejected at configuration and send time; redirects cannot escape the validated destination policy.
- Duplicate publishers, workers, timeouts, and retries cannot create duplicate delivery records or overlapping attempt claims.
- Retryable failures follow the bounded schedule; permanent failures and exhausted retries become observable dead letters.
- Replay preserves the original event ID, records a new delivery generation and reason, and cannot cross endpoint, project, or organization scope.
- Disabling the endpoint or capability prevents unstarted sends while retaining authorized, redacted history.

## Remaining Phase 3 epics

### 3.3 Task Dependencies

Status: **Implemented and validated on development**

- **TD-01 — Dependency edge persistence:** organization-scoped `task -> prerequisite` edges, immutable title snapshots, reversible removal, and deletion archival.
- **TD-02 — Authorization and privacy:** contribute permission on the blocked task, view permission on the prerequisite, organization isolation, and redaction of inaccessible linked-task details.
- **TD-03 — Graph integrity:** reject self-dependencies, duplicate active edges, and direct or transitive circular dependencies.
- **TD-04 — Execution warnings:** expose unresolved blocker counts and warn before status changes that conflict with active prerequisites.
- **TD-05 — Workspace editing:** list, add, and remove blocking tasks in the task editor behind the `task_dependencies` capability.
- **TD-06 — History and audit:** retain understandable dependency history after either task is deleted and emit advanced-audit events for edge lifecycle changes.
- **TD-07 — Date propagation:** preview proposed downstream date shifts, return conflicts without mutation, and require an explicit preview token/confirmation before a bulk change.

Acceptance is covered: circular dependencies are rejected; inaccessible dependency details are never leaked; deletion archives edges with snapshots and audit history; status conflicts produce warnings; and bulk downstream date changes require a short-lived, user-bound preview token plus stale-data and permission revalidation before confirmation. The migration has been applied successfully to the development database.

The Advanced Audit Trail and Reliable Integration Delivery contracts are now finalized. Detailed ticket contracts for Advanced Recurring Work, Advanced Approvals, Semantic Duplicate Suggestions, and Conversational Automation will be finalized from the audit/event boundary established by AT-02. No later epic may introduce a second automation executor, a parallel audit ledger, or bypass the central domain services.

### 3.2 Advanced Recurring Work

Status: **Implemented and validated on development**

- **AR-01 — Advanced schedule contract:** extend existing recurrence definitions without creating a second scheduler; retain IANA timezone and retry-safe occurrence identities.
- **AR-02 — Business calendars:** organization-scoped holiday dates and explicit skip or next-business-day behavior, evaluated in the recurrence timezone.
- **AR-03 — Rotation and reusable content:** deterministic assignee rotation and version-pinned reusable checklist/template content.
- **AR-04 — Exceptions:** skip or reschedule one occurrence without rewriting history or silently changing the rest of the series.
- **AR-05 — Effective-dated changes:** preview changes from a selected date, preserve earlier occurrence history, and require confirmation before future generated work is changed.
- **AR-06 — Recovery and observability:** record generated, skipped, and failed outcomes; expose bounded history, failure codes, retry state, and missed-run recovery.
- **AR-07 — Dependency integration:** copy authorized dependency structure to generated work without creating cycles or leaking inaccessible tasks.
- **AR-08 — Workspace and validation:** capability-gated calendar, rotation, exception, future-change, and history controls with migration rehearsal, concurrency tests, builds, and default-off pilot rollout.

Acceptance: scheduler retries never duplicate work; holiday and DST behavior is deterministic; rotation remains stable across retries; exceptions and effective-dated changes preserve history; failures are recoverable and observable; and generation never bypasses current project, dependency, template, or entitlement authorization.

## Cross-surface requirements

### 3.4 Advanced Approvals

Status: **Implemented and validated on development**

- **AP-01 — Immutable policy snapshot:** every advanced request stores its ordered stages, required and optional reviewers, and unanimous or threshold decision rule.
- **AP-02 — Sequential evaluation:** only the active stage can respond; passing advances one stage and the final stage resolves the request.
- **AP-03 — Reviewer semantics:** required rejections are terminal, optional responses are advisory, and response records remain immutable.
- **AP-04 — Delegation:** only an active assigned reviewer may delegate to another active project member, with original attribution retained.
- **AP-05 — Reminders and escalation:** due-soon reminders retain stable delivery keys and overdue requests escalate once to the requester.
- **AP-06 — Audit and subject integrity:** subject snapshots invalidate on later subject changes and existing typed request, response, invalidation, and reminder audit events remain authoritative.
- **AP-07 — Workspace and rollout:** default-off advanced capability, staged-policy controls, active-stage presentation, and dynamic administrative entitlement rollout.

Acceptance is covered: policies cannot change after request creation; stages execute in order; threshold and unanimous policies resolve deterministically under the request lock; reviewers cannot respond twice or outside their stage; delegation is project-scoped; changed subjects invalidate approval; and reminders/escalations remain bounded.

- `track-a-project-backend`: authoritative contracts, authorization, event capture, execution, idempotency, domain-service integration, observability, activity, and audit.
- `track-a-project`: rule authoring, dry runs, execution history, and recovery UX; no client-only validation or execution.
- `tracker-admin`: entitlement override, safe usage/failure summaries, and audit review; no direct rule or execution mutation controls.

## Phase 3 release gate

Each epic needs a stable default-off capability where independently rollable, backend permission and tenant-isolation tests, important activity/audit coverage, bounded and redacted operational records, frontend recovery states, verified reversible migrations, production queue behavior, and explicit pilot enablement. An epic enters Validation only after its vertical slice passes across every applicable surface.
