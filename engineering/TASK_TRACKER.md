# Task Tracker

This file groups current work into practical execution buckets so we can move through the list with less context switching.

## Quick Wins

These look like smaller, more contained tasks that should be easier to ship without major architecture changes.

| Task | Current Status | Why It Fits Here | Context Needed |
| --- | --- | --- | --- |
| add notifications for done status | To Do | Narrow behavior change around status transitions. | In-app and push only, to avoid overloading email. |
| Task Add Bubble | To Do | Can be a lightweight global quick-add flow if we reuse existing task creation. | Show only inside authenticated dashboard screens, anchored bottom-right on desktop and mobile, with quick actions for task, note, and project. |

### Notes For `Task Add Bubble`

Recommended simplified scope:

- Available only inside authenticated dashboard screens.
- Floating action button stays bottom-right on desktop and mobile.
- Button expands into quick actions for `Add Task`, `Add Note`, and `Add Project`.
- Use Framer interactions for the reveal and action transitions.
- On desktop, open a better-designed modal for the selected action.
- On mobile, open a bottom drawer for the selected action.
- For `Add Task`, user selects a project, enters title and description, and the task is created in that project's `default_ingestion_status_id`.
- If a project has no default ingestion status, block task submit with a clear setup message.


## Urgent Fixes

These need attention soon either because they are overdue, affect delivery confidence, or unlock smoother shipping for the rest of the app.

| Task | Current Status | Why It Fits Here | Context Needed |
| --- | --- | --- | --- |
| add github deployment ci/cd to project | Done | CI checks and gated deployment workflows now cover all three applications. | Frontend and admin deploy through Vercel; backend deploys through Render. |
| subscription integration | To Do | Marked high priority and likely tied to billing or access control. | Build ahead of paying users with Stripe. First sequence: plan selection and checkout, then failed-payment handling, then upgrades and downgrades. |
| integrate with whatsapp | Paused | Valid feature direction, but should wait until Twilio access is confirmed. | Planned shape is phone-number linking plus structured bot commands for tasks, notes, and projects. |

## Larger Feature Work

These look broader, more product-shaped, or likely to need design and implementation passes across multiple screens.

| Task | Current Status | Why It Fits Here | Context Needed |
| --- | --- | --- | --- |
| documents page needs to be revamped | To Do | Broad UX and layout work rather than a narrow fix. | Scope this as document CRUD first, with some UI cleanup if needed. Documents are org-scoped and should support both typed content and uploads. Rich text likely already exists and can be reused. |
| micro animations across app | To Do | Cross-cutting polish task that should follow clearer priority work. | Focus on a few key moments first rather than trying to animate the whole app in one pass. |
| notes audio logging | on Hold | Better handled as a staged feature than a one-pass build. | Start with v1 audio recording, upload, and playback only. Transcription can come later. |
| Admin Implementation | on Hold | Broad area with unclear boundaries and likely multiple subfeatures. | Sequence should be user management, org settings, billing controls, system health, analytics, then moderation. |
| Personal Productivity Hub | Validation | MVP implemented across backend, workspace, and admin; awaiting migration deployment and pilot validation. | Capability remains disabled by default and is enabled per pilot organization through `tracker-admin`. |
| Basic Recurring Tasks | Validation | MVP implemented across backend, task editor, and admin; awaiting migration deployment and pilot validation. | Capability remains disabled by default. Validate scheduler retries and timezone behavior during pilot. |
| Structured Project Updates | Done | MVP implemented across backend, workspace, and admin, including Project Member Roles. | Default-off capability is ready for deployment and pilot validation. |

## Parking Lot

These are intentionally not in the active grouped list right now.

| Task | Current Status | Reason |
| --- | --- | --- |
| Projects query reset | Removed from active list | Needs product clarification before it should compete with current priorities. |

## Suggested Build Sequence

1. [x] add notifications for done status 
2. [x] Task Add Bubble
3. [x] add github deployment ci/cd to project
4. [ ] subscription integration
5. [x] documents page needs to be revamped
6. [x] micro animations across app
7. [x] notes audio logging
8. [ ] Admin Implementation

## Implementation Checklists

### Personal Productivity Hub

- [x] Add permission-aware cross-project task query
- [x] Add My Tasks, Today, Upcoming, Overdue, and Waiting On views
- [x] Apply project, status, priority, assignee, due-date, and search filters
- [x] Reuse identical authorization and user filters for view counts
- [x] Add sorting and server pagination
- [x] Protect APIs with `personal_productivity_hub`
- [x] Add organization-scoped saved views with visibility and default selection
- [x] Add capability-gated workspace route and navigation
- [x] Add loading, empty, error, retry, filter, and pagination states
- [x] Add quick status updates with project-valid statuses
- [x] Add explicit pilot rollout controls and guidance in `tracker-admin`
- [x] Keep the capability disabled by default
- [ ] Run migration and validate with an approved pilot organization

### Basic Recurring Tasks

- [x] Register the `recurring_tasks` capability with default-off rollout
- [x] Add recurrence definitions and idempotent occurrence records
- [x] Support daily, weekly, monthly, and selected-weekday rules
- [x] Support completion-triggered and before-due generation
- [x] Enforce organization and project write authorization
- [x] Add scheduled generation with retry behavior
- [x] Link generated tasks to their preceding occurrence
- [x] Add pause, resume, and removal controls in the task editor
- [x] Create a task and optional recurrence atomically from project and quick-add forms
- [x] Show recurrence details in the read-only task view
- [x] Add this-occurrence versus this-and-future edit scope
- [x] Gate member UI and APIs through the recurring-task entitlement
- [x] Add pilot labeling and enablement through `tracker-admin`
- [ ] Run migration and validate scheduler behavior with a pilot organization

### Structured Project Updates

- [x] Register a default-off `structured_project_updates` capability
- [x] Add project-scoped drafts and immutable published snapshots
- [x] Add health, narrative sections, and optional reporting periods
- [x] Preserve published corrections as append-only version history
- [x] Add snapshot references for tasks, documents, and users
- [x] Enforce organization and project authorization across update operations
- [x] Queue preference-aware notifications for project members on publication
- [x] Allow permitted editors to delete drafts while protecting published snapshots
- [x] Paginate update feeds and correction history on the server
- [x] Add project-scoped task, document, and user reference controls
- [ ] Enable milestone references after the dedicated milestone model is implemented
- [x] Group corrections into one latest update thread with expandable immutable versions
- [x] Add publication preview, recipient guidance, loading, retry, and unsaved-change handling
- [x] Add server-side status, health, and current-author update filters
- [x] Add debounced, paginated project-record reference search
- [x] Add project update history, drafting, publishing, and correction UI
- [x] Add notification routing in the workspace
- [x] Add pilot enablement in `tracker-admin`
- [x] Run the Project Updates migrations in the local environment
- [ ] Validate with a pilot organization
- [x] Add explicit viewer, contributor, editor, and owner project roles before broad rollout

### Project Member Roles

- [x] Add viewer, contributor, editor, and owner roles to project memberships
- [x] Backfill existing peers and pending invitations as editors
- [x] Preserve the selected role when an invitation is accepted
- [x] Add invitation-time role selection for email, peer-list, and link invitations
- [x] Add Project Settings with member role management
- [x] Restrict role management to project owners and protect creator ownership
- [x] Audit project member role changes
- [x] Apply the role matrix to Project Updates
- [x] Apply the shared role policy consistently to project-scoped tasks, recurring tasks, statuses, comments, updates, invitations, ingestion, and settings mutations

### 1. add notifications for done status

- [x] Identify the task status transition path used when tasks move into or out of `isTerminal`
- [x] Confirm the backend event source that should trigger notifications
- [x] Add in-app notification creation for move to terminal and move from terminal states
- [x] Add push notification trigger for the same transitions
- [x] Prevent duplicate notifications on repeated saves with no real status change
- [x] Decide which users receive the notification: creator, assignees, watchers, or project peers
- [x] Add UI copy for both completion and reopen flows
- [x] Test status changes from board, list, and any other task editing surfaces

### 2. Task Add Bubble

- [x] Add a global floating action button shell inside authenticated dashboard layouts only
- [x] Position the button bottom-right for desktop and mobile
- [x] Build the quick-action reveal for `Add Task`, `Add Note`, and `Add Project`
- [x] Add Framer Motion interactions for open, close, stagger, and tap states
- [x] Create the desktop modal variant for quick-add actions
- [x] Create the mobile bottom-drawer variant for quick-add actions
- [x] Reuse existing task creation flow for `Add Task`
- [x] Add project selection, title, and description fields for quick-add task creation
- [x] Auto-use the selected project's `default_ingestion_status_id` when creating a task
- [x] Show a clear validation message when a selected project has no default ingestion status
- [x] Ship `Add Task` as the only fully wired v1 action
- [x] Wire `Add Note` into the quick-add flow
- [x] Wire `Add Project` into the quick-add flow
- [x] Test behavior across dashboard pages so the button feels global but not intrusive

### 3. add github deployment ci/cd to project

- [x] Inventory the deployable apps: frontend, backend, and admin
- [x] Confirm current hosting target for each app
- [x] Define branch strategy and deployment environments
- [x] Add lint and test steps for each app where available
- [x] Add separate GitHub Actions workflows or jobs for each app
- [x] Configure required secrets for each deployment target
- [x] Add preview or staging behavior if needed
- [x] Add failure visibility through GitHub checks and notification hooks if desired
- [x] Document deploy and rollback steps in a short runbook
- [x] Validate one end-to-end deployment path per app

## Open Questions

Please add short notes under any item that needs more explanation, especially:

- None right now. Add notes as priorities shift.

## Removed From Active List

| Task | Reason |
| --- | --- |
| fix mailing | Not a product bug right now; email is just still pointed at Mailtrap sandbox. |

## Paused Items

| Task | Reason To Pause |
| --- | --- |
| integrate with whatsapp | Wait until Twilio access is confirmed before planning the first implementation slice. |

Once those are clearer, we can break each one into implementation-ready subtasks.
## Decision Register

- [x] Register default-off `decision_register` capability and admin pilot controls
- [x] Add project-scoped decision, typed-link, and history schema migration
- [x] Enforce Viewer, Contributor, Editor, and Owner decision permissions
- [x] Preserve accepted/rejected/superseded history and transactional supersession
- [x] Add status, owner, and decision-date filtering
- [x] Add project-details Decision Register and history UI
- [x] Add project-validated task, message, note, document, and user link workflows
- [x] Add edit, administrative correction, proposal deletion, and supersession controls
- [x] Add server-side decision and history pagination
- [x] Apply and verify the migration, API route initialization, builds, and focused tests

## Task Discussion Improvements

- [x] Add task-scoped threaded comments and root-thread pagination
- [x] Add project-member mentions and deduplicated notifications
- [x] Add reactions, permalinks, resolution, soft deletion, and edit history
- [x] Add the task-detail discussion UI
- [x] Apply the migration and add focused verification

## Work-to-Workflow Conversion

- [x] Add conversion, workflow-template, and ordered workflow-step schema
- [x] Convert selected whiteboard objects into linked tasks or notes
- [x] Save selected project tasks as an ordered reusable workflow template
- [x] Preview and instantiate a workflow in an authorized project
- [x] Map status, retain-or-clear assignees, and calculate relative due dates
- [x] Add transactional validation and duplicate-conversion confirmation
- [x] Enforce Editor/Owner and source/destination project authorization
- [ ] Add UI, tests, migration, and end-to-end verification — UI, focused tests, compilation, and database migration verification are complete; authenticated browser smoke testing remains.
