# Pinned Projects in the Sidebar — Product Requirements Document

## Document status

- Status: Proposed
- Product area: Projects and navigation
- Target release: To be scheduled
- Primary surfaces: Project details, Projects page, desktop sidebar, mobile sidebar
- Related roadmap: Standalone enhancement; this document does not modify `PRODUCT_FEATURE_ROADMAP.md`

## Summary

Users should be able to pin projects they can access to a personal section of the application sidebar. A pinned project is a private navigation preference belonging only to the current user in the current organization. Pinning or unpinning a project must never change another collaborator's sidebar.

The feature must work for project owners, organization administrators, and project peers with Viewer, Contributor, or Editor access.

## Problem

Users who participate in several projects repeatedly navigate through the Projects page to reach the projects they use most. Existing organization menu settings control navigation for the organization and are not suitable for personal shortcuts.

A project-level `show_in_sidebar` field would also be incorrect because it would apply one collaborator's preference to everyone with access to that project.

## Goals

- Let each user pin frequently used projects to their own sidebar.
- Make pinned projects available across devices and authenticated sessions.
- Support both owned projects and projects shared with the user.
- Respect organization and project authorization at every read and write.
- Remove inaccessible or deleted projects from the rendered shortcut list.
- Keep personal project shortcuts separate from organization-wide menu configuration.
- Provide clear UI language indicating that the preference is personal.

## Non-goals

- Pinning projects for another user.
- Configuring a shared set of pinned projects for an organization or team.
- Changing project visibility, membership, or permissions.
- Replacing the Projects page or project search.
- Pinning projects across organizations in one combined list.
- Adding favorites, priority, or notification behavior to a project.

## Product principles

1. **Personal:** A pin belongs to one user and cannot affect collaborators.
2. **Organization-scoped:** Only pins for the active organization appear.
3. **Permission-safe:** A pin never grants access to a project.
4. **Fast:** The sidebar should render pinned projects without loading full project records or task data.
5. **Recoverable:** Unpinning removes only the shortcut, never the project.

## Terminology

- **Pinned project:** A project selected by the current user for quick sidebar access.
- **Pin:** The personal preference record connecting a user and project in an organization.
- **Accessible project:** A project the current user can view through ownership, organization-administrator access, or confirmed project membership.

## User stories

### Core stories

- As a project owner, I can pin my project so I can open it quickly.
- As a project peer, I can pin a shared project without changing another collaborator's sidebar.
- As a Viewer or Contributor, I can pin a project even though I cannot edit its settings.
- As a user in multiple organizations, I see only the pins belonging to the active organization.
- As a user, I can unpin a project without deleting or modifying the project.
- As a user, I no longer see a pinned project after I lose access to it.

### Administrative stories

- As an organization administrator, I can pin projects for myself but not silently pin them for other members.
- As a project owner, I do not need to approve another member's personal pin.

## User experience

### Project details control

Add a personal preference control to the project details or settings surface:

```text
Show in my sidebar  [toggle]
```

Supporting copy:

```text
This only changes your sidebar. Other project members are not affected.
```

The control must be available to every user who can view the project. It must not be restricted to users who can manage project settings.

If placing the control inside a settings page that Viewers cannot access would make it unavailable, also provide it in an accessible project action menu or project header.

### Projects page control

Each accessible project card or row should provide a secondary action:

- `Pin to sidebar` when not pinned.
- `Unpin from sidebar` when pinned.

The action should use a pin icon and expose an accessible label.

### Sidebar section

Add a `Pinned projects` section beneath primary navigation and above lower utility/account actions.

Each pinned-project item displays:

- Project icon or a deterministic fallback icon.
- Project title.
- Active-route treatment when the user is viewing that project.
- A link to the canonical project details route.
- An accessible unpin action in the expanded sidebar.

When the main sidebar is collapsed, show project icons with tooltips. Do not show a context-menu or direct unpin action in the collapsed sidebar. The mobile sidebar must show the same organization-scoped list and close after navigation.

When no projects are pinned, hide the section rather than displaying an empty state permanently.

### Limits

The first release uses a plan-configured pin allowance with an absolute product maximum of 10 pinned projects per user per organization. The effective limit is the lower of the organization's plan allowance and 10. If a plan does not define an allowance, default it to 10.

When the limit is reached:

- Disable additional pin actions.
- Explain that the user must unpin another project first.
- Continue allowing existing pins to be removed.

### Ordering

The first release includes drag-and-drop ordering in the expanded desktop sidebar and an accessible reorder interaction for keyboard users. Order pins by `position` ascending and then `created_at` ascending. New pins are appended to the end.

Dropping a project updates the sidebar optimistically and persists the complete ordered pin list through the reorder endpoint. A failed request restores the previous order and displays an error toast. Mobile uses the saved order but does not need a drag interaction in the first release.

## Functional requirements

### Pin a project

1. The user selects `Show in my sidebar` or `Pin to sidebar`.
2. The backend verifies organization membership and project View permission.
3. The backend resolves the organization's plan allowance and verifies that the user is below the effective limit, which can never exceed 10.
4. The backend creates the preference idempotently.
5. The sidebar updates without a full-page browser refresh.

Repeated pin requests must return the existing pin or a successful no-op response.

### Unpin a project

1. The user disables the toggle or selects `Unpin from sidebar`.
2. The backend deletes only the current user's matching preference.
3. The project disappears from the sidebar without a full-page refresh.
4. The project and other users' pins remain unchanged.

Repeated unpin requests must be successful no-ops.

### Load pinned projects

The sidebar request returns only pins that satisfy all of the following:

- The pin belongs to the authenticated user.
- The pin belongs to the active organization.
- The project belongs to the active organization.
- The project still exists.
- The user can currently view the project.

The response must not include full task, status, peer, document, or activity collections.

### Access changes

- A pin remains valid while the user has effective `ProjectPermission.VIEW` through at least one authorization path.
- Removing a project-peer record deletes the pin only when that removal also eliminates the user's effective View permission. For example, a user who remains an organization administrator retains the pin after their direct peer membership is removed.
- If a peer invitation is pending, the invited user cannot pin the project yet.
- If a membership becomes blocked or unconfirmed, the pin must no longer render.
- When all paths granting project View permission are explicitly revoked, delete that user's pin as part of the access-removal workflow.
- If access is later restored, the user must choose to pin the project again.

### Project deletion

Deleting a project must delete all associated pin records through a database foreign key with `ON DELETE CASCADE`.

### User and organization deletion

Deleting a user or organization must cascade-delete its pin records.

## Data model

Create a dedicated entity and table. Do not add `show_in_sidebar` to `projects`, `project_peers`, `global_menus`, or `organization_menus`.

Suggested table:

```text
user_project_sidebar_pins
-------------------------
id               uuid primary key
organization_id  uuid not null
user_id           bigint/int not null
project_id        int not null
position          int not null default 0
created_at        datetime not null
updated_at        datetime not null
```

Constraints and indexes:

- Unique: `(organization_id, user_id, project_id)`
- Index: `(organization_id, user_id, position)`
- Foreign key `organization_id` → organizations, `ON DELETE CASCADE`
- Foreign key `user_id` → users, `ON DELETE CASCADE`
- Foreign key `project_id` → projects, `ON DELETE CASCADE`
- Check or application validation ensuring `position >= 0`

The organization ID is intentionally stored even though it can be derived from the project. It makes organization-scoped reads efficient and supports explicit tenant filtering. The service must still verify that the project's organization matches the record.

`projects.id` is a globally unique primary key across the system, not an identifier scoped within an organization. The pin schema and foreign key rely on this invariant. A future cross-organization sharing feature must continue referencing the same globally unique project record rather than creating organization-local records that reuse a project ID. The pin's organization ID records the context in which the shortcut is visible; it does not disambiguate project identity.

## API requirements

All endpoints require authentication and the existing organization-access guard. The active organization is supplied through `x-organization-id`.

Pin, unpin, and reorder mutations must use the application's shared per-user rate limiter. Apply a combined default limit of 30 sidebar-pin mutations per user per minute across all three mutation endpoints. A rejected request returns `429 Too Many Requests` and must not partially mutate pin state. Listing pins continues to use the standard authenticated read limit.

### List pins

```http
GET /users/me/sidebar-projects
```

Example response:

```json
{
  "data": [
    {
      "id": "pin-uuid",
      "projectId": 42,
      "title": "Website Redesign",
      "slug": "website-redesign",
      "icon": "briefcase",
      "position": 0,
      "role": "editor"
    }
  ],
  "limit": 10
}
```

### Pin a project

```http
PUT /users/me/sidebar-projects/:projectId
```

Use `PUT` because the operation is idempotent.

Responses:

- `200` when the pin already exists.
- `201` when a pin is created.
- `403` when the user cannot view the project.
- `404` when the project does not exist in the active organization.
- `409` when the user has reached their plan's effective pin limit. The response should include the effective limit.

### Unpin a project

```http
DELETE /users/me/sidebar-projects/:projectId
```

Return a successful response even when no matching pin exists.

### Reorder pins

This endpoint and the corresponding drag-and-drop UI are required in the first release:

```http
PATCH /users/me/sidebar-projects/order
```

Example request:

```json
{
  "projectIds": [42, 81, 17]
}
```

Validation rules:

- IDs must be unique.
- Every ID must already be pinned by the current user in the active organization.
- The submitted collection must contain the complete current pin set.
- Positions are rewritten transactionally.

## Authorization model

Use the centralized project authorization service.

### Read and pin

Require `ProjectPermission.VIEW`. This deliberately includes:

- Project owner.
- Organization administrator.
- Confirmed connected Viewer.
- Confirmed connected Contributor.
- Confirmed connected Editor.

Project ownership and project-peer membership are different access paths. Do not change the existing project-peer endpoint merely to represent owners as peers. The pin service must use centralized project authorization, and pinned-project responses must normalize the user's effective role whether access comes from ownership, organization administration, or peer membership.

### Unpin

The user may remove their own preference even if project access has since been revoked. The deletion query must be scoped by authenticated `user_id` and `organization_id`; it must not require project Edit permission.

### Prohibited actions

- A user cannot create or delete another user's pin.
- A project owner cannot pin a project for collaborators.
- Organization administrators cannot manage another member's pins through these endpoints.
- A project in another organization must behave as not found.

## Frontend state and caching

Create a dedicated pinned-project query/store rather than adding dynamic project records to organization menu configuration.

Recommended query key:

```text
["sidebar-project-pins", organizationId]
```

On pin or unpin:

- Optimistically update the sidebar when safe.
- Roll back and show an error toast if the request fails.
- Invalidate or refetch the pinned-project query after success.
- Keep desktop and mobile sidebar instances synchronized through the same query cache/store.

On organization change:

- Clear the previous organization's visible pins immediately.
- Clear organization-specific optimistic state, selected pin state, reorder state, limit information, errors, and pending mutation indicators.
- Cancel in-flight pin queries and mutations for the previous organization where the client library supports cancellation.
- Load the new organization's pins.
- Never show stale project shortcuts from the previous organization while loading.
- Key every cache entry and mutation context by organization ID; do not use an unscoped global pin array.
- Ignore a response when its organization ID no longer matches the active organization, even if cancellation failed or the response arrived after the switch.
- Never copy the previous organization's pins into the new organization's cache as placeholder data.

## Performance requirements

- Return at most 10 records.
- Select only sidebar fields.
- Avoid N+1 membership queries.
- The list endpoint should use one bounded query or a small fixed number of queries.
- Target server response time: less than 300 ms at p95 under normal load.
- Sidebar loading must not block rendering of the main navigation.

## Accessibility requirements

- The toggle has an associated label and supporting description.
- Pin and unpin icon buttons have explicit accessible names.
- Project links are keyboard accessible.
- Tooltips are available when the sidebar is collapsed.
- Active project state is not communicated by color alone.
- Loading and failure states do not trap keyboard focus.

## Failure and empty states

### Loading

Render a compact skeleton within the pinned-project area without shifting the entire sidebar.

### Empty

Hide the section. The Projects page may explain how to pin a project.

### Request failure

Keep standard navigation usable. Show a retry action or allow the next sidebar refresh to retry. Do not display cached pins from a different organization.

### Inaccessible project

Exclude the project from the response and optionally remove the stale pin asynchronously.

## Analytics and observability

Track product events without recording sensitive project content:

- `project_sidebar_pin_created`
- `project_sidebar_pin_removed`
- `project_sidebar_pin_limit_reached`
- `project_sidebar_pins_reordered`

Opening a pinned project must use the application's canonical project-navigation event with `navigation_source: "pinned_sidebar"`. Do not also emit a separate `project_sidebar_pin_opened` event for the same click. If no canonical project-navigation event exists when this feature is implemented, introduce one shared event for every project entry point instead of adding a pinned-sidebar-only open event.

Suggested properties:

- Organization ID
- Project ID
- User's project role
- Entry point (`project_settings`, `project_card`, or `sidebar`)
- Current pin count

Backend logs should capture authorization failures, invalid tenant combinations, and database constraint failures with request context.

## Migration and rollout

1. Add the pin table, constraints, and indexes.
2. Deploy backend endpoints with the frontend feature disabled.
3. Verify organization, user, project, and membership authorization.
4. Deploy sidebar query and rendering.
5. Add project-level pin controls.
6. Enable for internal users.
7. Monitor request failures and sidebar performance.
8. Enable for all organizations.

No data backfill is required because existing users have no implicit pins.

## Testing requirements

### Backend unit tests

- Owner can pin their project.
- Organization administrator can pin an accessible project.
- Viewer, Contributor, and Editor peers can pin.
- Pending, blocked, unconfirmed, and removed peers cannot pin.
- A user cannot pin a cross-organization project.
- Repeated pin requests are idempotent.
- Repeated unpin requests are idempotent.
- The plan-configured pin limit and absolute maximum of 10 are enforced.
- One user's pin does not appear for another user.
- Project deletion cascades to pin deletion.
- List results exclude inaccessible projects.
- Reordering rejects missing, duplicate, foreign, and unpinned project IDs.
- Pin, unpin, and reorder mutations share the per-user rate limit and return `429` without writing when it is exceeded.

### Frontend tests

- Toggle reflects the current personal pin state.
- Pinning updates desktop and mobile sidebars without a full refresh.
- Unpinning from either surface updates every visible instance.
- Loading, error, limit, and empty states render correctly.
- Organization switching clears stale pins.
- Late query or mutation responses from the previous organization cannot repopulate or modify the active organization's pin state.
- Collapsed sidebar provides accessible project tooltips.
- Drag-and-drop reordering persists, rolls back on failure, and supports keyboard users.

### End-to-end scenarios

1. An owner pins a project and sees it after signing in on another device/session.
2. A Viewer pins a shared project; another collaborator's sidebar remains unchanged.
3. A peer loses access and the pinned shortcut disappears.
4. A user switches organizations and sees the correct organization-specific pins.
5. A delayed response from the previous organization arrives after switching and is ignored.
6. A project is deleted and its shortcut disappears without causing sidebar errors.
7. A user reaches their plan allowance and receives a clear limit message.
8. A user reorders pins and sees the same order after signing in again.

## Acceptance criteria

- Users can pin and unpin any project they are authorized to view.
- Pins are private to the current user and scoped to the active organization.
- Peers can pin projects without affecting owners or other peers.
- Pinned projects appear in desktop and mobile sidebars with project icons.
- Pin and unpin changes appear without a full-page refresh.
- Users cannot pin inaccessible or cross-organization projects.
- Pins remain while any valid View-access path remains; losing all access paths or deleting the project removes the shortcut.
- The plan-configured allowance is enforced and can never exceed the general maximum of 10 pins.
- Drag-and-drop ordering ships in the first release and persists across sessions.
- The collapsed sidebar does not expose an unpin action; direct unpin is available in the expanded sidebar.
- Database uniqueness prevents duplicate pins.
- Project, user, and organization deletion cannot leave orphaned pins.
- The implementation does not modify organization-wide menu visibility.
- Pin, unpin, and reorder writes are protected by the shared per-user mutation rate limit.
- Opening a pinned project produces one canonical navigation event with `navigation_source: "pinned_sidebar"` and is not double-counted.
- Backend, frontend, migration, and authenticated end-to-end tests pass before release.

## Deferred enhancements

- User-configurable sidebar groups or folders.
- Recently visited projects.
- Automatic suggestions based on project activity.
- Shared team shortcuts curated by administrators.
- Cross-organization combined favorites.
- Keyboard shortcuts for pinned projects.

## Resolved product decisions

1. Losing one access path does not remove a pin while another path still grants View permission. Losing all effective project access deletes the affected user's pin; restoring access does not restore it automatically.
2. Project owners are not converted into artificial peer records. The pin service uses centralized authorization and returns a normalized effective role for owners, organization administrators, and peers.
3. Drag-and-drop ordering is included in the first release, along with persistence, optimistic rollback, and keyboard-accessible reordering.
4. Pin allowances are configurable by plan, subject to an absolute product maximum of 10 per user per organization. An undefined plan allowance defaults to 10.
5. Direct unpin is available only in the expanded sidebar and other full project surfaces. The collapsed sidebar provides navigation and tooltips without an unpin context menu.
