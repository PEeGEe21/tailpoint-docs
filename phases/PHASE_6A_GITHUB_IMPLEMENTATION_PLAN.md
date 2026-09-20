# Phase 6A GitHub Integration Implementation Plan

For setup and day-to-day use, see the [GitHub integration usage guide](../operations/GITHUB_INTEGRATION_GUIDE.md).

## Implementation status

**Validation (2026-09-19).** GH-01 through GH-07 are implemented across the backend, workspace, and admin surfaces. The capability is default-off; persistence, multi-repository connection management, one-time and overlapping secret rotation, verified raw-body webhook receipt, durable claim/fast acknowledgement, queued processing, stable repository-ID rename reconciliation, concurrency-safe delivery claims, failed-delivery retry recovery, five bounded event normalizers, provider-timestamp ordering, project-scoped task references, provenance and precedence, durable manual overrides, safe pull-request inheritance, unresolved-reference diagnostics, link cleanup, archived history, silence/failure health, Owner alerts, project setup/health UX, lifecycle activity/audit records, safe aggregate support health, and OpenAPI contracts are present. Backend and workspace production builds, focused GitHub tests, type checks, OpenAPI checks, and the local hardening migration pass. Release still requires a live GitHub repository pilot.

## Delivery decision

Phase 6A delivers a project-scoped, one-way GitHub integration while Phase 3 remains under production observation. The MVP uses verified repository webhooks and the existing entitlement, authorization, ingestion, activity, audit, automation, and notification boundaries. It does not require GitHub OAuth, a GitHub App installation, or write access to GitHub.

## MVP experience

A project Owner may create any number of GitHub repository connections and receives a one-time webhook URL and secret for each. After configuring those webhooks in GitHub, Tailpoint receives supported repository activity and shows linked issues, pull requests, commits, deployments, and releases on referenced tasks. One task can accumulate development activity from multiple connected repositories. Delivery health, silence, and rejected/failed events are visible without exposing webhook secrets or unrestricted payload contents.

## Authorization and security

- The `github_integration` capability is default-off and enforced by the backend.
- Project Owners may create, rotate, disable, and remove connections. Project members with task view access may see safe linked-development summaries.
- A connection belongs to one organization, one Tailpoint project, and one stable GitHub repository ID. The signed `owner/name` is reconciled after a repository rename or transfer.
- Webhooks require `X-Hub-Signature-256`; comparison is constant-time and uses the exact raw request bytes.
- `X-GitHub-Delivery` is required and unique per connection. Retries return the stored outcome and never duplicate links or activity.
- Secrets are encrypted at rest, shown only once, never returned by list/detail APIs, and excluded from logs and audit payloads.
- Payloads are size-bounded. Only allowlisted event fields are persisted; full webhook bodies, access tokens, comments, diffs, and file contents are not retained.
- The public receiver is rate-limited by connection and source address. Optional `GITHUB_WEBHOOK_IP_ALLOWLIST` CIDRs may mirror GitHub's published `hooks` ranges as defense-in-depth; HMAC remains authoritative. Behind Render or another trusted reverse proxy, set `HTTP_TRUST_PROXY_HOPS` correctly before enabling the allowlist.
- Signed repository IDs prevent an unrelated repository from taking over a bound connection. A first signed delivery binds the stable ID; later name changes are reconciled and reported to Owners.

## Supported events and linking

- `issues`: opened, edited, closed, reopened, and labeled.
- `pull_request`: opened, synchronize, ready-for-review, closed, and reopened.
- `push`: bounded commit summaries from the pushed range.
- `deployment_status`: environment and deployment state.
- `release`: published, released, edited, and deleted.
- Task references use an explicit, case-insensitive `TP-<taskId>` token in supported title/body/branch/commit-message fields. A task is linked only when it belongs to the configured Tailpoint project.
- `TP-<taskId>` is the deterministic automatic shortcut, not the only linking path. Editors and Owners can paste the URL of a received artifact to link it manually, move it between tasks in the same project, or suppress an incorrect association. The task UI provides an editable branch-name generator with `feature`, `fix`, `chore`, `refactor`, `docs`, `test`, and `hotfix` types, a task-title slug, and a browser-local type preference so contributors do not need to memorize task IDs.
- Association precedence is manual override, direct reference on the artifact, then inherited pull-request association. A commit with any direct reference never inherits its pull request's tasks; invalid direct references remain unresolved instead of silently falling back. A reference may intentionally identify multiple tasks.
- Every association stores durable provenance (`manual`, source field, or `inherited_pull_request`) and the bounded source token. Manual unlinking is a retained suppression record rather than deletion, so replayed or later deliveries cannot recreate a rejected link. Moving a link suppresses the old pair and creates the new manual pair atomically.
- Commits without direct references may inherit from exactly one known open pull request whose head branch matches the push branch. Ambiguous branch matches remain unlinked and produce a diagnostic; the MVP does not make GitHub API enrichment a hidden dependency.
- Invalid, out-of-project, and ambiguous references create bounded Owner-visible diagnostics. Cross-project diagnostics never disclose whether the ID exists elsewhere or reveal another project's metadata.
- Matching runs independently across every active repository connection in the project. One task may therefore show artifacts from several repositories.
- One provider object has one stable identity per repository and may link to multiple referenced tasks. An artifact update is applied only when its provider timestamp is newer than the stored timestamp; stale deliveries remain recorded but cannot overwrite state or link history.
- A newer editable artifact synchronizes its task references, task deletion cascades its links, out-of-project links are hidden, and Owners can explicitly unlink immutable artifacts when a reference was a typo.

## Persistence

- `github_connections`: tenant/project/repository identity, encrypted current/previous secrets, rotation overlap, active/archive state, creator, timestamps, and delivery/silence health. Uniqueness is `(project, repository)`, not project alone; a project has no Phase 6A repository-count cap.
- `github_deliveries`: connection, provider delivery ID, event/action, state, safe failure code, received/processed timestamps, and bounded counters.
- `github_artifacts`: connection, provider type/stable ID, number/SHA, title, state, URL, actor label, safe metadata, provider timestamps, and last delivery.
- `github_task_links`: artifact/task identity, first/last delivery, timestamps, and a unique artifact/task constraint. Connection removal through the product/API is archival: artifacts and links remain readable and no new deliveries are accepted. Full project deletion retains its existing cascade semantics.

## API and product surface

- Authenticated project APIs: list all repository connections for a project, create/detail/disable/archive a connection, rotate secret, list delivery health, list task development links, and explicitly unlink an artifact.
- Public webhook endpoint: rate limit, optional source-CIDR check, raw-body verification, stable repository reconciliation, durable delivery claim, `202` acknowledgement, bounded normalization, queued idempotent artifact/link processing, and safe response.
- Workspace project settings: connection setup instructions, one-time secret handling, repository/status display, rotation/disable controls, and recent delivery health.
- Task detail: a Development section grouped by pull requests, issues, commits, deployments, and releases with safe external links.
- Admin: existing entitlement override controls plus tenant-safe connection/delivery counts; no repository URLs, secrets, payloads, or task content.

## Delivery tickets

1. **GH-01 — Capability and contracts:** register `github_integration`; finalize event, reference, redaction, authorization, and rotation contracts. **Status: complete.**
2. **GH-02 — Persistence and migration:** add four entities, uniqueness/index constraints, encrypted secrets, and reversible migration coverage. **Status: complete.**
3. **GH-03 — Connection management API:** implement project-owner CRUD/rotation and member-safe reads with activity and audit capture. **Status: complete.**
4. **GH-04 — Verified webhook receiver:** preserve raw bytes, rate-limit ingress, optionally check source CIDRs, validate headers/signature/repository ID, claim deliveries idempotently, acknowledge with `202`, and process through the Redis-backed queue (with non-production in-process fallback). **Status: complete.**
5. **GH-05 — Normalization and task linking:** support the five MVP event families, bounded fields, explicit task references across all project connections, provider-timestamp ordering, project validation, idempotent links, and cleanup/unlink behavior. **Status: complete.**
6. **GH-06 — Workspace surfaces:** implement project setup/health and task Development UI with loading, empty, error, retry, disabled, and secret-copy states. **Status: complete for validation.**
7. **GH-07 — Operations and validation:** add safe admin health counts, hourly silence and rotation checks, repeated-failure Owner alerts, permission/tenant/signature/idempotency/ordering tests, migration rehearsal, typechecks/builds, a signed five-event pilot harness, and pilot rollout. **Status: validation; automated gates and migration rehearsal pass, live repository pilot pending.**

## Acceptance criteria

- Invalid signatures, missing delivery IDs, oversized payloads, and repository mismatches are rejected without writes or information leakage.
- Concurrent or retried delivery IDs create one delivery outcome and no duplicate artifacts, task links, activity, or automation events.
- The receiver durably claims and queues a valid delivery before returning `202`; artifact processing never extends the public request lifetime.
- A delayed delivery cannot overwrite artifact state written from a newer provider timestamp.
- A signed delivery for the same stable repository ID reconciles a rename/transfer instead of failing on the old `owner/name`.
- Cross-project task references are ignored and never reveal whether the referenced task exists.
- Disabling the connection or capability rejects new webhook processing while preserving authorized history.
- Archiving a connection retains read-only artifact/task history. Task deletion removes links; newer edited references and explicit Owner unlinking clean up mistaken links.
- Secret rotation supports a bounded overlap period and neither secret appears in reads, audit, logs, or operational records.
- Members see development links only through tasks they may currently view; only Owners manage connections.
- GitHub outages cannot block normal Tailpoint task mutations because 6A performs no synchronous outbound GitHub calls.
- A task can display linked development activity from more than one repository connection within the same project.
- Silent connections, repeated processing failures, and expired rotation overlap produce visible health state and deduplicated Owner notifications.

## Pilot harness

Use the generated connection URL and one-time secret to exercise all five event schemas without waiting for live repository activity:

```bash
npm run github:pilot -- \
  --url https://api.example.com/api/github/webhooks/<key> \
  --secret '<one-time-secret>' \
  --repository owner/repository \
  --repository-id 123456789 \
  --task-id 123
```

Add `--dry-run` to validate arguments and payload generation without sending. The harness signs the exact JSON bytes and uses unique delivery IDs. It creates synthetic development artifacts, so use a pilot project/task.

## Deferred beyond 6A

- GitHub App/OAuth installation and automatic repository discovery.
- GitLab support.
- Two-way issue synchronization, GitHub mutations, comments, checks, and status writes.
- Automatic task creation from arbitrary provider events.
- Backfill of historical repository activity.
- Release-management and incident-workflow state machines.
