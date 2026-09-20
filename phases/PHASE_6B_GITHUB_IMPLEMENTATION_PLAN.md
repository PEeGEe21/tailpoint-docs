# Phase 6B GitHub Integration Implementation Plan

**Status:** Proposed  
**Last updated:** 2026-09-20  
**Depends on:** [Phase 6A GitHub live-pilot sign-off](./PHASE_6A_GITHUB_IMPLEMENTATION_PLAN.md)

## Delivery decision

Phase 6B is the second GitHub integration increment inside the product's overall Phase 6. It turns the verified, project-scoped webhook foundation from Phase 6A into an installable repository integration with project-wide development visibility and carefully controlled two-way workflows.

Phase 6B must not start its rollout until the remaining Phase 6A release gate is complete. A live repository pilot in staging or production must prove webhook delivery, automatic and manual task linking, duplicate/redelivery handling, interrupted-delivery recovery, secret rotation overlap, silence/failure health alerts, and unresolved-reference diagnostics. The pilot result and any remediation must be recorded before 6A is signed off.

## Outcomes

Phase 6B should let an authorized project Owner:

- install and manage Tailpoint through a GitHub App, with OAuth used only where an explicit user authorization is required;
- discover organizations and repositories the installation may access;
- connect a repository without manually creating and maintaining a webhook;
- see project-level repository activity whether or not it is linked to a Tailpoint task;
- attach an unlinked GitHub artifact to an existing task or create a proposed task from it;
- opt into narrowly scoped two-way issue, comment, check, and deployment workflows;
- understand token health, API rate limits, synchronization state, failures, and recovery actions.

Phase 6B must preserve the Phase 6A principles of project and tenant isolation, least privilege, explicit user intent, bounded retained content, idempotency, observable delivery, and no hidden cross-project association.

## Entry gate: finish Phase 6A

Before engineering begins on 6B:

1. Run the signed five-event pilot against a live GitHub repository.
2. Confirm `ping`, issue, pull request, push, deployment status, and release deliveries receive timely acknowledgements and finish processing.
3. Confirm direct `TP-<taskId>` links, pull-request inheritance, manual link/move/unlink, and out-of-project rejection.
4. Redeliver the same provider delivery and prove that artifacts, activity, and links remain idempotent.
5. Simulate queue interruption and verify retry recovery without losing or duplicating a delivery.
6. Rotate the webhook secret, verify the overlap window, and confirm that the retired secret stops working.
7. Exercise silence and repeated-failure alerts plus unresolved-reference diagnostics.
8. Record evidence, defects, remediation, and a named release sign-off.

No 6B migration or installation rollout should be coupled to unresolved 6A pilot defects.

## Scope

### 1. Project repository activity

Add a project-level **Repository Activity** surface backed by the safe artifacts already captured in 6A. A GitHub event does not need a task reference to be useful or visible.

- Show linked and unlinked commits, pull requests, issues, deployments, releases, and supported GitHub Actions workflow/check activity.
- Display repository, branch or environment, event type, safe title/summary, actor label, state, provider time, receipt time, and external GitHub link.
- Group a push while still allowing its bounded commit summaries to be inspected.
- Filter by repository, event type, branch/environment, actor, state, linked/unlinked status, and date range.
- Search only the bounded fields Tailpoint already permits itself to retain.
- Clearly label association provenance: direct task reference, inherited pull request, manual, or unlinked.
- Allow an authorized Editor or Owner to attach an unlinked artifact to one or more existing tasks.
- Allow an authorized user to open a prefilled task-creation draft from an artifact. Nothing is created until the user confirms it.
- Preserve ordinary commits in the activity feed without forcing artificial task creation.
- Paginate by stable provider time plus ID and define retention before enabling high-volume repositories.

The feed is project-scoped. It must never reveal repositories, artifacts, task existence, or diagnostics from another organization or project.

### 2. GitHub Actions and software-delivery events

Extend the allowlist beyond Phase 6A repository artifacts where the GitHub App permissions and event contracts permit it:

- workflow runs and their queued, in-progress, successful, failed, cancelled, or timed-out state;
- check suites/check runs needed for task or release visibility;
- deployment and deployment-status transitions with environment context;
- safe links back to GitHub for logs and details rather than retaining logs in Tailpoint.

Workflow logs, diffs, source files, secrets, environment values, and arbitrary payload bodies must not be stored. Workflow/check events first appear in Repository Activity; linking them to tasks or releases is optional and explicit.

### 3. GitHub App installation and discovery

- Register separate GitHub App identities for development/staging and production.
- Use installation access tokens for repository operations. Do not use long-lived personal access tokens.
- Start installation from an organization-scoped Tailpoint route and carry a short-lived, signed, single-use state value.
- Validate installation ownership and callback state before showing any repository.
- List only organizations and repositories visible to the installation and current Tailpoint actor.
- Let an Owner select repositories and map each one to exactly one Tailpoint project per connection.
- Create, update, suspend, and remove repository connections from installation events without deleting retained history.
- Reconcile repository rename, transfer, archive, installation suspension, permission changes, and installation deletion.
- Retain the manual Phase 6A webhook connection as a controlled migration/fallback path until installed connections are proven.

### 4. Token and permission lifecycle

- Encrypt installation identifiers and any refresh credentials that require confidentiality; never expose tokens through product APIs, logs, analytics, audit payloads, or browser storage.
- Mint short-lived installation tokens on the server and cache them only within their bounded lifetime.
- Request the minimum GitHub App permissions for the enabled features. Read-only ingestion must remain possible without enabling write features.
- Show Owners the permissions currently granted, permissions newly required by an opt-in feature, and a reauthorization path.
- Treat suspension, revocation, expiry, permission downgrade, and token-generation failure as explicit health states.
- Audit installation, repository selection, permission changes, feature enablement, token-health transitions, and removal without recording credentials.

### 5. Optional two-way synchronization

Two-way behavior is default-off per connection and split into independently enabled capabilities:

- **Create GitHub issue from task:** show repository, title, body preview, labels, and assignee mapping before confirmation; store the returned stable issue identity.
- **Create Tailpoint task draft from GitHub issue:** prefill a draft from bounded issue fields and require confirmation unless a separately approved automation rule applies.
- **Status synchronization:** use an explicit mapping per project/repository. Define direction, supported states, loop prevention, and the source of truth before activation.
- **Comments and references:** synchronize only comments explicitly marked for sharing or created through the integration. Never mirror all task discussion by default.
- **Checks/status writes:** publish a bounded Tailpoint status or approval result only when configured and authorized.
- **Deployment-status writes:** keep separate from release management and require an explicit environment mapping.

Every outbound mutation requires a stable idempotency key, actor attribution, audit entry, permission revalidation, retry policy, and visible result. Provider-originated changes must not echo back indefinitely. Conflicts must be surfaced rather than resolved through silent last-write-wins behavior.

## Architecture and persistence

Prefer extending the 6A connection and delivery boundaries instead of creating a second unrelated GitHub subsystem.

Expected additions include:

- `github_installations`: organization, provider installation/account identity, encrypted credential metadata where needed, permissions, state, creator, and lifecycle timestamps;
- repository-selection records connecting an installation repository to a Tailpoint project and the existing connection/history;
- installation token cache metadata without persisted plaintext tokens;
- artifact visibility/query indexes for project feed pagination and linked/unlinked filtering;
- outbound synchronization operations and attempts with provider request IDs, idempotency keys, state, safe failure codes, retry timing, and completion timestamps;
- explicit field/status mappings and per-connection feature flags;
- provider cursor/checkpoint state where polling is unavoidable;
- safe rate-limit snapshots and installation health transitions.

Migration must preserve Phase 6A connection IDs, artifacts, task links, delivery history, manual suppressions, and audit provenance. Connecting an installation to an existing manual connection must be an explicit, idempotent adoption operation, not a delete-and-recreate flow.

## API and workspace surfaces

### Backend

- installation start/callback/status/removal endpoints;
- authorized organization/repository discovery endpoints;
- repository selection and Phase 6A connection-adoption endpoints;
- project Repository Activity list/detail endpoints with cursor pagination and filters;
- attach, move, unlink, and create-task-draft actions for unlinked artifacts;
- two-way feature configuration and mapping endpoints;
- outbound operation status/retry endpoints;
- installation/repository/rate-limit health endpoints;
- verified GitHub App webhook handling for installation, repository, workflow, check, and existing repository events.

### Workspace

- installation wizard with account, permission, repository, and project mapping steps;
- connection health that distinguishes webhook delivery, installation authorization, rate-limit, and outbound-sync health;
- Repository Activity tab with linked/unlinked states and task actions;
- permission-expansion and reauthorization flows;
- two-way configuration with clear previews, source-of-truth language, and per-feature opt-in;
- operation history and actionable recovery states.

### Admin and support

- tenant-safe aggregate installation, connection, delivery, rate-limit, and synchronization health;
- no repository names, artifact content, tokens, or task details in platform-wide views;
- scoped support inspection with audited access where content-level troubleshooting is authorized.

## Rate limits, reliability, and observability

- Track GitHub REST and GraphQL rate-limit headers per installation and operation class.
- Respect primary and secondary rate limits, `Retry-After`, and abuse protection with jittered backoff.
- Prioritize user-confirmed writes over background enrichment and stop nonessential polling near exhaustion.
- Avoid API calls when verified webhook data is sufficient; cache repository metadata with bounded freshness.
- Queue outbound work durably and recover expired leases. A Tailpoint request must not wait for GitHub beyond a short bounded provider timeout.
- Expose correlation from webhook delivery or user action through queue operation and provider response without exposing tokens or unsafe payloads.
- Measure acknowledgement latency, processing latency, queue age, provider API latency, remaining quota, retry count, terminal failure rate, installation suspension, and sync-loop prevention.
- Alert on sustained failure or exhaustion risk, not isolated transient responses.

## Security and privacy requirements

- Installation callbacks use signed, expiring, single-use state bound to the initiating Tailpoint user and organization.
- Repository discovery and mutation revalidate current organization membership, project permission, capability entitlement, installation state, and repository selection.
- GitHub App webhook signatures use exact raw bytes and support provider secret rotation.
- All provider identifiers are tenant/project scoped in reads and writes.
- URLs are allowlisted to GitHub HTTPS origins before rendering.
- Retained event fields remain bounded; full payloads, workflow logs, diffs, code, secrets, and unrestricted comments are excluded.
- Outbound issue/comment previews make exactly what will leave Tailpoint visible before confirmation.
- Installation deletion revokes future access while preserving authorized historical summaries under the documented retention policy.

## Delivery tickets

1. **GH2-00 — Phase 6A live-pilot gate:** execute, remediate, document, and sign off the production/staging pilot.
2. **GH2-01 — Repository Activity contracts:** define feed artifacts, push grouping, filters, pagination, retention, permissions, and linked/unlinked actions.
3. **GH2-02 — Project activity APIs and UI:** expose stored 6A artifacts at project level and implement attach/task-draft workflows.
4. **GH2-03 — GitHub Actions ingestion:** add bounded workflow/check event contracts, persistence, activity rendering, and operational tests.
5. **GH2-04 — GitHub App foundation:** implement app registration configuration, callback state, installation lifecycle, and secure token minting.
6. **GH2-05 — Discovery and connection adoption:** discover authorized accounts/repositories, select project mappings, and adopt existing 6A connections without history loss.
7. **GH2-06 — Permission and health UX:** add least-privilege feature permissions, reauthorization, suspension/revocation handling, and connection health.
8. **GH2-07 — Two-way issue workflow:** implement explicit task-to-issue creation, issue-to-task drafts, identity mapping, idempotency, audit, and retries.
9. **GH2-08 — Status and comment synchronization:** add opt-in mappings, conflict policy, loop prevention, and explicitly shared comments.
10. **GH2-09 — Checks and deployment writes:** add separately permissioned, opt-in provider writes with environment/release boundaries.
11. **GH2-10 — Rate limits and observability:** implement quota-aware scheduling, metrics, alerts, safe support views, and recovery controls.
12. **GH2-11 — Security and rollout validation:** complete tenant, permission, callback, token, SSRF/URL, replay, idempotency, conflict, load, and revocation tests; run staged pilots and rollback rehearsal.

## Recommended implementation sequence

1. Complete GH2-00 and freeze the verified 6A baseline.
2. Deliver GH2-01 and GH2-02 so already-ingested unlinked activity becomes useful before adding provider complexity.
3. Add read-only GitHub Actions visibility through GH2-03.
4. Build GH2-04 through GH2-06 behind a default-off entitlement and migrate a controlled pilot connection.
5. Release read-only installation/discovery broadly after security and operational sign-off.
6. Add GH2-07, then GH2-08, then GH2-09 as separate opt-ins; do not launch them as one indivisible permission bundle.
7. Complete GH2-10 and GH2-11 before general availability of provider writes.

## Acceptance criteria

- A project member with access can see authorized repository activity even when no task is linked, while another project or organization cannot infer it.
- An unlinked commit remains visible and can be attached to an existing task or used to prepare a task draft without forced automatic task creation.
- Workflow/check activity is represented by bounded summaries and GitHub links; Tailpoint retains no workflow logs or source content.
- An Owner can install the GitHub App, discover only authorized repositories, select repositories, and adopt an existing 6A connection without losing history.
- Read-only usage does not require write permissions; enabling a write feature clearly identifies and obtains only its additional permissions.
- Expired, revoked, suspended, or downgraded installations fail safely and present an actionable health state.
- Duplicate webhooks and retried outbound operations do not duplicate artifacts, links, issues, comments, checks, or status transitions.
- Status/comment synchronization cannot create an update loop and surfaces conflicts according to the configured source-of-truth policy.
- Rate-limit exhaustion cannot starve confirmed user writes behind optional enrichment, and support can diagnose quota and retry state without viewing tenant content.
- Removing an installation stops new access and synchronization while preserving authorized historical summaries according to retention policy.
- Staged rollout, kill switch, rollback, migration rehearsal, security review, and live pilot evidence are complete before general availability.

## Explicitly deferred beyond 6B

- GitLab installation and synchronization.
- Repository source browsing, code search, diffs, or workflow-log storage.
- Autonomous task creation from every commit or provider event.
- Unreviewed bidirectional mirroring of all task and GitHub comments.
- General release-management and incident-command state machines outside their dedicated Phase 6 workstreams.
- AI-generated code review, remediation, or release decisions.
