# Implementation Audit

Date: 2026-06-02

Scope:
- Audited the non-admin product only.
- Traced frontend and backend for user-facing flows.
- Did not implement fixes in this pass.
- Admin-only screens were excluded from feature completeness, but admin-only dependencies were noted where they block normal org-admin usage.

Implementation follow-up note:
- Subsequent code changes on 2026-06-02 addressed two Wave 1 issues in the workspace:
  - tour dismissal/skipped state is now persisted and no longer reset during organization switching
  - user onboarding completion is now wired from the modal to the backend `PATCH /users/:id/onboarding` route
- Additional code changes on 2026-06-02 addressed two more Wave 1 issues:
  - `/settings` and `/profile` now use a live account surface instead of static demo pages
  - password changes are now wired to the backend `POST /users/:id/update-password` route
- Additional code changes on 2026-06-02 also addressed avatar persistence:
  - account settings can now upload a user avatar, and the backend persists the avatar URL through the storage service
- These changes should still receive browser QA before the audit items are considered fully closed.

Status legend:
- `Working`: flow exists and is wired with no obvious blocker from code trace.
- `Partial`: flow exists but is missing important UI, persistence, backend handling, or polish.
- `Broken`: route mismatch, missing page, placeholder UI, or code path that will not complete as intended.
- `Missing`: backend and/or UI surface expected for the product is absent.

## Executive Summary

The deployed app is now usable for core login, onboarding, dashboard entry, menu loading, and several collaboration flows, but it is not yet feature-complete as a polished end-user product.

Highest-priority issues found:

1. Tour experience is not truly persistent and restarts too easily.
2. User onboarding completion is not persisted to backend.
3. Profile/settings/account management is not production-ready.
4. Organization management for org admins is still effectively admin-only.
5. Billing/subscription visibility for normal users/org admins is missing.
6. AI Overview and Reports are not real modules yet.
7. Error boundary / not-found / recovery UX is missing at app level.

## Cross-Cutting Issues

### 1. Tour keeps reappearing
Status: `Broken`

What is happening:
- `OnboardingGate` auto-starts the tour whenever `tourComplete` is false.
- `useTourStore` does not persist `skipped`, so reloading the app restarts the tour if the user skipped or dismissed it instead of completing it.
- `OrganizationSwitcher` resets the tour on every organization switch, which can cause repeated tour prompts.

Evidence:
- [useTourStore.ts](/var/www/html/trackr-main/track-a-project/app/lib/stores/useTourStore.ts)
- [OnboardiingGate.tsx](/var/www/html/trackr-main/track-a-project/app/components/Dashboard/_components/onboarding/OnboardiingGate.tsx)
- [OrganizationSwitcher.tsx](/var/www/html/trackr-main/track-a-project/app/components/OrganizationSwitcher.tsx)

Needed:
- Persist “dismissed/skipped” behavior per user and probably per organization.
- Separate “Replay Tour” from “Auto-start Tour”.
- Stop resetting tour state on every org switch unless explicitly desired.

### 2. General app-level recovery UX is missing
Status: `Missing`

What is missing:
- No app-level `error.tsx`
- No `global-error.tsx`
- No `not-found.tsx`
- No central empty/error fallback strategy across modules

Evidence:
- No route-level error boundary files found under `app/`

Needed:
- Global error boundary
- Route-level not-found handling
- Consistent retry/recover patterns
- Better unauthorized/session-expired UX

### 3. Search is UI-only
Status: `Broken`

What is happening:
- Navbar search is a local input with no query execution, navigation, or result panel.

Evidence:
- [Navbar.tsx](/var/www/html/trackr-main/track-a-project/app/components/Dashboard/Navbar.tsx)

Needed:
- Actual cross-module search backend or scoped search experience
- Search results page / dropdown
- Keyboard UX and empty states

## Module Audit

### Auth
Status: `Partial`

What works:
- Login page exists and now surfaces backend messages.
- Organization creation signup exists.
- Join-by-invitation signup exists.
- Logout exists.

What is missing or broken:
- Forgot password / verify / reset actions exist, but matching user-facing pages are not present in `app/auth`.
- Login page links to `/forgot-password`, but no corresponding page was found.
- Legacy and newer auth patterns coexist, which increases drift risk.

Evidence:
- [auth actions](/var/www/html/trackr-main/track-a-project/app/actions/auth.ts)
- [app/auth](/var/www/html/trackr-main/track-a-project/app/auth)

Needed:
- Complete password recovery UI
- Verification screens
- Clean one-path auth UX

### Onboarding
Status: `Partial`

What works:
- Organization onboarding flow exists and now preserves draft on refresh.
- Organization onboarding completion route is now wired.

What is missing or broken:
- User onboarding modal does not persist completion to backend.
- `finish()` in user onboarding has the backend call commented out.
- Backend user onboarding route is still declared as `id/onboarding` instead of `:id/onboarding`.
- Sample task creation in user onboarding is stubbed/commented out.

Evidence:
- [UserOnboardingModal.tsx](/var/www/html/trackr-main/track-a-project/app/components/Dashboard/_components/onboarding/UserOnboardingModal.tsx)
- [users.controller.ts](/var/www/html/trackr-main/track-a-project-backend/src/users/controllers/users.controller.ts)

Needed:
- Persist user onboarding completion
- Fix user onboarding route
- Decide whether sample-task creation is real or remove the step

### Organization / Workspace Management
Status: `Partial`

What works:
- Users can belong to multiple organizations.
- Organization switcher exists.
- Organization onboarding updates org details.

What is missing or broken:
- There is no non-admin organization settings/profile management UI for org admins.
- Onboarding copy says org settings can be changed later, but outside admin there is no proper org settings surface.
- Organization logo upload is not implemented on backend. `file` is accepted but not handled.
- Organization menu seeding required fallback repair logic instead of a stable bootstrap path.
- Team invite UI is only mounted from admin pages, so org-admin users in the normal app do not have a dedicated non-admin team management/invite surface.

Evidence:
- [organizations.service.ts](/var/www/html/trackr-main/track-a-project-backend/src/organizations/services/organizations.service.ts)
- [organizations.controller.ts](/var/www/html/trackr-main/track-a-project-backend/src/organizations/controllers/organizations.controller.ts)
- [TeamInviteModal.tsx](/var/www/html/trackr-main/track-a-project/app/components/modalMain/TeamInviteModal.tsx)
- TeamInviteModal usage only found in admin tools page

Needed:
- Non-admin organization settings page
- Editable org profile, slug, logo, limits, plan display
- Team management / invite management for org admins in normal workspace UI

### Organization Switcher
Status: `Partial`

What works:
- Switching organizations updates token scope through backend action.
- Menu refetch is tied to current organization.

What is missing or broken:
- Switching org resets onboarding and tour state unconditionally.
- Tier badge config does not match backend tier names:
  - backend uses `free`, `basic`, `professional`, `enterprise`
  - switcher styles expect `free`, `pro`, `business`, `enterprise`
- This means some tiers will render with fallback styling.

Evidence:
- [OrganizationSwitcher.tsx](/var/www/html/trackr-main/track-a-project/app/components/OrganizationSwitcher.tsx)
- [subscriptionTier.ts](/var/www/html/trackr-main/track-a-project-backend/src/utils/constants/subscriptionTier.ts)

Needed:
- Align tier names
- Stop resetting unrelated product state on switch
- Add clearer switch loading / success feedback

### Dashboard
Status: `Working` with gaps

What works:
- Dashboard route exists.
- Sidebar and navbar render based on current organization and menus.

What is missing or broken:
- Heavy dependency on current org state means failures cascade widely.
- No global fallback when org context is missing.
- Search bar is non-functional.

Evidence:
- [dashboard layout](/var/www/html/trackr-main/track-a-project/app/(dashboard)/layout.tsx)
- [Navbar.tsx](/var/www/html/trackr-main/track-a-project/app/components/Dashboard/Navbar.tsx)
- [Sidebar.tsx](/var/www/html/trackr-main/track-a-project/app/components/Dashboard/Sidebar.tsx)

### Projects
Status: `Partial`

What appears implemented:
- Project listing
- Project drawer
- Comments
- Board view
- Invite-related tables/components

Risks / gaps observed:
- Large amount of legacy / duplicate project UI variants still in repo.
- Some comments/share/project invite components appear inconsistent in style and architecture.
- Search/filter complexity is high and likely needs QA for empty/error states and org switching.

Needed:
- Consolidate duplicate project UIs
- Full functional QA on create/edit/delete/invite/comment flows

### Notes
Status: `Partial`

What works:
- Notes page exists.
- Quick notes popover exists.
- CRUD actions exist.
- Position/order update routes exist.

What is missing or broken:
- Backend task filter uses the wrong alias: `tasks.id` instead of joined alias `task`, so task-scoped note filtering looks broken.
- Update/order/position operations do not appear to verify the note belongs to the current organization in service layer.
- Quick note UX exists, but error handling is inconsistent.

Evidence:
- [note actions](/var/www/html/trackr-main/track-a-project/app/actions/note.ts)
- [notes.controller.ts](/var/www/html/trackr-main/track-a-project-backend/src/notes/controllers/notes.controller.ts)
- [notes.service.ts](/var/www/html/trackr-main/track-a-project-backend/src/notes/services/notes.service.ts)

Needed:
- Fix task filter
- Enforce org ownership checks on mutating note routes
- QA both popover notes and full notes page

### Documents
Status: `Partial`

What appears implemented:
- Documents page exists
- Editor/new document components exist
- Document/file listing exists

Gaps from trace:
- No strong evidence of complete recovery/error UX
- No app-level boundary coverage
- Needs functional QA around uploads, org switching, and permissions

Needed:
- End-to-end QA for create/edit/upload/share/search
- Better empty/error flows

### Folders
Status: `Partial`

What appears implemented:
- Folder page exists
- Folder modal/tree exist
- Actions exist

Gaps from trace:
- Needs full CRUD QA, especially move/reorder/org context behaviors
- No dedicated recovery UX if org context or folder tree fails

Needed:
- Functional QA
- Better error/empty states

### Whiteboards
Status: `Partial`

What appears implemented:
- Whiteboards page exists
- Save/new modals exist
- Organization-aware actions exist

Gaps from trace:
- Needs full QA on create/save/delete/open with org context
- Collaboration behavior and persistence need validation

Needed:
- End-to-end QA for board creation, save, retrieval, delete

### Chat / Messages
Status: `Partial`

What works:
- Conversations, peer list, message fetch, start conversation, send message actions exist.
- Backend endpoints exist and are org-guarded.

What is missing or broken:
- No clear evidence of read receipts / typing / delivery UX on frontend.
- Need QA for empty conversations, peer discovery, socket sync, and org switching.

Evidence:
- [message actions](/var/www/html/trackr-main/track-a-project/app/actions/message.ts)
- [messages.controller.ts](/var/www/html/trackr-main/track-a-project-backend/src/messages/controllers/messages.controller.ts)

Needed:
- Functional QA
- Better loading/empty states
- Clarify real-time scope

### Peers
Status: `Partial`

What works:
- Peer list / invite list / sent invites actions exist.
- Backend peer invite service appears to send email and notifications.

What is missing or broken:
- Peer invite modal is older-style and likely bypasses newer app patterns.
- Needs full QA for accept/reject/delete flows.
- Need confirmation that role values and invite semantics still match current product expectations.

Evidence:
- [PeerInviteModal.tsx](/var/www/html/trackr-main/track-a-project/app/components/modalMain/peers/PeerInviteModal.tsx)
- [peers actions](/var/www/html/trackr-main/track-a-project/app/actions/peers.ts)
- [users.service.ts](/var/www/html/trackr-main/track-a-project-backend/src/users/services/users.service.ts)

Needed:
- Full peer invite QA
- Modernize peer invite UI and validation

### Team Invites
Status: `Partial` / `Missing in non-admin UI`

What works:
- Backend organization invitation creation exists.
- Invite link generation exists.
- TeamInviteModal exists.

What is missing or broken:
- Non-admin org-admin workspace does not appear to expose this modal.
- Invite management views are currently admin-oriented.
- No dedicated non-admin team-management surface.

Evidence:
- [TeamInviteModal.tsx](/var/www/html/trackr-main/track-a-project/app/components/modalMain/TeamInviteModal.tsx)
- Only mounted from admin tools page
- [organizations.service.ts](/var/www/html/trackr-main/track-a-project-backend/src/organizations/services/organizations.service.ts)

Needed:
- Team invite access from normal org-admin workspace
- Sent/pending invite management for org-admins outside admin UI

### Notifications
Status: `Partial`

What works:
- Notification list page exists.
- Notification popover exists.
- Read, read-all, delete actions exist.

What is missing or broken:
- Real-time socket notification setup is commented out in popover.
- Popover still contains hardcoded `testMessages` scaffolding.
- Needs QA for cross-org notification filtering and refresh behavior.

Evidence:
- [NotificationPopOver.tsx](/var/www/html/trackr-main/track-a-project/app/components/NotificationPopOver.tsx)
- [notifications actions](/var/www/html/trackr-main/track-a-project/app/actions/notifications.ts)

Needed:
- Decide on real-time notification support
- Clean legacy mock/test scaffolding

### Profile
Status: `Broken`

What exists:
- A `/profile` page exists.
- Dropdown “My Profile” menu item exists conceptually.
- Backend user update endpoint exists.

What is broken:
- The profile dropdown item does not navigate anywhere. `action: "profile"` only closes the menu.
- The `/profile` page is hardcoded demo data, not the real user.
- Profile avatar rendering in dropdown ignores `user.avatar` and always uses `/default.png` when avatar exists.
- Backend user avatar upload handling is not implemented. `file` is accepted but ignored.

Evidence:
- [Profile.tsx](/var/www/html/trackr-main/track-a-project/app/components/Profile.tsx)
- [profile page](/var/www/html/trackr-main/track-a-project/app/(dashboard)/profile/page.tsx)
- [users.service.ts updateUser](/var/www/html/trackr-main/track-a-project-backend/src/users/services/users.service.ts)

Needed:
- Real profile page wired to live data
- Working profile menu navigation
- Avatar upload + display pipeline

### Settings / Account
Status: `Broken`

What exists:
- Settings page route exists.
- Backend endpoints exist for account update and password update.
- Older profile/password forms exist in components.

What is broken:
- Settings page is mostly static UI with no real data loading or save wiring.
- Frontend `updatePassword()` server action points to `/account/update-password`, but backend route is actually `POST /users/:id/update-password`.
- Existing password/profile forms are old isolated components and are not clearly integrated into the main settings page.

Evidence:
- [settings page](/var/www/html/trackr-main/track-a-project/app/(dashboard)/settings/page.tsx)
- [account actions](/var/www/html/trackr-main/track-a-project/app/actions/account.ts)
- [PasswordChangeForm.tsx](/var/www/html/trackr-main/track-a-project/app/components/forms/PasswordChangeForm.tsx)
- [ProfileForm.tsx](/var/www/html/trackr-main/track-a-project/app/components/forms/ProfileForm.tsx)
- [users.controller.ts](/var/www/html/trackr-main/track-a-project-backend/src/users/controllers/users.controller.ts)

Needed:
- Real settings shell wired to backend
- Password change flow using one canonical route
- Editable profile/account/preferences UI

### Billing / Subscriptions
Status: `Missing in non-admin UI`

What exists:
- Backend billing/subscription services exist.
- Current plan endpoint exists on organizations.
- Billing cancel endpoint exists.

What is missing or broken:
- No non-admin subscription or billing UI was found.
- Users/org-admins cannot view plan/limits/history from the normal app.
- Billing cancel endpoint currently has no visible auth guard in controller, which is a backend security concern.

Evidence:
- No non-admin subscription page usage found in `app/`
- [organizations.controller.ts current-plan](/var/www/html/trackr-main/track-a-project-backend/src/organizations/controllers/organizations.controller.ts)
- [billing.controller.ts](/var/www/html/trackr-main/track-a-project-backend/src/billing/controllers/billing.controller.ts)

Needed:
- Non-admin billing/subscription screen
- Plan/limits/history/cancel UX
- Tighten backend auth/authorization around billing endpoints

### AI Overview
Status: `Broken / Placeholder`

What exists:
- Route exists

What is broken:
- Page is a mock wireframe with hardcoded cards, tabs, and fake analytics
- No backend integration detected

Evidence:
- [AI Overview page](/var/www/html/trackr-main/track-a-project/app/(dashboard)/ai-overview/page.tsx)

Needed:
- Real data contracts
- Real insights pipeline or hide feature until ready

### Reports
Status: `Missing`

What exists:
- Route exists

What is broken:
- Page returns an empty `<div />`

Evidence:
- [Reports page](/var/www/html/trackr-main/track-a-project/app/(dashboard)/reports/page.tsx)

Needed:
- Entire reports module

### Notifications Page / Auxiliary Module Pages
Status: `Partial`

What exists:
- Notifications page has filters, table, and action wiring

Risks:
- Mix of older patterns and newer app patterns
- Needs cleanup and QA

Evidence:
- [notifications page](/var/www/html/trackr-main/track-a-project/app/(dashboard)/notifications/page.tsx)

## Backend / API Design Issues That Will Affect Product Quality

These are not all UI issues, but they will surface as broken or inconsistent product behavior:

1. User onboarding completion route still declared incorrectly
   - [users.controller.ts](/var/www/html/trackr-main/track-a-project-backend/src/users/controllers/users.controller.ts)

2. User onboarding modal does not call backend completion route
   - [UserOnboardingModal.tsx](/var/www/html/trackr-main/track-a-project/app/components/Dashboard/_components/onboarding/UserOnboardingModal.tsx)

3. User avatar file handling is stubbed
   - [users.service.ts updateUser](/var/www/html/trackr-main/track-a-project-backend/src/users/services/users.service.ts)

4. Organization logo file handling is stubbed
   - [organizations.service.ts update](/var/www/html/trackr-main/track-a-project-backend/src/organizations/services/organizations.service.ts)

5. Billing cancel endpoint lacks visible auth guard
   - [billing.controller.ts](/var/www/html/trackr-main/track-a-project-backend/src/billing/controllers/billing.controller.ts)

6. Notes task filter alias looks wrong
   - [notes.service.ts](/var/www/html/trackr-main/track-a-project-backend/src/notes/services/notes.service.ts)

## Recommended Implementation Order

### Wave 1: Fix obvious broken user trust issues
1. [Done] Tour persistence / repeat logic
2. [Done] Persist user onboarding completion
3. [Done] Real profile/settings/account surface
4. [Done] Working password change flow
5. [Done] Working avatar upload/display
6. [Done] Non-admin org settings surface

### Wave 2: Fill missing product surfaces
1. Team management and team invites for org-admins in non-admin app
2. [Done] Billing/subscription UI for org-admins
3. [Done] Forgot password / recovery pages
4. [Done] Search UX
5. [Done] Error boundaries / not-found / recovery screens
6. [Done] Route access model for hidden vs blocked vs upgrade-gated pages
7. [Done] Dedicated access-state screens and clearer upgrade messaging

### Wave 3: Replace placeholders
1. AI Overview
2. Reports

### Wave 4: Separate personal dashboard from workspace analytics
1. Rework `/dashboard` into a true user dashboard centered on my tasks, my deadlines, my active work, my mentions, and my project activity
2. Move organization-wide delivery health, collaborator rankings, activity heatmaps, and workspace-wide rollups out of the personal dashboard
3. Treat `/reports` as the primary workspace reporting surface for org-level analytics and operational summaries
4. Treat `/ai-overview` as the workspace intelligence layer for synthesized insights, risks, patterns, and recommendations
5. Keep only a lightweight workspace snapshot on the user dashboard when it helps orientation, not a full admin-style analytics surface
6. Review `/admin/home` and related org-admin surfaces to decide what workspace-health information belongs there versus in reports
7. Define separate personal `Reports` and personal `AI Overview` experiences for individual users based on their own work, deadlines, activity, and collaboration footprint
8. Define org-admin `Reports` and org-admin `AI Overview` experiences focused on organization-wide health, team performance, access coverage, and operational risk

### Wave 5: Harden collaboration modules
1. [Done] Notifications real-time behavior
2. [Done] Notes QA and org-safety checks
3. [Done] Peer invite flow cleanup
4. [Done] Documents / folders / whiteboards / chat regression pass

### Wave 6: Full UI/UX review
Design guardrail: Preserve the product's existing personality and avoid turning the UI into a generic "AI app" aesthetic. Prefer grounded product UI, clearer workflows, stronger hierarchy, and brand-consistent polish over futuristic gradients, floating glass panels, or novelty motion.

Checklist:
1. Inventory every non-super-admin route and group them by module, owner flow, and completion status
2. Review each screen for clarity of purpose, obvious next action, readable hierarchy, and mobile responsiveness
3. Check navigation labels, menu grouping, breadcrumbs, and route naming for consistency across the app
4. Review the frontend admin sidebar separately for structure, discoverability, and reduced clutter
5. Standardize empty states, loading states, error states, and success feedback across core modules
6. Review forms for label clarity, helper text, validation timing, disabled states, and submission feedback
7. Check tables, cards, and detail pages for spacing, scanability, and action placement consistency
8. Review dashboard, projects, documents, folders, whiteboards, chat, peers, notes, notifications, and settings end-to-end for UX regressions
9. Remove UI patterns that feel overly synthetic, over-decorated, or disconnected from the rest of the product
10. Add only restrained motion where it improves comprehension or feedback, not decoration
11. [Done] Produce a punch list of fixes grouped into quick wins, medium lifts, and deeper redesign candidates
12. Run a final regression pass after UI cleanup to confirm functional behavior still matches the completed waves
13. [Done] Fix the notifications route type/runtime regression where `currentOrganization` is referenced without being defined, because the current Wave 6 branch does not type-check cleanly
14. [Done] Align admin reports and admin AI overview with real organization-level data instead of reusing the personal `/users/dashboard` payload and presenting it as workspace intelligence
15. [Done] Make workspace access changes update the live sidebar, search, and route-gate state immediately after save, instead of requiring a full reload or organization switch to reflect menu changes
16. [Done] Unify route search behavior so the dedicated search page and navbar quick-search use the same access model, especially for hidden, gated, or blocked workspace routes

Punch list:

Quick wins:
- Replace `process.env.NEXT_NODE_ENV` checks with `process.env.NODE_ENV` so development-only dashboard diagnostics behave predictably
- Continue standardizing workspace-empty, loading, and no-results states on legacy pages like notifications, peers, and notes
- Trim dead imports, unused state, and legacy scaffolding from heavily edited dashboard routes to reduce maintenance noise during QA
- [Done] Trim dead imports, unused state, and peer-invite scaffolding from the notifications route so the screen is easier to maintain during the remaining Wave 6 pass
- [Done] Trim duplicate drag handlers, random fallback jitter, and unused note-management scaffolding from the notes route so drag interactions and maintenance are more predictable
- [Done] Trim duplicated tab rendering, dead whiteboard route state, and noisy modal/socket logging so the whiteboards flow is easier to maintain during the remaining Wave 6 pass
- Make search results and blocked-route states more explicit about the next best action, especially when an upgrade or admin change is required

Medium lifts:
- Refactor oversized route files such as notifications, notes, and dashboard/admin analytics into smaller view sections with shared feedback components
- [Done] Refactor the notifications route into a smaller notification-focused screen instead of carrying forward unrelated peer-invite state and handlers
- [Done] Refactor the notes route around one typed note model with local-first freeform dragging and optimistic grid reordering instead of duplicated legacy drag paths
- [Done] Refactor the whiteboards route and modal around a simpler workspace-scoped board flow instead of repeated tab content blocks and brittle close/title-sync handling
- [Done] Refactor the admin reports and admin AI overview routes around shared query-driven workspace context loading instead of separate ad hoc data orchestration patterns
- [Done] Refactor the personal reports route around the same query-driven reporting pattern used by the admin analytics surfaces, and add a matching refresh affordance to workspace billing
- Normalize table toolbars, filter rows, and pagination patterns so core modules behave consistently on desktop and mobile
- Unify dashboard and admin page content hierarchy so summary cards, narrative panels, and action bars follow one predictable structure
- Review route naming and nav group labels across dashboard, reports, AI overview, workspace admin, and access screens for clearer mental models

Deeper redesign candidates:
- Build true organization-wide data sources for admin reports and admin AI overview instead of contextualizing personal dashboard payloads
- Revisit the full notes and notifications experience to separate quick actions from full management views with clearer information architecture
- Define a product-wide design language for analytics/reporting surfaces so personal, admin, and workspace views feel related without becoming visually repetitive
- Run a route-by-route mobile UX pass across high-traffic modules to decide where dense tables should become stacked cards or alternate mobile layouts

### Wave 7: Dev Task Agent
1. Dev Task Agent Implentation

## Modules That Need Full Manual QA After Fixes

Even where code paths exist, these should be manually exercised end-to-end:

- Dashboard
- Projects
- Documents
- Folders
- Whiteboards
- Chat
- Peers
- Notes
- Notifications
- Organization switching
- Onboarding (org + user)
- Auth recovery flows once pages exist

## Final Note

After the functional waves above, we should run a dedicated UI/UX review across every product surface. That review should include the frontend admin experience, especially the admin sidebar, while keeping super admin out of scope for now.
