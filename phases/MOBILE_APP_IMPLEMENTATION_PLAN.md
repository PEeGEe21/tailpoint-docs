# Tailpoint Mobile App — Implementation Plan and Checklist

**Status:** In progress — Phase 1 foundation  
**Document version:** 1.0  
**Last updated:** 2026-09-04  
**Target:** Production-ready iOS and Android companion app  
**Related:** [Mobile PRD](../product/mobile/MOBILE_APP_PRD.md) · [Mobile infrastructure](../product/mobile/MOBILE_APP_INFRASTRUCTURE.md)

## 1. Delivery objective

Deliver a secure, accessible Tailpoint mobile app that lets users authenticate, select a workspace, manage daily project/task work, understand dependencies, respond to approvals, and act on notifications from iOS and Android.

The mobile app is a focused companion to the web workspace. Configuration-heavy surfaces—including automation, workflow, integration, audit, billing, and schema builders—remain web-first during the initial release.

## 2. Target schedule

The working estimate assumes one focused mobile engineer, available backend support, and timely product/design review.

| Phase | Target duration | Cumulative target |
|---|---:|---:|
| 0. Product and delivery readiness | 2–3 days | Week 1 |
| 1. Repository and app foundation | 1 week | Week 1–2 |
| 2. Design system and navigation shell | 1 week | Week 2–3 |
| 3. API contracts and authentication | 1–2 weeks | Week 3–5 |
| 4. Home and projects | 1–2 weeks | Week 5–7 |
| 5. Tasks and collaboration | 2–3 weeks | Week 6–9 |
| 6. Dependencies and approvals | 1–2 weeks | Week 8–10 |
| 7. Notifications, deep links, and resilience | 1–2 weeks | Week 9–11 |
| 8. Quality, pilot, and store release | 2–3 weeks | Week 11–14 |

Expected milestones:

- Internal development build: **week 2–3**
- End-to-end authenticated alpha: **week 4–5**
- Feature-complete MVP: **week 8–10**
- External pilot: **week 10–12**
- Production store release: **week 12–14**

Phases may overlap only when their dependency gates are satisfied. Calendar estimates should be re-baselined after Phase 3 because authentication and API-contract gaps carry the highest early uncertainty.

## 3. Status legend

- `[ ]` Not started
- `[~]` In progress
- `[x]` Complete and verified
- `[!]` Blocked; include the reason next to the item

A checkbox is complete only after its acceptance evidence exists. Merging code without the required test, review, or build evidence does not complete the item.

## 4. Release scope

### Mobile v1

- Splash, welcome, sign in, account creation, invitation handling, and password recovery handoff
- Authenticated workspace selection and organization switching
- Resumable onboarding
- System, Light, and Dark appearance modes
- Phone bottom-tab navigation and adaptive tablet/foldable sidebar
- Home/My Work
- Project list, pinned/recent projects, overview, activity summary, and permitted project creation
- Task list, detail, quick create, full edit, transitions, comments, and attachments
- Task dependencies and truthful blocked-state behavior
- Approval Inbox, context, and decisions
- Notification Inbox, push registration, and authenticated deep links
- Search across permitted projects, tasks, and people
- Explicit offline/stale states and local drafts where safe
- Accessibility, telemetry, testing, app-store delivery, and operational runbooks

### Deferred beyond mobile v1

- Automation builder and detailed execution administration
- Workflow and Custom Field schema builders
- Integration/webhook configuration and delivery administration
- Audit export and retention controls
- Form and template builders
- Billing, entitlements, and platform administration
- Full offline mutation outbox until idempotency/conflict contracts are proven
- Rich document, whiteboard, video, and reporting authoring

## 5. Phase 0 — Product and delivery readiness

**Goal:** Remove decisions that would cause foundation rework.

### Checklist

- [~] Confirm public app name, subtitle, bundle identifiers, package names, and URL scheme. Tailpoint, `com.tailpoint.mobile`, and `tailpoint` are configured; store subtitle remains.
- [x] Confirm repository name and private remote location: `PEeGEe21/tailpoint-mobile`.
- [x] Confirm mobile v1 scope and explicitly record deferred features.
- [ ] Confirm whether self-serve organization creation ships in v1.
- [ ] Confirm whether basic project creation ships in v1.
- [ ] Select pilot organizations and identify pilot owners.
- [x] Set initial minimums to the Expo SDK 57 platform floor: iOS 16.4+ and Android 7+.
- [ ] Confirm App Store and Play Console account ownership and access.
- [ ] Confirm privacy policy, terms, support URL, account deletion path, and data-contact owner.
- [ ] Select crash reporting and product analytics providers after privacy review.
- [ ] Confirm notification categories and safe preview defaults.
- [x] Approve initial information architecture and primary navigation.
- [x] Approve the System/Light/Dark theme requirement.
- [x] Approve phone bottom tabs and adaptive large-screen sidebar behavior.
- [x] Confirm NativeWind/Tailwind CSS as the styling utility layer while preserving semantic design tokens.
- [ ] Create a design file/project for mobile screens and components.

### Exit gate

- [x] Product owner approves the Mobile PRD and this implementation plan.
- [ ] No unresolved decision blocks app identity, authentication, navigation, or store setup.

## 6. Phase 1 — Repository and app foundation

**Goal:** Produce reproducible signed development builds with a tested engineering baseline.

### Repository and tooling

- [~] Create the mobile Git repository with protected `main` and `dev` branches. GitHub remote exists; remote `dev` and branch protection remain.
- [x] Add ownership, pull-request, issue, and contribution templates.
- [x] Scaffold the current stable Expo Router TypeScript application (Expo SDK 57 / React Native 0.86).
- [x] Enable strict TypeScript and stable import aliases.
- [x] Configure ESLint, Prettier, and pre-commit checks.
- [~] Pin dependencies and commit the lockfile. Dependencies are pinned in the generated lockfile; foundation commit remains.
- [x] Add unit/component test setup and one passing smoke test.
- [x] Add environment validation that fails fast for missing or malformed public configuration.
- [x] Add CI for formatting, linting, typecheck, tests, and Expo configuration validation.

### Expo and delivery environments

- [ ] Create the EAS project and assign organizational ownership.
- [x] Configure development, preview, and production build profiles.
- [~] Configure app identities. Production uses `com.tailpoint.mobile`; non-production suffixes remain before preview distribution.
- [ ] Configure development, staging, and production API base URLs.
- [ ] Store EAS variables and credential files with appropriate visibility.
- [ ] Create iOS and Android development builds.
- [ ] Verify builds install and launch on physical devices.
- [ ] Document signing credential ownership and recovery.

### Foundation services

- [x] Add application error boundary and recovery screen.
- [x] Add connectivity/app-state providers.
- [x] Add query client with conservative defaults.
- [x] Add Zustand stores only for approved client state.
- [x] Add SecureStore adapter with namespaced keys and test doubles.
- [x] Add logging/telemetry interfaces with sensitive-value redaction.

### Exit gate

- [ ] A clean checkout can install, test, and build reproducibly.
- [ ] Signed development builds launch on physical iOS and Android devices.
- [ ] Development, preview, and production configuration cannot be confused silently.

## 7. Phase 2 — Design system and navigation shell

**Goal:** Establish the visual and interaction system before feature screens multiply.

### Brand and design tokens

- [x] Replace Expo artwork with Tailpoint assets: opaque 1024px store icon, adaptive/monochrome exports, splash artwork, and licensed Figtree variable font.
- [x] Configure NativeWind/Tailwind CSS and expose Tailpoint color, spacing, radius, and typography tokens.
- [~] Implement System, Light, and Dark appearance modes. Persistent selection and live application are implemented; physical-device visual validation remains.
- [x] Restore the selected appearance before protected UI renders.
- [~] Match native status bar, navigation bar, keyboard, and supported system UI to the theme. Status and navigation surfaces are wired; device validation remains.
- [x] Test core foreground/background contrast in both themes with automated WCAG ratio assertions.
- [x] Support dynamic type/font scaling and reduced motion. Native text scaling is preserved, Figtree uses a centralized scale, and a live reduced-motion hook is available.

### Component primitives

- [x] Build Button, IconButton, TextField, searchable Select, Checkbox, Radio, and Switch.
- [x] Build Card, ListItem, Badge, Avatar, Progress, Divider, and Skeleton.
- [x] Build Sheet, Modal, anchored Menu, Alert, Toast, and confirmation dialog.
- [x] Build loading, empty, offline, forbidden, not-found, and error states.
- [x] Define destructive, warning, blocked, success, and pending semantic color/status patterns.
- [x] Add accessibility labels, modal focus boundaries, minimum touch targets, and component screen-reader contract tests.

### Navigation

- [~] Implement public, onboarding, and protected route groups. Groups and placeholder screens exist; session guards arrive with Phase 3 authentication.
- [x] Implement phone tabs: Home, Projects, Inbox, and You.
- [x] Implement global quick-create action.
- [x] Implement adaptive sidebar navigation for supported tablets/foldables.
- [~] Add screen headers, back behavior, modal presentation, and safe-area handling. Shell behavior exists; physical-device validation remains.
- [x] Add placeholder route handling for deep links.

### Exit gate

- [x] Component gallery demonstrates implemented primitives, searchable Select, and anchored Menu in Light and Dark modes.
- [ ] Phone and large-screen navigation pass keyboard, back-button, rotation, and accessibility checks.
- [ ] Theme changes require no restart and preserve navigation/form state.

## 8. Phase 3 — API contracts and authentication

**Goal:** Establish secure identity, tenant isolation, and generated backend communication.

### Backend API contracts

- [ ] Audit Swagger/OpenAPI coverage for authentication, organizations, projects, tasks, dependencies, approvals, notifications, search, and attachments.
- [ ] Add explicit request/response DTO documentation where schemas are missing or ambiguous.
- [ ] Add a reproducible OpenAPI export in backend CI.
- [ ] Add breaking-contract detection against the last mobile-compatible contract.
- [ ] Generate mobile TypeScript types/client using `openapi-typescript` and `openapi-fetch`.
- [ ] Add generated-client drift validation in mobile CI.
- [ ] Normalize the backend error envelope in one mobile API layer.
- [ ] Confirm pagination, dates, IDs, nullable fields, and enum compatibility rules.

### Backend authentication readiness

- [ ] Add mobile-safe `POST /api/auth/refresh` using a request body.
- [ ] Rotate refresh tokens and atomically revoke replaced tokens.
- [ ] Define refresh reuse, expiration, revocation, and logout behavior.
- [ ] Keep/deprecate the existing query-based refresh route without breaking web clients.
- [ ] Add rate-limit and safe-error tests for login, refresh, password recovery, and invitations.
- [ ] Confirm organization-switch token/session semantics.

### Mobile authentication

- [ ] Implement the session bootstrap state machine.
- [ ] Keep access tokens in memory only.
- [ ] Store refresh tokens only in SecureStore.
- [ ] Implement single-flight refresh and retry waiting requests once.
- [ ] Distinguish invalid credentials from offline/timeouts.
- [ ] Clear tokens, queries, drafts, and user state on sign out.
- [ ] Implement protected-route guards.
- [ ] Preserve pending invitation/deep-link intent through authentication.

### Authentication screens

- [ ] Splash/bootstrap without artificial delay.
- [ ] Welcome/get started.
- [ ] Sign in with keyboard, loading, rate-limit, and safe error states.
- [ ] Create account.
- [ ] Forgot/reset-password handoff.
- [ ] Invitation acceptance.
- [ ] Dedicated authenticated Choose workspace screen.
- [ ] Create organization flow if included in v1.
- [ ] Organization switching with cache isolation.
- [ ] Resumable onboarding and notification pre-prompt.

### Exit gate

- [ ] Login, refresh, restore, logout, invitation, and organization switching pass backend integration tests.
- [ ] Simultaneous expiry cannot create refresh races or loops.
- [ ] Organization switching cannot display records from the previous organization.
- [ ] One authenticated end-to-end smoke test passes on physical iOS and Android builds.

## 9. Phase 4 — Home and projects

**Goal:** Give users a useful, navigable view of their work.

### Home / My Work

- [ ] Implement greeting and active workspace control.
- [ ] Implement overdue, due-soon, blocked, and approval attention summaries.
- [ ] Implement My Tasks sections: Today, Upcoming, and Later.
- [ ] Implement pinned/recent projects.
- [ ] Add pull-to-refresh, pagination, and refresh-on-focus rules.
- [ ] Add empty, partial, stale, offline, forbidden, and error states.
- [ ] Measure Home screen-ready time without logging content.

### Projects

- [ ] Implement searchable project list.
- [ ] Implement pinned/recent/all groupings.
- [ ] Implement project cards with accessible status/progress summaries.
- [ ] Implement project overview.
- [ ] Implement compact Overview, Tasks, Activity, and More navigation.
- [ ] Hide unavailable modules based on capabilities and permissions.
- [ ] Implement basic project creation if included in v1.
- [ ] Verify archived and role-restricted project behavior.

### Exit gate

- [ ] A user can reach all permitted projects from Home or Projects.
- [ ] Lists remain responsive under production-scale test fixtures.
- [ ] Cached and fresh state are visibly distinguishable where necessary.

## 10. Phase 5 — Tasks and collaboration

**Goal:** Complete the core daily mobile work loop.

### Task discovery and detail

- [ ] Implement paginated task list with project and My Work contexts.
- [ ] Add assignee, status, priority, deadline, blocked, and mine filters.
- [ ] Persist non-sensitive recent filters per project.
- [ ] Implement task detail with identity/current state first.
- [ ] Render standard fields, supported Custom Fields, description, assignees, deadline, and priority.
- [ ] Render comments, attachments, dependencies, and activity progressively.
- [ ] Display only backend-authorized transitions and actions.

### Task mutations

- [ ] Implement global quick task creation.
- [ ] Implement full task create/edit form from current project schema.
- [ ] Implement assignment and workflow transitions.
- [ ] Implement comments with retry-safe failure behavior.
- [ ] Implement task watching if exposed for v1.
- [ ] Add local validation while preserving server authority.
- [ ] Add idempotency keys to retried create/comment operations once supported.
- [ ] Implement explicit offline drafts without claiming server success.
- [ ] Define conflict UI for stale task updates.

### Attachments

- [ ] Implement contextual document/photo/camera permissions.
- [ ] Validate file size/type before upload.
- [ ] Add image compression and metadata handling where appropriate.
- [ ] Implement progress, cancellation, retry, processing, and failure states.
- [ ] Clean up abandoned temporary files safely.

### Exit gate

- [ ] Authorized users can create, edit, transition, comment on, and attach files to tasks.
- [ ] Invalid workflow/permission changes fail with actionable safe explanations.
- [ ] No failed or queued mutation is shown as confirmed.
- [ ] Large task lists and long task details meet agreed performance budgets.

## 11. Phase 6 — Dependencies and advanced approvals

**Goal:** Bring the completed Phase 3 execution constraints into the mobile workflow.

### Task dependencies

- [ ] Display “Blocked by” and “Blocking” relationships.
- [ ] Show dependency completion and status summaries.
- [ ] Implement authorized add/remove dependency selection.
- [ ] Handle cycle rejection without exposing unauthorized records.
- [ ] Explain blocked transitions and link to visible blockers.
- [ ] Refresh dependent task state after relevant changes.

### Advanced approvals

- [ ] Implement pending, decided, and requested-by-me lists.
- [ ] Implement approval detail with stages, requester, context, prior decisions, and comments.
- [ ] Implement approve, reject, and request-changes actions when permitted.
- [ ] Enforce required comments and policy-specific confirmations.
- [ ] Refresh authoritative approval state immediately before a decision.
- [ ] Handle stale/concurrent decisions without optimistic finalization.
- [ ] Display escalation/due context and next-stage outcome.
- [ ] Preserve audit/correlation identifiers in telemetry without decision content.

### Exit gate

- [ ] Mobile behavior matches backend dependency and approval authorization tests.
- [ ] A dependency cannot be used to create a cycle through mobile.
- [ ] Concurrent approval decisions resolve safely and visibly.
- [ ] Approval policy configuration remains absent or clearly links to web.

## 12. Phase 7 — Notifications, deep links, search, and resilience

**Goal:** Bring users into the correct authorized context and make failure/recovery dependable.

### Notifications

- [ ] Add backend device-installation registration, update, preference, and revocation endpoints.
- [ ] Request notification permission only after an explanatory prompt.
- [ ] Register and rotate Expo push tokens.
- [ ] Remove provider-invalid tokens through backend delivery handling.
- [ ] Define privacy-safe push payload schemas with opaque identifiers.
- [ ] Implement notification Inbox and read/unread behavior.
- [ ] Implement foreground notification handling and targeted query invalidation.
- [ ] Test notification receipt/open in foreground, background, and terminated states.

### Deep links

- [ ] Configure development custom scheme.
- [ ] Configure verified iOS universal links and Android app links.
- [ ] Support invitations, projects, tasks, approvals, and Inbox intents.
- [ ] Resolve authentication before protected navigation.
- [ ] Resolve or switch organization before opening a scoped record.
- [ ] Handle forbidden, removed, malformed, and expired targets safely.

### Search and resilience

- [ ] Implement debounced global search for permitted projects, tasks, and people.
- [ ] Keep recent query history organization-scoped.
- [ ] Add connectivity and stale-data indicators.
- [ ] Add bounded read retries and user-directed mutation retries.
- [ ] Preserve recoverable form drafts across crashes/restarts.
- [ ] Add global recovery without destroying drafts.
- [ ] Document post-MVP SQLite outbox prerequisites; do not enable it prematurely.

### Exit gate

- [ ] Every supported push/deep link opens the correct organization and record or fails safely.
- [ ] Push payload and application logs pass sensitive-data review.
- [ ] Core screens recover correctly from offline, timeout, expired-session, and server-error scenarios.

## 13. Phase 8 — Quality, pilot, and production release

**Goal:** Prove the app is safe and supportable before public availability.

### Accessibility and usability

- [ ] Complete screen-reader audit for all critical flows.
- [ ] Test largest supported font sizes and display scaling.
- [ ] Test reduced motion, increased contrast, and non-color status communication.
- [ ] Verify 44 × 44 point minimum interactive targets.
- [ ] Conduct structured pilot usability sessions and resolve critical findings.

### Performance and reliability

- [ ] Set and measure cold start, Home-ready, navigation, API, bundle, and memory budgets.
- [ ] Profile long lists, attachment-heavy tasks, and lower-end Android devices.
- [ ] Meet the agreed crash-free session target.
- [ ] Verify refresh, query, upload, and navigation behavior under slow/flaky networks.
- [ ] Complete production-scale load checks for mobile-facing backend endpoints.

### Security and privacy

- [ ] Complete mobile threat model and security review.
- [ ] Confirm tokens/content never enter analytics, logs, crash reports, URLs, or push payloads.
- [ ] Validate local data cleanup on logout and organization switch.
- [ ] Run dependency, secret, and mobile configuration scans.
- [ ] Review permissions and platform purpose strings.
- [ ] Confirm privacy labels/data safety declarations.
- [ ] Verify account deletion and support escalation flows.

### Release operations

- [ ] Complete Maestro critical-flow tests on the supported device matrix.
- [ ] Prepare App Store and Play Store copy, screenshots, icon, privacy, and support metadata.
- [ ] Configure internal testing/TestFlight and Play testing tracks.
- [ ] Document build, submission, signing, rollback, and incident runbooks.
- [ ] Configure EAS Update runtime versions and environment-matched channels.
- [ ] Test staged over-the-air rollout and emergency stop/rollback.
- [ ] Run internal alpha and resolve release-blocking issues.
- [ ] Run limited external pilot and review health/product metrics.
- [ ] Obtain product, engineering, security/privacy, and operations sign-off.
- [ ] Submit production binaries and monitor review/release.

### Exit gate

- [ ] Critical end-to-end flows pass on the supported iOS/Android matrix.
- [ ] No open release-blocking security, privacy, accessibility, data-loss, or tenant-isolation issue.
- [ ] Support, monitoring, rollback, and ownership are operational.
- [ ] Production release is approved and store deployment is verified.

## 14. Critical end-to-end release checklist

- [ ] New user creates an account and enters an authorized workspace.
- [ ] Invited user authenticates, accepts the invitation, and reaches the intended workspace/resource.
- [ ] Returning user securely restores a session.
- [ ] Multi-workspace user selects and switches organizations without cache leakage.
- [ ] User changes between System, Light, and Dark appearance modes.
- [ ] User finds a project from Home and Projects.
- [ ] User creates, edits, transitions, comments on, and attaches a file to a task.
- [ ] User sees and resolves or understands a task dependency blocker.
- [ ] Approver opens a request and makes a valid decision.
- [ ] User opens task and approval push notifications from terminated state.
- [ ] Expired-session, offline, timeout, forbidden, not-found, and conflict states recover safely.
- [ ] Sign out revokes the session where reachable and removes local sensitive data.

## 15. Cross-repository dependencies

| Dependency | Owning repository | Needed by | Status |
|---|---|---|---|
| OpenAPI schema completeness/export | Backend | Phase 3 | Not started |
| Body-based rotating refresh endpoint | Backend | Phase 3 | Not started |
| Mobile idempotency support | Backend | Phases 5–7 | Not started |
| Record version/conflict contract | Backend | Phases 5–6 | Not started |
| Device/push registration endpoints | Backend | Phase 7 | Not started |
| Universal/app link association files | Web/deployment | Phase 7 | Not started |
| Mobile repository and EAS ownership | Mobile/operations | Phase 1 | Local repository created; remote/EAS pending |
| App-store accounts and legal metadata | Product/operations | Phase 8 | Not started |

## 16. Risks and controls

| Risk | Control |
|---|---|
| Mobile types drift from backend | Generated OpenAPI client and CI drift/breaking-change checks |
| Refresh token leakage or race | SecureStore, body-based refresh, rotation, single-flight refresh |
| Cross-organization cached data | Organization-prefixed keys plus hard cache clearing on switch |
| Older installed apps break after backend changes | Backwards-compatible APIs and contract versioning |
| Offline UI implies false success | Explicit draft/queued/confirmed states and restricted mutation retries |
| Approval conflict causes wrong decision display | Authoritative refresh and no optimistic finalization |
| Push notification leaks content | Opaque identifiers and safe preview defaults |
| Schedule expands through desktop parity | Enforce v1/deferred scope at phase reviews |
| Theme/accessibility defects multiply | Complete design primitives and audits before feature expansion |
| Store/release access delays launch | Resolve accounts, credentials, policies, and ownership in Phase 0/1 |

## 17. Progress summary

| Phase | Status | Completed | Owner | Evidence/notes |
|---|---|---:|---|---|
| 0. Product and delivery readiness | In progress | 59% | Product | Repository, identifiers, platform floors, scope, navigation, and themes settled; store/pilot/legal decisions remain |
| 1. Repository and app foundation | In progress | 81% | Engineering | Isolated Node 22, quality/pre-commit gates, EAS profiles, runtime providers, identifiers, security, and telemetry pass |
| 2. Design system and navigation shell | In progress | 88% | Engineering | Complete primitive contracts, gallery, contrast tests, reduced-motion hook, persistent themes, responsive navigation, and quick create pass checks; fonts/assets/device evidence remain |
| 3. API contracts and authentication | Not started | 0% | — | — |
| 4. Home and projects | Not started | 0% | — | — |
| 5. Tasks and collaboration | Not started | 0% | — | — |
| 6. Dependencies and advanced approvals | Not started | 0% | — | — |
| 7. Notifications, deep links, search, and resilience | Not started | 0% | — | — |
| 8. Quality, pilot, and production release | Not started | 0% | — | — |

Update this table only from verified checklist state. Record material scope or sequencing changes in the decision log below.

## 18. Decision log

| Date | Decision | Reason |
|---|---|---|
| 2026-09-04 | Build a focused mobile companion before full desktop parity | Daily execution and approvals provide the clearest initial mobile value |
| 2026-09-04 | Use bottom tabs on phones and an adaptive sidebar on supported larger screens | Preserves discoverability without consuming phone workspace |
| 2026-09-04 | Ship System, Light, and Dark appearance modes in v1 | Theme support is a foundation concern and expensive to retrofit |
| 2026-09-04 | Use a dedicated authenticated workspace chooser | Organization selection follows identity verification and stays separate from credentials |
| 2026-09-04 | Generate shared types from backend OpenAPI | Prevents manual type drift without sharing server entities |
| 2026-09-04 | Defer the general offline mutation outbox | Requires proven idempotency and conflict semantics before safe delivery |
