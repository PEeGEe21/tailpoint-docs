# Mobile product convergence matrix

**Status:** Working baseline for prototype correction
**Last updated:** 2026-09-07
**Evidence:** Backend controllers/OpenAPI, current web routes and redesigned dashboard/project surfaces, Mobile PRD, and current mobile prototypes

## Decision key

- **Keep now** — supported by the backend and important to the mobile daily-work promise.
- **Keep later** — supported or planned, but not required before the core mobile loop is dependable.
- **Remove** — invented, misleading, unsupported, or inappropriate for mobile v1.
- **Redesign** — valid capability whose current mobile model, terminology, or interaction does not match the product.

## Cross-product findings

1. The mobile information architecture broadly matches the intended companion product: Home, Projects, Inbox, You, and global task creation.
2. The web redesign has a stronger visual identity than mobile: deep-slate/teal/sky hero surfaces, warm layered canvases, larger editorial hierarchy, and fewer uniformly weighted cards.
3. Several mobile mocks invent operational details that the API does not return. These must not survive into API models.
4. Mobile project and task statuses currently simplify backend-configurable values into hard-coded enums. The UI may group states for comprehension, but it must retain server identity, label, color, and allowed transitions.
5. Configuration-heavy web areas remain intentionally excluded from mobile v1.

## Feature and mock-data decisions

| Mobile concept | Backend/web evidence | Decision | Required correction |
|---|---|---|---|
| Welcome, sign in, sign up, recovery, invitation | Auth and organization contracts exist; equivalent web flows exist | Keep now | Align exact DTO fields, errors, and post-auth membership resolution before API wiring |
| Self-serve organization creation | Backend supports organization creation; v1 product decision remains open | Keep later | Keep prototype isolated and gate entry until product confirms v1 |
| Workspace selection | Backend supports memberships and organization-scoped sessions | Keep now | Reuse chooser after sign-in; switching during normal use remains later checkpoint |
| Home greeting | Web dashboard and PRD support it | Keep now | Use real profile name and active workspace; never hard-code a person |
| Attention queue | Backend exposes overdue tasks, dependencies, approvals, and notifications | Redesign | Derive cards from real typed sources; remove fabricated summary copy and actions |
| Today / Upcoming / Later tasks | PRD and task APIs support date-driven discovery | Keep now | Derive buckets from timestamps/time zone rather than display-label parsing |
| Pinned/recent projects | Sidebar/recent project APIs and web dashboard support them | Keep now | Preserve server project identity and status metadata |
| Project “On track/Attention” state | Backend project states are `active`, `upcoming`, `in_progress`, `inactive`, `completed`, `cancelled`, `on_hold`, `paused`, `on_review`, `overdue`, and `draft`; templates may customize presentation | Redesign | Replace invented project enums with server status key/label/color; health may be a separate derived metric only if defined |
| Project progress and due date | Present in web dashboard/project data | Keep now | Use backend calculation and nullable-date behavior |
| Project milestones, members, activity | Dedicated backend routes exist and web project surfaces use them | Keep now | Load progressively and retain real empty/permission states |
| Fabricated project descriptions and counts | Prototype-only content | Remove | Replace with coherent fixtures shaped exactly like contract responses |
| Task title and description | Create/update DTOs support both | Keep now | Preserve `description_html` separately when rich content is introduced |
| Mobile string priority (`low/medium/high`) | Backend has numeric `priority` and separate severity `low/medium/high/critical` | Redesign | Do not conflate priority and severity; expose severity now only if the web workflow uses it, and preserve numeric ordering internally |
| Mobile static task status (`todo/in-progress/done`) | Backend task status is an ID tied to project/custom workflow and exposes transition history | Redesign | Render server workflow status and only authorized transitions; remove static enum from API-facing model |
| Assignee full-name text field | Backend create/update currently accepts comma-separated assignee emails; web uses people selection | Redesign | Use a searchable member picker; never submit arbitrary display names |
| Task due date | Backend accepts nullable date-time | Keep now | Use a native date/time control and support clearing; remove raw `YYYY-MM-DD` production input |
| Task custom fields | Backend project/task custom-field contracts exist | Keep now | Render from project schema; omit unavailable fields and never submit undefined IDs |
| Task checklist displayed in current detail | No core task checklist instance contract was found; reusable checklist templates are a separate capability | Remove | Remove from the core task fixture/detail until an instance contract is confirmed |
| Task comments/discussion | Dedicated task discussion API exists | Keep now | Add after aligned task identity/state; include retry-safe failure behavior |
| Task attachments | Resource/attachment contracts exist | Keep now | Add progressive upload states after task core alignment |
| Task dependencies/blocking | Dedicated dependency contracts exist | Keep now | Replace fabricated `blockedBy` labels with typed dependency data and transition warnings |
| Approval inbox | Approval subject types, reviewers, stages, decisions, delegation, due date, and status exist | Keep now | Model subject context and allowed decisions from API |
| “Deployment gate”, “Scan clean”, commits, PR diff, RFC, and monthly infrastructure price | Not present in approval response contract; some may arrive only through opaque subject metadata | Remove | Do not present as universal approval fields; render subject-specific context only when explicitly supplied |
| Approval “Changes” action | Backend decision enum must remain authoritative | Redesign | Use exact server decisions and require comment when policy says so |
| Notification inbox/read state | Notification list and read operations exist | Keep now | Use `title`, `message`, `type`, `metadata`, `is_read`, and timestamps; remove invented icon semantics unless mapped by type |
| Profile identity and personal information | User/profile surfaces exist | Keep now | Replace fabricated verification, role, counts, membership date, and version copy with real/derived data |
| Workspace plan in Profile | Billing is web-first | Remove | Show organization and membership role; omit plan/entitlement marketing from mobile profile |
| Appearance preference | Mobile requirement | Keep now | Retain System/Light/Dark and apply instantly |
| Notification preferences | User notification-preference route exists | Keep later | Retain local prototype, integrate after notification taxonomy is approved |
| Security/password shortcut | Auth flows exist; configuration details need design | Keep later | Link only when a real destination exists |
| Automation, workflow/schema builders, integrations, audit export, billing, forms/templates | Backend/web features exist but PRD declares them configuration-heavy | Keep later | Remain web-first; mobile may show read-only consequences where needed |

## Visual direction: Evolved Tailpoint

Tailpoint retains teal as its signature action color, supported by deep ink/navy, clear sky blue, and a restrained warm accent. The visual system should express hierarchy through composition—not by coloring every component.

### Palette roles

| Role | Light | Dark | Purpose |
|---|---:|---:|---|
| Canvas | `#F4F7F6` | `#08111F` | Warm, quiet application background |
| Surface | `#FFFFFF` | `#101C2E` | Cards and controls |
| Ink hero | `#102A43` | `#07111F` | Branded high-emphasis regions |
| Primary teal | `#087F76` | `#45C5BB` | Primary action and active state |
| Sky | `#2D8CFF` | `#68B5FF` | Information, navigation support, data accents |
| Warm accent | `#F2B84B` | `#F6C861` | Focus, due-soon, and small human highlights |
| Text | `#102A43` | `#F4F8FC` | Primary content |
| Muted text | `#66788A` | `#A9B8C8` | Supporting content |

### Composition rules

- Use one expressive, deep-ink focal region per primary screen; do not turn every card into a gradient or accent surface.
- Prefer grouped lists and open composition over a separate bordered card for every fact.
- Use 28–32 px editorial headings, compact metadata, and short human copy.
- Teal owns primary actions; sky supports information; warm accent is sparse and meaningful.
- Use subtle layered circles/paths, project color, progress, and purposeful icon containers for personality without stock imagery.
- Keep status semantics distinct from brand colors and never rely on color alone.

## Representative-screen corrections

### Home

- Introduce an ink/teal focus hero tied to active workspace and the most important work signal.
- Replace hard-coded greeting identity and fabricated attention actions during API integration.
- Reduce repetitive card weight and strengthen the transition from attention to tasks to projects.

### Project Detail

- Make project identity and progress the focal composition.
- Replace simplified status keys with backend project status metadata.
- Retain Overview, Tasks, and Activity; add More only when backed by available modules and permissions.

### Task Detail/Create/Edit

- Lead with task identity and current workflow state.
- Separate numeric priority from severity.
- Replace static status choices with authorized project-workflow transitions.
- Replace free-text assignee and raw date input with member and native date controls.
- Remove checklist until a task checklist-instance contract is confirmed.

## Resume gate

Live feature API integration may start when the representative screens use this direction, mobile fixtures match contract shapes, unsupported mock fields are removed, and unresolved product decisions are explicitly gated rather than implied by the UI.
