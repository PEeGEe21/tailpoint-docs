# Tailpoint Product Feature Roadmap

## Purpose

This document turns the proposed AI and non-AI features into a buildable roadmap. The phases are ordered from lower implementation risk to higher implementation risk while respecting feature dependencies.

This is a product and implementation guide, not a promise that every item must ship. Before starting a phase, convert its features into smaller tickets with estimates and assign an owner.

## Product Direction

Tailpoint should become the workspace where teams receive work, organize it through configurable workflows, collaborate with internal and external stakeholders, automate repeatable processes, and understand what needs attention.

AI should enhance this workflow later. It must not be required for core project execution.

## Delivery Rules

Apply these rules to every feature:

1. Build the smallest useful end-to-end version first.
2. Define authorization rules before creating endpoints or screens.
3. Record important mutations in the activity or audit system.
4. Support organization and project isolation in every query.
5. Add backend tests for permissions, validation, and business rules.
6. Add frontend loading, empty, error, and retry states.
7. Put unfinished features behind organization-level feature flags.
8. Measure adoption before expanding the feature.
9. AI-generated changes must be reviewed by a user before they modify project data.
10. AI answers about workspace data must cite records the user is permitted to access.
11. Feature availability must be resolved by one entitlement service combining feature flags, subscription plan, organization settings, and user permissions.
12. Treat `tracker-admin` as a first-class delivery surface. Any feature with rollout, plan, organization override, support, governance, usage, or audit controls must include explicit admin requirements and acceptance criteria.

## Product Surfaces

Roadmap work may span three separately deployed applications:

- `track-a-project`: the organization member and project workspace.
- `track-a-project-backend`: the authoritative API, authorization, entitlement, audit, and business-rule layer.
- `tracker-admin`: the Tailpoint operator console for controlled rollout, subscription support, organization overrides, usage visibility, governance, and audit review.

The admin application must never become an alternate path around backend authorization or entitlement rules. It calls protected backend APIs, uses explicit super-admin permissions, validates every mutation, and records sensitive operational changes in the audit system. A roadmap ticket is not implementation-ready until it states which of these surfaces are affected or explicitly marks a surface as not applicable.

## Status Vocabulary

- `Proposed`: approved as a roadmap candidate but not designed.
- `Ready`: requirements, schema, API, and UI behavior are agreed.
- `In progress`: actively being implemented.
- `Validation`: code is complete and undergoing QA or pilot use.
- `Released`: available to its intended users.
- `Deferred`: deliberately removed from the current delivery window.

## Foundation Gate

**Completion audit (2026-07-24):** Complete. App-level error, global-error, not-found, retry/recovery experiences; profile/account settings; organization administration; authorized billing visibility; onboarding and tour persistence; real Reports and AI Overview surfaces; centralized entitlement/admin override controls; and shared project authorization and activity/audit infrastructure are implemented. Global search combines navigation discovery with organization-scoped, permission-aware results for accessible projects, tasks, documents, personal notes, project resources, document files, and active-participant conversations.

Complete these existing product-quality items before treating the roadmap as expansion work:

- Make global search functional instead of UI-only.
- Add application-level error, not-found, retry, and recovery experiences.
- Finish profile and account settings.
- Finish organization administration for organization owners and admins.
- Expose subscription and billing information to authorized organization users.
- Persist onboarding and product-tour completion.
- Replace or hide placeholder AI Overview and Reports screens.
- Normalize authorization checks across projects, documents, notes, messages, and files.
- Establish a consistent activity-event model for important mutations.
- Add a centralized entitlement check so plan-gated features are enforced by both backend APIs and frontend navigation.
- Add protected `tracker-admin` controls for viewing effective entitlements, changing organization feature overrides, and reviewing the resulting audit history.

These are prerequisites because later automation, reporting, audit, and AI features depend on trustworthy data and permissions.

## Foundation Entitlement Implementation Status

The initial entitlement foundation is implemented on `feature/foundation-entitlements` across the backend, workspace, and `tracker-admin`. It includes stable capability keys, subscription checks, tri-state organization rollout overrides, active-membership checks, a reusable backend capability guard, member-safe entitlement reads, workspace capability/navigation helpers, protected admin controls, audit records, and a feature-overrides migration. New roadmap features must register a capability and use this foundation rather than adding local plan or flag checks.

---

# Phase 1: Everyday Productivity

## Goal

Increase the value of the existing task, project, note, message, and whiteboard modules without introducing large new infrastructure.

**Phase completion audit (2026-07-24):** Complete. The Foundation Gate and MVP implementation for sections 1.1 through 1.9 are complete across the applicable product surfaces, migrations are deployed in the audited development environment, one pilot organization is enabled, and the focused Phase 1 smoke and backend suites pass. Production release verification remains a separate operational gate.

## 1.1 Personal Productivity Hub

**Implementation status (feature branch):** The MVP vertical slice is implemented across all three product surfaces on `feature/personal-productivity-hub`. The backend provides capability-protected, permission-aware cross-project views, shared-filter counts, sorting, pagination, saved views, and quick-update support. The workspace provides the gated route, navigation, filters, saved-view controls, task states, and quick status changes. `tracker-admin` provides explicit organization pilot enablement through audited entitlement overrides. Development migrations are deployed and the capability was enabled for the `grace-filled` pilot organization on 2026-07-22; production deployment and pilot acceptance remain pending.

**Problem:** Users must enter individual projects to discover what they need to do.

**MVP experience:** Add a personal page containing `My Tasks`, `Today`, `Upcoming`, `Overdue`, and `Waiting On`. Users can filter by organization, project, priority, status, and due date. They can save a filter and choose a default view.

**Waiting On scope:** For the MVP, `Waiting On` means a task whose current workflow status is named `Waiting On` (matched case-insensitively). This is an intentional compatibility rule for the existing task model, not the long-term dependency model. Keep the API view key stable so the implementation can later move to structured dependencies without requiring a frontend contract change.

**Implementation work:**

- Backend: create an authenticated cross-project task query constrained by organization membership and project permissions.
- Frontend: add list sections, filters, sorting, pagination, empty states, and quick status updates.
- Data: add saved-view records containing owner, scope, filters, sort, and visibility.
- Notifications: support optional personal reminders without changing the task deadline.

**Later — structured task dependencies:** Replace the status-name rule with unresolved dependency records kept separate from workflow status. Support dependencies on another task, a person or reviewer, an external party or event, a future date, and a manual hold. The design must include organization and project authorization for linked records, self-dependency and cycle prevention, automatic resolution for completed prerequisite tasks and elapsed dates, explicit resolution for other dependency types, audit history, and waiting-reason/dependency-count response fields. This enhancement is deliberately outside the Productivity Hub MVP so it does not block saved views, quick updates, frontend delivery, or pilot validation.

**Acceptance criteria:**

- A user sees only tasks from projects they may access.
- Counts and results use identical filters.
- A saved view restores its filters and ordering.
- The MVP `Waiting On` view consistently matches the existing `Waiting On` workflow status without requiring structured dependency data.
- Quick edits update both the personal view and project views.
- The page performs acceptably with a large number of tasks through server pagination.

**Dependencies:** Functional authorization and task filtering.

## 1.2 Basic Recurring Tasks

**Implementation status (feature branch):** The MVP vertical slice is implemented on `feature/basic-recurring-tasks`. It includes an entitlement-protected recurrence model, retry-safe occurrence records, daily/weekly/monthly/selected-weekday rules, completion-triggered and before-due generation, transactional task-plus-recurrence creation, occurrence-versus-future edit scope, read-only series details, pause/resume/removal controls, and explicit pilot rollout through `tracker-admin`. Development migrations are deployed and `recurring_tasks` was enabled for the `grace-filled` pilot organization on 2026-07-22; production deployment and pilot acceptance remain pending.

**Problem:** Repeated operational work must be recreated manually.

**MVP experience:** A task can repeat daily, weekly, monthly, or on selected weekdays. The user chooses whether the next occurrence is generated on completion or a fixed time before its due date.

**Implementation work:**

- Add recurrence rule, timezone, next-run time, start/end date, and active state.
- Use an idempotent scheduled job to generate occurrences.
- Link generated tasks to the recurrence definition and previous occurrence.
- Add controls to edit only one occurrence or future occurrences.

**Acceptance criteria:**

- A recurrence never produces duplicate occurrences when a job retries.
- Dates respect the configured organization/user timezone.
- Pausing prevents new tasks without deleting existing tasks.
- Users without task-creation permission cannot create recurrences.

**Later:** Custom calendar expressions, holiday skipping, and assignee rotation belong in Phase 3.

## 1.3 Structured Project Updates

**Status:** Validation on `feature/structured-project-updates`. The MVP and Project Member Roles follow-up are implemented across the backend, workspace, and admin. Development migrations are deployed and the capability was enabled for the `grace-filled` pilot organization on 2026-07-22; production deployment and pilot acceptance remain operational follow-up.

**Problem:** Stakeholders cannot quickly understand project progress or review historical status reports.

**MVP experience:** A project owner publishes an update with health (`on track`, `at risk`, or `off track`), accomplishments, blockers, next steps, and an optional reporting period.

**Implementation work:**

- Create immutable published update snapshots with draft support.
- Allow references to tasks, documents, and users. Enable milestone references after the dedicated milestone model in Phase 2.4 is implemented.
- Notify every connected project member when an update is published, subject to each member's notification preferences.
- Add an update history to the project screen.

**Acceptance criteria:**

- Drafts are visible only to permitted editors.
- Published updates retain their original content when linked tasks change.
- The author can correct an update while preserving edit history.
- Notification recipients respect project membership and preferences.
- Drafts can be deleted by permitted editors; published snapshots cannot be deleted.
- Update history is paginated on the server and remains usable for long-running projects.

**Completed follow-up — project roles management:** Project memberships now support `viewer`, `contributor`, `editor`, and `owner`. Existing members were backfilled as editors, invitation-time role selection is supported, the creator remains a protected owner, additional owners may be assigned, role changes are audited, and the shared policy is enforced across project updates and other project-scoped editing surfaces.

**Milestone dependency:** Milestones must be dedicated project records rather than task flags. Phase 2.5 owns the milestone model, including title, description, target date, status, owner, achieved date, and linked tasks. Structured Project Updates keeps a reserved milestone reference type but must reject milestone references until it can validate those records through project and organization authorization.

## 1.4 Decision Register

**Problem:** Important decisions disappear inside messages, notes, and meetings.

**MVP experience:** Users record a decision, context, owner, date, status, and related project records. A decision can supersede an earlier decision.

**Implementation work:**

- Add project-scoped decision CRUD and relationships to tasks, messages, notes, and documents.
- Provide `proposed`, `accepted`, `rejected`, and `superseded` states.
- Show decision history and backlinks from related records.

**Acceptance criteria:**

- Related records are permission-checked before linking or displaying.
- Superseding a decision does not erase its history.
- Users can filter decisions by state, owner, and date.

**Completed MVP:** Implemented on `feature/decision-register` across the API, project app, and admin app. The register includes typed, project-validated links; project-role authorization; server-side decision and history pagination; immutable accepted, rejected, and superseded history; administrative corrections; transactional supersession; project-details UI; and organization enablement controls. Development migrations are deployed and the capability was enabled for the `grace-filled` pilot organization on 2026-07-22; it remains default-off elsewhere.

## 1.5 Task Discussion Improvements

**Completed MVP:** Added task-scoped threaded replies, validated and deduplicated mentions with notifications, emoji reactions, stable comment permalinks, resolved/reopened threads, soft deletion that preserves replies, author/owner editing with edit history, and server-side root-thread pagination that keeps replies with their thread.

**Problem:** Long task conversations are difficult to follow and resolve.

**MVP experience:** Support threaded replies, mentions, reactions, comment permalinks, and resolved threads.

**Implementation work:**

- Extend the comment model with parent comment, resolution state, and edit metadata.
- Reuse notification and realtime services for mentions, replies, and reactions.
- Add pagination that does not break thread ordering.

**Acceptance criteria:**

- Deleted comments do not orphan or expose restricted replies.
- Mention notifications are deduplicated.
- A permalink opens and highlights the correct comment.
- Resolution and edits appear in activity history.

## 1.6 Work-to-Workflow Conversion

**Implementation status (feature branch):** Complete for the MVP. Whiteboard objects can be previewed and converted transactionally into linked tasks or notes, while selected project tasks can be saved as ordered workflow templates, previewed, and instantiated in authorized projects. Stable conversion claims enforce explicit duplicate confirmation, source records remain unchanged, project-role authorization is enforced, and the workspace exposes conversion and reusable-workflow dialogs. Dependency graphs, cross-organization templates, automatic synchronization, workflow versioning, and automation rules remain deferred as documented below.

**Problem:** Planning performed on a whiteboard must be manually reproduced as actionable work, while useful task structures cannot be captured as reusable workflows.

**Compact MVP:** Ship two conversion paths backed by one conversion record and one preview flow:

1. Select whiteboard objects and create linked tasks or notes in the same project.
2. Select project tasks, save them as an ordered reusable workflow template, and instantiate that template in an authorized project.

Source whiteboard objects and tasks remain unchanged.

**Implementation work:**

- Add stable whiteboard object IDs and a conversion record linking each source to its created record.
- Add workflow template and ordered workflow-step records.
- Use one preview/mapping dialog for title, description, target status, assignee handling, and due date.
- Support `retain assignee` or `leave unassigned`; map due dates to offsets from the earliest selected task.
- Add endpoints for whiteboard conversion, template creation, template preview, and template instantiation.
- Require Editor or Owner access and validate both source and destination projects.
- Require explicit confirmation when a source was already converted.

**Acceptance criteria:**

- Conversion is transactional for the selected batch and reports any validation failure before writing.
- Duplicate conversion requires confirmation.
- Removing a board object does not delete the created task.
- Converting tasks does not mutate, move, or delete the source tasks.
- Only Editors and Owners can create reusable workflows from project tasks.
- Users cannot include tasks they cannot access or instantiate a workflow in an unauthorized project.
- Repeated conversion of the same task selection requires explicit confirmation.

**Deferred:** Dependency graphs, role-based assignee placeholders, cross-organization templates, automatic synchronization after conversion, workflow versioning, and automation rules.

## 1.7 AI Assistance Service and Minimum Governance

**Deployment verification:** All development migrations, including the AI audit schema, were verified applied on 2026-07-22. The capability was enabled with audited overrides for the `grace-filled` pilot organization. Production remains default-off pending deployment and pilot acceptance.

**Implementation status (feature branch):** Complete on `feature/ai-assistance-service-governance`; enabled only for the selected development pilot and default-off elsewhere. The foundation includes centralized entitlement enforcement, an authenticated organization-scoped assistance endpoint, capability-specific text-generation and audio-transcription provider contracts, OpenAI, Hugging Face, and disabled-provider adapters, server-only credentials, configurable provider/model/timeout and cost settings, versioned prompt templates, credential redaction, transactionally reserved per-user and per-organization hourly limits, correlation IDs, review-gated draft contracts, content-free request audits with estimated text-generation cost, basic untrusted-content instructions, provider-side storage disabled for generated drafts, and audited pilot enablement through `tracker-admin`. Existing queued note-audio transcription retains its queue, retries, and note-status behavior while using the same entitlement, usage-limit, correlation, and audit governance. Permission-aware server context assemblers are implemented for task-thread summaries and project-update drafts; arbitrary client-supplied workspace context is not accepted.

Build this shared substrate before releasing any AI-assisted feature. Individual AI features must use it rather than integrating directly with model providers.

**Ticketing rule:** Create this as a standalone infrastructure epic with its own owner, implementation tickets, security review, tests, and release gate. Do not bundle it into AI Text Assistance or classify it as optional polish. The infrastructure may remain disabled in production until an AI feature is ready, but its acceptance criteria must pass before that feature enters release validation.

**MVP capabilities:**

- Provider abstraction with centrally selected models and timeouts.
- Organization-level AI on/off setting with explicit administrator control.
- Per-organization and per-user rate limits and usage counters.
- Central authorization and context assembly before data reaches a provider.
- Basic redaction for credentials, tokens, and known sensitive-field types.
- Prompt templates with versions and feature identifiers.
- AI-call audit metadata: organization, user, feature, model, timestamp, input/output size, latency, status, and estimated cost. Do not log raw prompts or outputs by default.
- Consistent review-gated response contracts for drafts, citations, confidence, and errors.
- Basic defenses that treat retrieved messages, documents, and submissions as untrusted data rather than executable instructions.

**Acceptance criteria:**

- Disabling AI for an organization blocks all AI endpoints server-side.
- Usage limits cannot be bypassed by calling an endpoint directly.
- Provider credentials never reach the browser or audit payload.
- Every AI request has a correlation identifier and observable success/failure result.
- A provider can be replaced without changing individual feature contracts.
- Raw workspace content is retained only according to the configured provider and organization policy.

**Later:** Phase 7 matures this foundation with permission-aware indexing, evaluation datasets, advanced injection defenses, retention controls, and workspace-wide retrieval.

## 1.8 AI Text Assistance (Optional)

**Implementation status (feature branch):** Validation for the pilot. Rewrite Selected Text, Summarize Task Thread, Generate Checklist, and Draft Project Update are implemented as explicit, entitlement-gated actions through the shared AI Assistance Service, with editable review drafts and correlation IDs. Rewrite and checklist generation send only the user's current selection; accept replaces only the unchanged selection as sanitized rich text and never saves the containing record automatically. Checklist drafts are bounded to ten actionable bullet items and cannot invent requirements. Task-thread summarization accepts only a task identifier from the client, verifies project view permission server-side, loads up to the 100 most recent non-deleted comments, and lets the user edit, discard, or place the summary into the unsaved comment composer. Project-update drafting verifies contributor access, assembles bounded project/task/recent-update context server-side, returns validated structured health, accomplishment, blocker, and next-step fields, and only populates the unsaved update form after explicit review and acceptance. No AI action saves or publishes automatically. The capability was enabled for the `grace-filled` development pilot organization on 2026-07-22.

**MVP experience:** Explicit buttons summarize a task thread, rewrite selected text, generate a checklist, or draft a project update.

**Safety boundary:** The result appears as an editable draft and is never saved automatically. It must run through the shared AI Assistance Service in 1.7.

**Acceptance criteria:** Input is permission-scoped, secrets are excluded where possible, usage is rate-limited, and users can accept, edit, or discard the output.

**Release gate:** Section 1.7 must be complete and enabled for the pilot organization. If the feature processes decisions, audio, transcripts, or other lifecycle-controlled data, the applicable work in 1.9 must also be complete.

## 1.9 Baseline Data Lifecycle Controls

**Deployment verification:** All development migrations, including the lifecycle-event table and note-consent columns, were verified applied on 2026-07-22. Production deployment remains pending.

**Implementation status (feature branch):** Complete; pending deployment migration. Decisions and note audio now conform to a shared content-free lifecycle-event contract. Permission-checked JSON export, hard deletion, and access-history endpoints cover both record types; decision deletion also removes links and content-bearing history snapshots, while audio deletion removes object-storage content and clears transcripts, metadata, and consent fields. Recording notice and affirmative consent are enforced in both the recorder UI and API before upload or transcription. Default retention, backup expiry, legal-hold limitations, derived-data cleanup, and provider-artifact behavior are documented in `DATA_LIFECYCLE_POLICY.md`.

**Problem:** Decisions, audio, transcripts, and future AI inputs may contain sensitive data that users need to export or remove before enterprise retention controls exist.

**MVP experience:** Define retention ownership and deletion behavior for decisions immediately. Before audio or transcripts are stored, add explicit recording consent, deletion, export, and access-history behavior for those record types.

**Ticketing rule:** Create this as a standalone data-governance epic rather than embedding it in the decision register, transcription, or AI feature tickets. Split implementation by record type, but require every type to conform to the shared lifecycle contract. A user-facing feature that introduces a new sensitive record type cannot ship until its lifecycle ticket passes acceptance.

**Implementation work:**

- Classify each sensitive record type and document its default retention behavior.
- Define hard delete, soft delete, legal/audit preservation, backup expiry, and derived-data cleanup behavior.
- Add authorized export and deletion endpoints with activity/audit records.
- Ensure deleted source content is removed from search and AI indexes.
- Display consent and participant notice before meeting recording begins.

**Acceptance criteria:** Deletion removes or schedules removal of source files, transcripts, search/index copies, and provider-side artifacts where supported; exports are permission-checked; and preserved audit records do not contain the deleted content.

## Phase 1 Ticket-Cutting Order

Create Phase 1 work in this order so infrastructure cannot disappear inside feature scope:

1. Foundation Gate tickets required by the selected Phase 1 release.
2. `1.7 AI Assistance Service and Minimum Governance` as a standalone infrastructure epic.
3. `1.9 Baseline Data Lifecycle Controls` as a standalone data-governance epic, initially covering decisions and any currently stored audio.
4. Independent product epics for 1.1 through 1.6.
5. `1.8 AI Text Assistance` as its own optional user-facing epic.

Items 1.7 and 1.9 are prerequisites, not sub-tasks of 1.8. They may be developed before committing to an AI feature release, and their services must support future features without feature-specific coupling.

---

# Phase 2: Configurable Workflows and Intake

## Goal

Allow organizations to adapt Tailpoint to their processes without custom development.

## 2.1 Custom Fields

**Status:** Validation. CF-01 through CF-07 are implemented across the backend, workspace, and admin surfaces. The development migration is applied, the capability remains default-off, and organization activation is available through the audited admin pilot controls.

**MVP experience:** Project administrators create text, number, date, single-select, multi-select, checkbox, person, and URL fields. Fields may be required and can have default values.

**Implementation work:** Create field-definition and typed field-value models; validate values server-side; add field ordering, archiving, filtering, and API serialization; render reusable field editors in task forms and details.

**Acceptance criteria:** Archived fields retain historical values, required fields are enforced server-side, select options cannot silently invalidate old data, and field values participate in filters and exports.

## 2.2 Custom Workflows

**Status:** Validation. WF-01 through WF-07 are implemented across backend, workspace, and admin: default-off capability controls, versioned persistence and compatibility backfill, safe drafts and immutable publishing, centralized transition enforcement/history, current status-mutation integration, project workflow editing, authoritative permitted transitions, task history, and audited pilot controls. Development migration/backfill and production builds are verified; organization activation remains an operator rollout decision.

**MVP experience:** Project administrators configure statuses, allowed transitions, required transition fields, and which roles may perform a transition.

**Implementation work:** Separate workflow definitions from task status values; create a centralized transition service; validate every transition through that service; record transition history; provide a safe editor with validation.

**Acceptance criteria:** Tasks cannot bypass rules through API calls, workflows cannot remove an in-use status without migration instructions, invalid workflows cannot be published, and existing projects receive a compatible default workflow.

## 2.3 Forms and Request Intake

**Status:** Development. RF-01 through RF-03 are implemented, covering the capability contract, versioned persistence, and project-scoped builder/preview/publish/archive APIs with immutable published versions and validated mappings. Normalized submission-to-task processing, public abuse/upload controls, workspace/public UI, and pilot validation remain.

**MVP experience:** Users build public or organization-only forms whose submissions create tasks. Forms support conditional fields, attachments, confirmation text, and destination project/status.

**Implementation work:** Add form definitions, versions, submission records, field validation, public rate limiting, spam protection, upload restrictions, and deterministic mapping into task/custom fields.

**Acceptance criteria:** Published form versions remain stable, submissions retain original answers, permissions apply to private forms, abuse controls protect public endpoints, and users can trace a task back to its submission.

## 2.4 Reusable Templates

**Status:** Complete. Versioned organization templates support task, checklist, and project snapshots, visible target-workflow compatibility previews, target-project-validated status remapping, relative dates, immutable new-version editing, transactional instantiation, workspace library controls, focused contract tests, and default-off pilot enablement.

**MVP experience:** Save and instantiate task, checklist, and project templates containing relative dates, default assignees/roles, statuses, and selected documents.

**Implementation work:** Store versioned snapshots, validate compatibility with the target project workflow, preview all objects before creation, and create them transactionally.

**Acceptance criteria:** Relative dates resolve from a chosen start date, missing users or statuses require remapping, and changing a template never mutates previously created projects.

## 2.5 Milestones

**Status:** Validation. MS-01 through MS-07 are implemented across backend, workspace, and admin: default-off capability controls, tenant-safe milestone/task-link persistence, scoped CRUD, deterministic progress and guarded completion, Structured Project Update references, project activity snapshots, milestone timeline and management UX, focused boundary/scale tests, production builds, and audited pilot controls. Development migration and runtime initialization are verified; organization activation remains an operator rollout decision.

**MVP experience:** A milestone has an owner, target date, description, completion criteria, health, and related tasks. Progress is calculated from eligible task completion.

**Implementation work:** Add milestone CRUD, task relationships, calculation rules, project timeline display, and milestone activity events.

**Acceptance criteria:** Progress rules are documented and consistent, manual completion requires a reason when tasks remain open, and milestone permissions follow project permissions.

## 2.6 Basic Approvals

**Status:** Complete. Tenant-safe polymorphic requests support tasks, documents, and milestones; named project-peer selection, immutable responses and snapshots, unanimous approval/any-rejection resolution, configurable due dates and required rejection comments, subject-change invalidation, due reminders, audit records, named reviewer history, workspace controls, and default-off pilot enablement are implemented.

**MVP experience:** A user requests approval for a task, document, or milestone from one or more reviewers. Reviewers approve or reject with an optional comment.

**Implementation work:** Use a polymorphic approval request model, immutable responses, due dates, reminders, and subject snapshots for high-value approval types.

**Acceptance criteria:** Requesters cannot approve on behalf of reviewers, changes after approval visibly invalidate or preserve approval according to subject rules, rejection reasons can be required, and all actions are audited.

## 2.7 Universal Intake Expansion

**Status:** In progress. The default-off capability, normalized event/attempt schema, verified development migration, durable receive/process boundary, API/SDK adapter migration, and stable SDK retry idempotency are implemented. Operator retries, imports, webhooks, email, and workspace operations remain.

Add CSV and Excel imports, authenticated inbound webhooks, email-to-task, and improved SDK/API intake. Every channel must use a common normalized intake event, idempotency key, validation result, and task-creation pipeline.

### Email-to-Task

**Problem:** Work requests received by email remain outside the project system.

**MVP experience:** Each project may have a generated inbound address. An email creates a task using the subject, body, sender, and supported attachments.

**Implementation work:** Add an inbound email webhook with signature verification; map recipient tokens to a project and default status; sanitize HTML; apply spam protection; scan/validate attachments and enforce size limits; and store the provider message identifier for deduplication.

**Acceptance criteria:** Retried provider webhooks do not duplicate tasks, unknown or disabled addresses are rejected safely, sender information does not grant workspace access, malicious content is quarantined, and failed imports are observable and retryable.

## 2.8 AI Intake Assistance (Optional)

Suggest cleaned titles, categories, priority, duplicates, assignee, or destination project. Suggestions must display reasons and confidence where useful, and require confirmation before routing or merging.

---

# Phase 3: Automation and Execution Control

## Goal

Make repeated processes reliable, observable, and enforceable.

**Phase completion audit (2026-09-16):** The non-AI Phase 3 scope in sections 3.1 through 3.6 is implementation-complete and enabled for production rollout. All Phase 3 migrations are applied without observed regressions, the production task-status mutation timeout has been resolved, and the capabilities have been enabled. Production Redis/queue behavior still needs an operational verification window, but that follow-up does not block starting Phase 4. The former optional AI items 3.7 and 3.8 are deferred to the later AI backlog and are not part of the Phase 3 completion gate.

## 3.1 Rule-Based Automation Builder

**Status:** Released. The default-off capability, reversible persistence, builder/publish API, durable trigger capture, execution engine, dry-run API, run observability, controlled retry, and workspace history are implemented. Database-backed matching and claims provide concurrency-safe runs, typed conditions, bounded operational retries, stable idempotent actions, forward-only activation, current authorization/resource checks, and causation-based loop protection. Actions cover assignment, standard/Custom Field updates, workflow transitions, watchers, notifications, and task-template creation.

**MVP experience:** Administrators build rules with one trigger, multiple conditions, and multiple actions. Initial triggers include task created, field changed, status changed, deadline reached, and form submitted. Actions include assign, update field/status, add watcher, notify, and create task from template.

**Implementation work:** Define versioned trigger/condition/action contracts; enqueue executions; add idempotency and loop protection; keep execution logs; support dry-run testing; and introduce an organization-scoped synthetic `Tailpoint Automation` actor.

The synthetic actor is used for attribution, not unrestricted authorization. At execution time the engine must verify that the rule is active, its organization subscription still permits the feature, its referenced project/resources still exist, and the rule's stored authorization policy permits each action. Audit entries record both the synthetic actor and the human administrator who created or last materially changed the rule.

**Acceptance criteria:** Rules do not recursively trigger forever, every run explains matched conditions and action results, failures can retry safely, disabling a rule stops new runs, users can test rules against sample data, and actions remain attributable even after the original rule creator leaves the organization.

## 3.2 Advanced Recurring Work

**Status:** Released.

Extend recurrence to projects, holiday calendars, assignee rotation, reusable checklists, and exceptions. Preserve an occurrence history and make schedule changes effective from a selected date.

## 3.3 Task Dependencies

**Status:** Released.

**MVP experience:** Tasks can block other tasks. Users receive warnings when starting or completing work in conflict with dependencies.

**Implementation work:** Add dependency edges, cycle detection, permission checks across projects, activity records, and optional date propagation previews.

**Acceptance criteria:** Circular dependencies are rejected, inaccessible dependency details are not leaked, deletion leaves a clear history, and bulk date changes require preview and confirmation.

## 3.4 Advanced Approvals

**Status:** Released.

Add sequential stages, required/optional reviewers, unanimous or threshold policies, delegation, reminders, and escalation. Approval policy must be snapshotted when a request begins.

## 3.5 Audit Trail

**Status:** Released.

Record actor, organization, action, subject, timestamp, request correlation identifier, and safe before/after metadata for important events. Provide filtering, export, and retention controls. Secrets, tokens, and sensitive file contents must never enter the audit payload.

## 3.6 Reliable Integration Delivery

**Status:** Released.

Provide signed outbound webhooks, retry schedules, delivery logs, secret rotation, replay controls, and a dead-letter state. Replays must preserve the original event identifier.

---

# Phase 4: Clients and Commercial Work

## Goal

Make Tailpoint suitable for agencies, consultants, and service organizations working with external stakeholders.

**Status (2026-09-16):** Active next phase. Begin with the shared client-visibility and authorization model required by the Guest and Client Portal, Client Approvals, and all later client-facing records.

## 4.1 Guest and Client Portal

**MVP experience:** External users access selected projects, milestones, tasks, files, updates, and forms through an explicit client role.

**Implementation work:** Introduce resource-level visibility (`internal` or `client-visible`), invitations, portal navigation, restricted search, and separate notification preferences.

**Acceptance criteria:** Internal records never appear in portal queries, counts, search results, notifications, or exports; revocation takes effect immediately; project admins can preview the portal as a client.

## 4.2 Client Approvals

Expose selected approval requests in the portal with formal sign-off, rejection comments, deadlines, reminders, and downloadable history.

## 4.3 Internal vs Client-Visible Content

Apply visibility consistently to tasks, comments, documents, files, updates, milestones, and notifications. Centralize the policy instead of implementing one-off filters in controllers.

## 4.4 Time Tracking and Timesheets

Support running timers, manual entries, billable state, descriptions, task/project association, weekly submission, and manager approval. Prevent overlapping timers unless the organization enables them.

## 4.5 Budgets and Profitability

Add project budget type, cost and billing rates, approved time, expenses where required, threshold alerts, and margin calculations. Limit financial data to authorized roles and define how historical rates are preserved.

## 4.6 Branded and Scheduled Reports

Provide client-safe report templates, logos/colors, selected reporting periods, PDF/CSV exports, and scheduled email delivery. Generated reports should be immutable snapshots with delivery history.

## 4.7 AI Client Update Composer (Optional)

Draft an external update only from records explicitly marked client-visible. A human reviews the exact recipients and content before sending.

## 4.8 Meeting Transcription and Meeting-to-Work (Optional AI)

Upload or record audio, transcribe with timestamps, and suggest decisions/tasks/owners. Retain consent and deletion controls. Suggested records remain drafts and link to transcript timestamps when approved.

---

# Phase 5: Planning, Capacity, and Reporting

## Goal

Support larger projects and organization-wide planning using structured data created in earlier phases.

## 5.1 Timeline and Gantt

Render tasks and milestones over time with dependency lines and drag-to-reschedule. Changes must preview dependency impact, respect working calendars, and be committed through normal task APIs.

## 5.2 Capacity and Workload

Store user/team weekly capacity, time off, and task estimates. Display allocation by week and warn about over-allocation. Clearly label unestimated work instead of treating it as zero.

## 5.3 Baselines and Variance

Allow authorized users to snapshot an approved schedule and scope. Compare baseline dates, estimates, milestones, and task set with current values. Require a change reason for formal re-baselining.

## 5.4 Custom Reports and Dashboards

Create a reporting query model with dimensions, metrics, filters, grouping, saved definitions, access rules, export, and scheduling. Start with completion rate, overdue rate, cycle time, lead time, workload, and estimate variance.

## 5.5 Goals and OKRs

Add objectives, measurable key results, owners, periods, confidence/status history, and project relationships. Progress can be manually entered or calculated from supported metrics.

## 5.6 Portfolio Management

Provide authorized leaders a cross-project timeline, milestone calendar, budget summary, capacity conflicts, dependencies, and explicitly calculated project-health rules.

## 5.7 Outcome-Based Planning (Optional AI)

Generate a proposed plan from an outcome and constraints. The result is an editable preview of milestones, tasks, estimates, roles, and dependencies. Nothing is created until confirmed.

## 5.8 Project Copilot v1 (Optional AI)

Answer questions about a single project using tasks, decisions, updates, notes, and documents. Every factual answer must link to its source, enforce current permissions, and state when evidence is insufficient.

---

# Phase 6: Integrations and Software Delivery

## Goal

Use the existing ingestion SDK to create a differentiated workflow for engineering and operations teams.

## 6.1 GitHub and GitLab

**Status (2026-09-19):** Phase 6A GitHub MVP is in Validation alongside Phase 3 production observation. The one-way, project-scoped vertical slice is implemented across backend, workspace, and admin: default-off entitlement controls, multiple repositories per project, encrypted/rotatable webhook secrets, rate-limited raw-body signature verification, fast `202` acknowledgement and queued processing, stable-ID rename reconciliation, delivery idempotency and retry recovery, provider-timestamp ordering, bounded event normalization, project-safe automatic and manual task linking, visible provenance, deterministic direct-over-inherited precedence, durable unlink/move overrides, safe pull-request inheritance, Owner-visible unresolved-reference diagnostics, archived history, silence/failure health and Owner alerts, task Development links, aggregate support health, activity/audit lifecycle capture, a signed five-event pilot harness, and reversible migrations. Backend and workspace production builds, focused tests, type checks, OpenAPI checks, and the local hardening migration pass. A live GitHub repository pilot remains before release. GitLab, OAuth/GitHub App installation, repository discovery, and two-way synchronization remain deferred. See the [Phase 6A implementation plan](../phases/PHASE_6A_GITHUB_IMPLEMENTATION_PLAN.md).

Link commits, pull requests, issues, deployments, and releases to Tailpoint tasks. Begin with one-way links and webhook ingestion before attempting two-way synchronization. Store provider delivery identifiers for idempotency.

## 6.2 Slack and Microsoft Teams

Allow users to create tasks from messages and receive selected notifications. Channel installations and user identity mapping must be organization-scoped. Avoid posting private project data to unauthorized channels.

## 6.3 Calendar Integration

Synchronize selected deadlines and milestones with Google or Microsoft calendars. Define the source of truth, handle deleted events, store provider cursors, and make synchronization failures visible.

## 6.4 Incident Workflows

Add incident severity, commander, responders, status, timeline events, checklists, linked ingestion events, resolution, and postmortem. Incident events should remain immutable except for explicit corrections.

## 6.5 Release Management

Model releases, environments, included tasks, deployment attempts, approvals, rollback records, and release notes. Provider deployments can update release state through verified webhooks.

## 6.6 Connector-Friendly API

Publish stable inbound/outbound webhook contracts and recipes for Zapier and Make. Include API versioning, scopes, test events, rate limits, and delivery observability.

## 6.7 AI Incident Assistance (Optional)

Cluster similar ingestion events, relate them to recent deployments, summarize an incident timeline, suggest investigation steps, and draft a postmortem. All conclusions should link to events and be presented as suggestions rather than facts when uncertain.

---

# Phase 7: Workspace Intelligence

## Goal

Introduce workspace-wide AI only after permissions, search, activity history, and structured workflows are dependable.

## Required Technical Foundation

This matures the minimum AI Assistance Service introduced in Phase 1; it is not the first point at which AI governance is implemented.

- Permission-aware unified indexing for tasks, messages, notes, documents, decisions, transcripts, and project updates.
- Document text extraction and chunking.
- Source identifiers and deep links for citations.
- Index updates for creates, changes, deletions, permission changes, and membership revocation.
- Mature organization retention, provider selection, usage budgets, deletion propagation, and administrator reporting controls.
- Prompt-injection defenses for untrusted document and message content.
- Evaluation datasets for permission leakage, citation correctness, and answer quality.

## 7.1 Project Copilot v2

Expand the copilot across permitted projects. Support follow-up questions, source citations, and explicit project scope selection. Never use inaccessible record titles as hints.

## 7.2 AI Command Center

Present prioritized issues such as stalled work, overdue approvals, unanswered questions, deadline risk, and workload imbalance. Each card must explain its evidence and offer a safe action or direct link.

## 7.3 Suggested Project Memory

Detect possible decisions in messages, notes, documents, and transcripts. Ask a user to confirm the structured decision before adding it to the register.

## 7.4 Smart Inbox

Group duplicate notifications, summarize threads, rank urgency, draft replies, and suggest task creation. Users retain deterministic filters and chronological view when AI ranking is disabled.

## 7.5 Predictive Project Health

Use deadline drift, blocked duration, scope growth, reopen rate, and workload concentration. Begin with transparent deterministic scoring; introduce learned predictions only after sufficient labeled history and evaluation.

## 7.6 Dependency Impact Assistance

Use the dependency graph to explain what a delay may affect and propose a revised sequence. Display assumptions and require confirmation for schedule changes.

---

# Phase 8: Ecosystem, Offline, and Enterprise

## Goal

Expand distribution and support demanding environments after the core product is mature.

## 8.1 Template Marketplace

Support private organization libraries first, then public publishing. Add template versioning, categories, previews, compatibility checks, moderation, ratings, and safe import behavior.

## 8.2 Offline-First PWA/Mobile

Cache selected projects and personal work, queue mutations and uploads, display sync state, and define conflict resolution per entity. Start with task viewing, status updates, comments, and note capture.

## 8.3 Collaborative Document Review

Add inline comments anchored to document versions, suggested edits, review requests, comparison, approval, and finalization locks. Anchors must degrade gracefully when content changes.

## 8.4 Advanced Dependency Graph

Visualize projects, milestones, tasks, decisions, documents, people, releases, and incidents. Apply permissions before graph traversal and aggregate hidden relationships without leaking details.

## 8.5 Enterprise Controls

Add SAML/SSO, SCIM, retention policies, organization exports, advanced audit controls, session management, and optionally IP restrictions. Treat these as security projects with threat modeling and dedicated testing.

---

# Deferred AI Backlog

These items were formerly Phase 3.7 and 3.8. They are deliberately deferred until after the current Phase 4 client and commercial-work priorities and do not reopen the completed Phase 3 delivery gate.

## AI-D1 Conversational Automation

**Status:** Deferred.

Convert a plain-language instruction into a draft automation definition. Show the parsed trigger, conditions, and actions for review; validate it through the same rules as the visual builder.

## AI-D2 Semantic Duplicate Suggestions

**Status:** Deferred.

Suggest similar tasks, requests, and incidents. Users may ignore, link, or merge them. Preserve attribution and activity history during a merge.

---

# Parallel Track: Communication Channels

WhatsApp automation and LiveKit-powered calls are deliberately managed as a parallel communications track rather than omitted. They have different delivery, consent, moderation, reliability, and infrastructure concerns from the core project-management phases. Their releases should still use the shared authorization, entitlement, notification, audit, storage, and retention services defined in this roadmap.

## WhatsApp Bot

Define supported inbound commands and conversational flows, organization/user identity linking, webhook verification, message-template approval, opt-in/opt-out, rate limits, attachment handling, delivery receipts, failure recovery, and which project actions are safe from chat. Mutating actions must require unambiguous confirmation and normal permission checks.

## LiveKit Calls

Define one-to-one and project/group call scope, room authorization, short-lived participant tokens, ringing and missed-call behavior, presence, moderation, call history, recording consent, recording/transcript retention, and reconnect behavior. Recording and transcription must use the baseline data-lifecycle controls before release.

## Relationship to Core Phases

- Basic chat reliability and participant permissions belong in the Foundation Gate.
- WhatsApp-created tasks should use the normalized intake pipeline from Phase 2.
- Communication-triggered actions should use the automation engine from Phase 3.
- Calls for guests must follow the client visibility model from Phase 4.
- Transcription and meeting-to-work use the shared AI Assistance Service and data-lifecycle controls.
- Slack and Teams remain in Phase 6 because they are external work integrations, not the app's primary realtime communication layer.

---

# Packaging, Entitlements, and Monetization

Feature flags control rollout safety; subscription entitlements control commercial availability; organization settings control optional use; user permissions control individual actions. These are separate concerns and must be evaluated centrally.

## Entitlement Service

The backend owns entitlement resolution. `track-a-project` consumes the effective result for navigation and feature UX, while `tracker-admin` manages permitted rollout overrides and explains why a capability is enabled or disabled. Neither frontend may independently reconstruct plan or flag logic.

For the first vertical slice, `tracker-admin` must provide an organization entitlement panel showing the stable capability key, effective state, subscription requirement, organization override, and resolution reason. Super-admin override mutations must be validated against the capability catalog and audited with actor, organization, previous value, new value, and timestamp.

Create one backend service that resolves a feature from organization plan, add-ons, trial state, limits, feature-flag rollout, and organization setting. The frontend consumes a capability response for navigation and upgrade prompts, but backend endpoints remain authoritative.

## Initial Packaging Candidates

- Core: personal productivity, standard projects/tasks, updates, basic recurrence, and standard collaboration.
- Workflow: custom fields, workflows, forms, templates, milestones, and basic automation quotas.
- Client/Agency: portal, approvals, time tracking, budgets, profitability, and branded reports.
- Portfolio: workload, baselines, advanced reporting, goals, and portfolio management.
- AI add-on or included allowance: AI assistance features with organization usage limits and transparent overage/disable behavior.

Final names and limits require pricing validation. Store entitlements as stable capability keys so pricing changes do not require rewriting feature code.

## Acceptance Criteria

- Downgrades preserve customer data while disabling restricted creation/actions safely.
- Trials and grace periods have explicit start/end behavior.
- Usage-based limits are consistent across UI, API, jobs, and integrations.
- Administrators can see current entitlement, consumption, and upgrade requirement.
- Billing webhooks are idempotent and entitlement changes are audited.

---

# Recommended Build Order Within the First Three Phases

The following order minimizes rework:

1. Foundation gate and real global search.
2. Personal productivity hub.
3. Structured project updates.
4. Decision register.
5. Task discussion improvements.
6. Basic recurring tasks.
7. Work-to-workflow conversion, including whiteboard objects to work and project tasks to reusable workflows.
8. Custom fields.
9. Custom workflow transition service.
10. Forms using custom fields.
11. Templates.
12. Milestones.
13. Basic approvals.
14. Normalized intake pipeline, then email-to-task.
15. Automation event contracts and job execution.
16. Automation builder.
17. Dependencies and advanced recurrence.
18. Advanced approvals and audit trail.
19. Reliable outbound integration delivery.

# Feature Ticket Template

Use this template when moving a roadmap item into implementation:

```md
## Feature name

### Outcome
What user or business outcome should improve?

### In scope
- Exact MVP behavior

### Out of scope
- Explicitly deferred behavior

### Roles and permissions
- Who can view, create, edit, delete, approve, export, or administer it?

### Data model
- New entities, fields, relationships, indexes, retention, and migration behavior

### API contract
- Endpoints/events, request and response shapes, validation, pagination, and errors

### UI states
- Loading, empty, success, validation, permission denied, error, retry, and mobile

### Realtime and notifications
- Events, recipients, deduplication, preferences, and unread behavior

### Activity and audit
- Events that must be recorded and sensitive values that must be excluded

### Failure and retry behavior
- Idempotency, partial failure, rollback, background jobs, and manual recovery

### Analytics
- Adoption and outcome events to measure

### Acceptance criteria
- Testable product behavior

### Test plan
- Unit, integration, permission, migration, load, and end-to-end tests

### Rollout
- Feature flag, pilot organizations, migration/backfill, monitoring, and rollback
```

# Definition of Done

A roadmap feature is complete only when:

- Product scope and deferred behavior are documented.
- Permissions are tested on both backend and frontend surfaces.
- Migrations and backward compatibility are verified.
- APIs validate input and return consistent errors.
- Loading, empty, error, retry, and mobile states exist.
- Important changes appear in activity/audit history.
- Notifications are deduplicated and preference-aware.
- Background work is idempotent and observable.
- Relevant unit, integration, and end-to-end tests pass.
- Usage and failure metrics are available.
- Documentation and feature flags are updated.
- The feature has been manually tested with owner, member, guest, and unauthorized roles where applicable.

# Initial Product Recommendation

The most cohesive first marketable bundle is:

1. Personal productivity hub.
2. Custom fields and workflows.
3. Forms and normalized intake.
4. Templates and recurring work.
5. Milestones and approvals.
6. Rule-based automation.
7. Client portal and client-visible content.
8. Structured updates and reports.

This supports a clear promise without depending on AI: organizations can design a workflow, receive work, execute it, collaborate with clients, approve outcomes, and report results in one system.

This bundle is a go-to-market cut across several phases, not an instruction to complete Phases 1–3 and then append Phase 4/5 work. Once a target market is selected, create a release plan containing only the prerequisite slices required for that bundle. For an agency release, client visibility and approvals may be prioritized ahead of dependencies; for a software-delivery release, dependencies, audit, ingestion, and incidents should come first.
