# Tailpoint Mobile App — Product Requirements Document

**Status:** Foundation approved for design and implementation planning  
**Document version:** 1.0  
**Last updated:** 2026-09-04  
**Platforms:** iOS and Android  
**Working product name:** Tailpoint Mobile

## 1. Product decision

Tailpoint Mobile will be a fast, calm companion to the web workspace. It will make the work people need to act on away from a desk immediately accessible: see priorities, open projects, create and update tasks, understand blockers, respond to approvals, and handle notifications.

The first release will not reproduce every web administration or builder surface. Automation rules, workflow design, integration configuration, audit retention/export, billing, and other configuration-heavy tools remain web-first until the mobile execution experience is dependable.

## 2. Product promise

> Know what matters, move work forward, and approve decisions from anywhere.

The app should feel focused rather than dense, capable rather than complicated, and dependable rather than flashy. Every screen should answer one of three questions quickly:

1. What needs my attention?
2. What is happening in this project?
3. What can I safely do next?

## 3. Goals and non-goals

### Goals for the first public release

- Securely authenticate existing users and support new organization creation or invitation-based joining.
- Let a user move between organizations without signing in again.
- Make assigned, overdue, blocked, and recently changed work easy to find.
- Support the daily task lifecycle: create, view, edit, assign, transition, comment, attach, and complete.
- Display dependencies clearly and prevent or explain invalid transitions.
- Let approvers review context and approve, reject, or request changes.
- Deliver useful push notifications that deep-link to the relevant record.
- Support complete light and dark themes, with a system-default option, from the first release.
- Remain understandable under slow, intermittent, or absent connectivity.
- Meet a production quality bar for accessibility, security, performance, and crash recovery.

### Non-goals for the first public release

- Full parity with the desktop workspace.
- Mobile builders for automations, workflows, forms, templates, Custom Field schemas, or integrations.
- Audit export and retention administration.
- Organization billing, entitlement administration, or platform administration.
- Offline collaborative editing or background synchronization that claims guaranteed delivery.
- Rich document, whiteboard, or video-call authoring.

## 4. Primary users and jobs

### Individual contributor

- Check today's work and urgent changes.
- Update status, fields, deadlines, and progress.
- Add a comment or attachment while away from a desk.
- Understand why a task is blocked and what it depends on.

### Project lead or manager

- Scan project health and late or blocked work.
- Reassign work, clarify priority, and create follow-up tasks.
- Review project activity without opening the desktop app.
- Respond to approval requests promptly.

### Organization owner

- Join or switch workspaces safely.
- Handle approval decisions and important operational alerts.
- Use the web app for policy, automation, integration, billing, and audit configuration.

## 5. Experience principles

1. **Attention before inventory.** Lead with work requiring action, not a database-shaped menu.
2. **One primary action per view.** Secondary operations belong in contextual menus or sheets.
3. **Progressive disclosure.** Show essentials first; expand metadata and history on demand.
4. **Safe confidence.** Destructive or policy-sensitive actions explain consequences and require deliberate confirmation.
5. **Honest connectivity.** Never imply that a server mutation succeeded while it is queued or failed.
6. **Platform-native behavior.** Respect iOS and Android navigation, back behavior, keyboard, share sheet, haptics, and accessibility conventions.
7. **Continuity with web.** Terminology, statuses, permissions, project colors, and domain behavior must match Tailpoint web.

## 6. Brand, theme, and feel

### Design direction

The existing Tailpoint visual language is the foundation: deep slate surfaces, teal as the primary action color, restrained blue/green accents, generous rounded cards, clear spacing, and minimal decorative gradients. Mobile should refine this into a more tactile, quieter interface.

The intended emotional qualities are:

- calm and organized;
- trustworthy and operational;
- warm enough to feel human;
- compact without feeling cramped;
- polished without ornamental animation.

### Initial design tokens

Tokens are semantic and must be tested for WCAG contrast before implementation is frozen.

| Role | Light theme | Dark theme | Use |
|---|---:|---:|---|
| Brand primary | `#008080` | `#35B8B2` | Primary actions, active navigation |
| Brand blue | `#0294E2` | `#55B8EC` | Informational accents and links |
| Accent lime | `#ADED22` | `#BCEB55` | Small success/highlight accents only |
| Canvas | `#F6F8FA` | `#0B1220` | App background |
| Surface | `#FFFFFF` | `#111B2E` | Cards, sheets, controls |
| Text strong | `#122033` | `#F5F8FC` | Titles and primary text |
| Text muted | `#667085` | `#AAB6C8` | Supporting text |
| Border | `#E4E9F0` | `#29364B` | Dividers and input boundaries |
| Danger | `#D92D20` | `#FF776D` | Destructive actions and errors |
| Warning | `#D97706` | `#F6B94A` | Due/blocked warnings |
| Success | `#14804A` | `#43C47B` | Completed and successful states |

### Typography

- Use **Figtree** for the product interface, matching the existing web product's friendly, legible tone.
- Use the system fallback while fonts load; the splash must never wait on a network font.
- Use a compact type scale with strong hierarchy: 28/34 display, 22/28 title, 17/24 body emphasis, 15/22 body, 13/18 metadata.
- Respect device font scaling; do not disable scaling to preserve a layout.

### Shape, spacing, and motion

- Use an 8-point spacing system with 4-point increments for fine alignment.
- Cards: 16–20 px radius; sheets: 24 px top radius; controls: 12–14 px radius.
- Minimum interactive target: 44 × 44 points.
- Motion should clarify navigation or state change, generally 150–250 ms, and honor reduced-motion preferences.
- Use haptics sparingly for successful completion, approval decisions, and destructive confirmation.

### Light and dark mode

- Theme selection is a first-release feature with three choices: **System**, **Light**, and **Dark**.
- **System** is the default and follows the device appearance in real time.
- An explicit Light or Dark choice overrides the device setting and persists across launches on that device.
- Every screen, modal, sheet, navigation surface, splash treatment, empty state, chart, status, and input must support both themes.
- Status meaning cannot depend on color alone, and both themes must meet the same contrast and accessibility requirements.
- Theme changes apply immediately without restarting the app or losing navigation or form state.
- Native status bars, navigation bars, keyboards, and other supported system UI should match the active theme.
- Project and user-selected colors must use contrast-safe containers instead of being rendered directly against either canvas.

### Illustration and imagery

- Prefer simple abstract line/shape illustrations derived from paths, milestones, connections, and progress.
- Avoid generic stock photography and character-heavy onboarding art.
- Product screenshots or real UI fragments may appear in store imagery, not inside core onboarding.

### Voice and content

- Direct, short, and specific: “Task updated” rather than “Your changes have been successfully saved.”
- Explain blocked actions: “Complete ‘Design review’ before moving this task to Done.”
- Never blame the user for network, permission, or server failures.
- Use sentence case across headings, buttons, and labels.

## 7. Information architecture

### Signed-out routes

- Splash/bootstrap
- Welcome/get started
- Sign in
- Create an account
- Join through invitation
- Forgot/reset password handoff
- Legal and privacy links

### Authenticated workspace-entry routes

These routes appear after identity has been verified but before the main app opens:

- **Choose workspace** — shown when the user has multiple active organization memberships or no valid previously selected organization.
- **Create an organization** — available from the workspace chooser when self-serve creation is permitted.
- **Accept invitation** — completes a preserved invitation after authentication, then selects the joined organization.
- **Onboarding** — completes any required profile, organization, or notification setup.

Users with exactly one active organization skip the chooser unless they arrived through an invitation for another organization. The app may restore a previously selected valid organization for returning users, but it must never infer access from locally stored organization data alone.

### Signed-in primary navigation

Use four bottom tabs and one global create action:

1. **Home** — My Work, attention queue, recent activity, quick resume.
2. **Projects** — project list, pinned/recent projects, project details.
3. **Inbox** — approvals and notifications in one attention surface with clear filters.
4. **You** — profile, organization switcher, preferences, security, help, sign out.

A global `+` action opens a bottom sheet for Task first; additional create targets can be added later. Search is available from Home and Projects and may become a dedicated tab only if usage data supports it.

### Project navigation

A project opens to an overview with compact sub-navigation:

- Overview
- Tasks
- Activity
- More

“More” exposes available read-oriented modules such as milestones, files, members, and updates. Capability- or permission-unavailable modules are omitted rather than disabled without explanation.

## 8. Screen requirements

### 8.1 Splash and bootstrap

- Display the Tailpoint mark on a solid brand/canvas surface.
- Read the local session, selected organization, theme, and onboarding state.
- Keep the native splash visible until the first navigation decision is ready; do not add an artificial timer.
- Route to Welcome, Sign in, Onboarding, or the authenticated app.
- If refresh fails because the device is offline, allow access only to clearly labeled cached read data; require reauthentication for mutations once the session cannot be validated.

### 8.2 Welcome/get started

- One strong promise, a restrained product illustration, and two actions: **Get started** and **Sign in**.
- At most three short value statements: organize work, remove blockers, decide faster.
- Skip lengthy carousels. Returning users should never see this screen after completing it.

### 8.3 Sign in

- Email and password, password visibility control, forgot password, loading state, keyboard-safe layout.
- Preserve a pending invitation or deep link through authentication.
- Use generic credential errors and rate-limit feedback without exposing account existence.
- SSO/passkeys are later additions when backend support exists; the layout must leave room for them.
- After successful authentication, resolve memberships before navigating: restore a still-valid previous workspace, open the dedicated workspace chooser when a choice is needed, or continue a pending invitation.

### 8.4 Sign up and workspace entry

- Collect the minimum account fields accepted by the current backend.
- After account creation, offer the contextually correct route:
  - accept a valid invitation;
  - create a new organization;
  - select an existing organization if membership already exists.
- Terms and privacy consent are explicit and versioned where required.

### 8.5 Choose workspace

- Use a dedicated full-screen chooser, not a field inside Sign in.
- List active organizations with name, logo/initials, and the user's role where appropriate.
- Put the last-used valid organization first without selecting it silently for a first-time session.
- Offer **Create organization** and **Join with invitation** only when those actions are available.
- Selecting an organization establishes the server-recognized organization context, clears any prior organization-scoped cache, and opens Home.
- Provide sign out and account recovery paths so the user cannot become trapped at this stage.
- During normal app use, tapping the organization name opens the same selector as a sheet or full-screen route; switching follows the same cache-isolation rules.

### 8.6 Onboarding

Onboarding is resumable and should take under two minutes:

1. Confirm profile name and optional avatar.
2. Create or confirm organization membership.
3. Set notification preference with a clear explanation; request OS permission only after the user opts in.
4. Choose a first project or create one if authorized.
5. Land on Home with a short contextual coach mark for creating a task.

Do not ask for every preference up front. Role, workflow, automation, and integration setup belongs on web.

### 8.7 Home / My Work

- Greeting and organization switcher.
- Attention summary: overdue, due soon, blocked, and approvals awaiting the user.
- My tasks grouped by Today, Upcoming, and Later, with filtering and pull-to-refresh.
- Recent or pinned projects.
- A compact offline/stale-data indicator when applicable.
- Empty state that offers to create a task or open a project rather than celebrating an ambiguous empty server response.

### 8.8 Projects

- Searchable list with pinned/recent first and an all-projects section.
- Project card shows name, color/icon, concise progress, relevant count, and optional health indicator.
- Respect archived access and project roles.
- Project overview shows description, progress, members, milestones/updates summary, task status distribution, and recent activity.
- Project creation in the first release is a short, permission-gated flow: name, description, color, and optional dates. Advanced setup links to web.

### 8.9 Tasks

#### Task list

- Default to a performant list grouped or filtered by workflow status.
- Initial mobile release supports list view; a horizontal board may follow after usability and performance validation.
- Filters: assignee, status, priority, due date, blocked, and “mine.”
- Persist recent filters per project as non-sensitive preferences.

#### Task detail

- Title, status, priority, assignees, due date, description, Custom Fields, dependencies, subtasks if supported, attachments, comments, and activity.
- Put the current status and next valid transitions near the top.
- Surface blocking dependencies before offering completion.
- Show permission or workflow restrictions in context.
- Load long sections progressively; task identity and current state render first.

#### Create/edit task

- Quick create requires title and project, with optional assignee, due date, and priority.
- Full editor exposes fields supported by that project's current workflow and Custom Field schema.
- Validate locally for responsiveness and on the server for authority.
- Save explicit drafts when offline; never silently claim a server-created task.

### 8.10 Dependencies

- Show “Blocked by” and “Blocking” sections with completion/status summaries.
- Allow authorized users to add/remove dependencies through searchable task selection.
- Prevent cycles and display the backend's safe explanation.
- When a transition is unavailable because of a blocker, identify the blocking task the user is permitted to see; otherwise use a privacy-safe generic explanation.

### 8.11 Advanced approvals

- Inbox filter for pending, decided, and requested-by-me.
- Approval card shows subject, stage, requester, due/escalation context, and safe summary.
- Detail displays all context the backend authorizes, current stage/progress, prior decisions, and comments.
- Actions: approve, reject, or request changes when allowed; collect a required comment when policy requires it.
- Show stale/conflicting decisions and refresh before retrying; never optimistically finalize a policy-sensitive decision.
- Approval policy creation and stage configuration remain web-only in the first release.

### 8.12 Inbox and notifications

- Combine actionable approvals and notification history without conflating their read/decision state.
- Filters: all, unread, mentions, assignments, approvals, deadlines, and system.
- Mark one or all as read, with undo where practical.
- Push taps deep-link to the authorized record after session and organization resolution.
- Notification previews contain no secrets or sensitive task content by default; users may opt into richer previews later.

### 8.13 Search

- Search tasks, projects, and permitted people through the existing global search boundary.
- Debounce input, show recent local queries, distinguish entity types, and support deep links.
- Never merge cached results from different organizations.

### 8.14 You, settings, and organization switching

- Profile, appearance (System/Light/Dark), notification preferences, privacy/security, help, app version, and sign out.
- Organization switcher displays only active memberships.
- Switching clears organization-scoped in-memory/query state before loading the new organization.
- Optional biometric app lock is a convenience privacy feature, not a substitute for server authentication.

## 9. Initial module sequence

| Order | Module | Release | Reason |
|---:|---|---|---|
| 1 | App shell, design system, auth, organization context | Foundation | Every protected module depends on it |
| 2 | Home / My Work | MVP | Establishes immediate daily value |
| 3 | Projects and project overview | MVP | Provides work context and navigation |
| 4 | Task list, detail, create, edit, comments | MVP | Core mobile execution loop |
| 5 | Task dependencies | MVP | Required for truthful task transitions |
| 6 | Approval inbox and decisions | MVP | High-value mobile action with Phase 3 support |
| 7 | Notifications and deep links | MVP | Brings users back to actionable work |
| 8 | Attachments, search, richer project activity | MVP hardening | Completes common field workflows |
| 9 | Offline drafts and resilient mutation outbox | Post-MVP/pilot | Add after online mutation semantics are proven |
| 10 | Recurring work controls and richer workflow views | Later | Useful but not required for first companion release |

### Web-first until later evidence supports mobile design

- Automation builder and execution administration
- Workflow and Custom Field schema builders
- Integration/webhook configuration and delivery operations
- Audit search/export/retention administration
- Form and template builders
- Billing, plan, entitlement, and organization administration
- Advanced reporting and portfolio configuration

## 10. Key end-to-end flows

### Returning user

Launch → validate/refresh session → restore organization → Home → open task → make allowed update → show confirmed state.

### Invitation

Open invitation link → preserve token → authenticate/create account → accept invitation → select organization → lightweight onboarding → project.

### Approval from push

Tap push → bootstrap session → switch/confirm target organization → open approval → refresh authoritative state → decide → show receipt and next stage.

### Offline task draft

Open cached task → connection is unavailable → edit supported fields → save labeled local draft → reconnect → review/send → server validates → reconcile or display a resolvable conflict.

## 11. Cross-cutting requirements

### Accessibility

- Target WCAG 2.2 AA for color and content behavior.
- Support screen readers, font scaling, reduced motion, increased contrast where available, keyboard navigation on tablets, and non-color status indicators.
- Every icon-only action has a clear accessibility label and state.

### Performance

- Avoid artificial splash delays; render a useful cached or skeleton shell quickly.
- Paginate lists and activity; virtualize long collections.
- Compress images before upload and never block the whole task screen on attachment thumbnails.
- Define measurable budgets during implementation: cold start, Home usable time, navigation latency, API latency, crash-free sessions, and bundle size.

### Reliability

- Every server mutation has loading, confirmed, failed, and retry behavior.
- Use idempotency keys for retried create/decision operations once the backend accepts them.
- Cached content displays its last-updated state when freshness matters.
- A global error boundary offers recovery without destroying drafts.

### Privacy and security

- Store no password, access token, raw attachment, or unrestricted sensitive payload in logs or analytics.
- Redact notification and crash metadata.
- Respect project roles, entitlements, workflow constraints, and organization boundaries on the server; mobile guards are usability controls only.
- Clipboard, screenshots, and biometric controls are evaluated before enterprise pilot rather than implied as protections.

### Product analytics

Track privacy-reviewed product events, not content:

- onboarding completed/abandoned by step;
- Home loaded and attention item opened;
- task create/update outcome and latency;
- approval opened/decided outcome and latency;
- push received/opened route category;
- organization switch outcome;
- offline draft created/reconciled;
- crash-free sessions and API error categories.

## 12. Success measures

Pilot targets must be finalized with baseline data, but the first release should measure:

- activation: signed-in users who open a project and complete one meaningful action within 24 hours;
- weekly mobile task-action rate;
- median time from approval notification to decision;
- successful mutation rate excluding validation rejections;
- push-to-relevant-screen success rate;
- crash-free session rate;
- retained weekly active mobile users;
- support incidents involving lost drafts, organization leakage, or incorrect approval state (target: zero).

## 13. Release gates

The app cannot enter external beta until:

- authentication refresh/logout and organization switching are mobile-safe;
- all P0 screens have loading, empty, offline, forbidden, and error states;
- task and approval mutations are verified against current backend authorization and workflow rules;
- deep links survive authentication and organization resolution;
- notification registration/revocation and privacy-safe payloads are tested on physical iOS and Android devices;
- accessibility checks cover all critical flows;
- logs, analytics, and crash reports pass a sensitive-data review;
- app-store privacy, permission, support, deletion, and legal metadata are ready;
- CI produces signed development, preview, and production builds from controlled branches;
- a rollback/kill-switch plan exists for remotely delivered JavaScript updates.

## 14. Product decisions still requiring owner input

These are not blockers for scaffolding, but should be decided before visual design is frozen:

- final App Store/Play Store name and subtitle;
- whether self-serve organization creation ships in mobile v1 or invitation/sign-in only;
- minimum supported iOS and Android versions;
- default notification categories and preview privacy;
- whether project creation belongs in MVP or first follow-up;
- analytics/crash provider and privacy policy wording;
- public beta geography and pilot organizations.

## 15. Definition of foundation complete

The mobile foundation is complete when a development build can launch with Tailpoint tokens, restore or reject a secure session, select an organization, navigate protected routes, call a generated typed API client, display standardized loading/error/offline states, switch light/dark themes, receive a test deep link, and pass unit plus one authenticated end-to-end smoke test on both platforms.
