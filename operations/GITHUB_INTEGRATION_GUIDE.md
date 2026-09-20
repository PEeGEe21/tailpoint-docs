# Using the Tailpoint GitHub Integration

**Status:** Phase 6A validation guide
**Last updated:** 2026-09-19

This guide explains how project Owners connect GitHub repositories to Tailpoint and how project members link GitHub activity to Tailpoint tasks. The Phase 6A integration is one-way: GitHub sends repository activity to Tailpoint, but Tailpoint does not create or modify anything in GitHub.

## What the integration does

After a repository is connected, Tailpoint can display these GitHub artifacts in a task's **Development** section:

- Issues
- Pull requests
- Commits from push events
- Deployment statuses
- Releases

An artifact is linked when its supported text contains the Tailpoint task reference `TP-<taskId>`. For example, task 123 is referenced as `TP-123`. Matching is case-insensitive, so `tp-123` also works.

This token is a fast automatic path, not a requirement for every commit. From a Tailpoint task, contributors can generate and edit a ready-to-use branch name, choose `feature`, `fix`, `chore`, `refactor`, `docs`, `test`, or `hotfix`, and copy the result. Generated slugs remove common HTTP/error noise and stop at a short word boundary; the result remains editable. Tailpoint remembers the contributor's most recently selected type in that browser. Editors or Owners can also paste the URL of an artifact Tailpoint has already received. A pull request association is inherited by commits pushed to its known head branch only when the commit has no direct task reference.

Association precedence is: manual choice, direct reference on the artifact, then pull-request inheritance. For example, if a pull request is linked to Task 123 but one of its commits says `TP-456`, that commit links to Task 456. A malformed, unknown, or out-of-project direct reference is left unresolved rather than silently inheriting another task.

A project can connect multiple repositories. A single task can show activity from any of those repositories, which supports frontend/backend splits and other multi-repository projects.

## Before you begin

You need:

- Project Owner access in Tailpoint.
- Repository administration permission in GitHub so you can create a webhook.
- The `github_integration` capability enabled for the Tailpoint workspace.
- A secure place to copy the one-time webhook secret while completing setup.

If the GitHub section is not visible in project settings, ask a Tailpoint administrator to enable the capability for the workspace.

## Connect a repository

1. Open the project in Tailpoint.
2. Open the project's settings and find **GitHub repositories**.
3. Enter the repository as `owner/repository`, for example `acme/payments-api`.
4. Select **Connect repository**.
5. Keep the displayed payload URL and secret open. The secret is shown only once.
6. In GitHub, open the repository and go to **Settings → Webhooks → Add webhook**.
7. Enter the Tailpoint payload URL in **Payload URL**.
8. Set **Content type** to `application/json`.
9. Enter the one-time Tailpoint secret in **Secret**.
10. Keep SSL verification enabled.
11. Choose individual events and enable:
    - Issues
    - Pull requests
    - Pushes
    - Deployment statuses
    - Releases
12. Make sure the webhook is active, then select **Add webhook**.

GitHub normally sends a ping after creation. Supported activity will appear as delivery health in Tailpoint once it is received and processed.

Repeat these steps for every repository that should contribute development activity to the project. Each connection has its own payload URL and secret.

## Link GitHub activity to a task

Include `TP-<taskId>` in one of the supported fields:

| GitHub activity | Fields Tailpoint examines |
| --- | --- |
| Issue | Title and body |
| Pull request | Title, body, and source branch name |
| Push/commit | Commit message |
| Deployment | Description, ref, and environment |
| Release | Name, body, and tag |

Examples:

```text
Fix invoice rounding for TP-123
```

```text
feature/TP-123-invoice-rounding
```

```text
Deploy TP-123 to production
```

The referenced task must belong to the same Tailpoint project as the repository connection. References to tasks in another project or workspace are ignored.

Open the task in Tailpoint and look for **Development**. Linked entries show the artifact type, number or SHA, title/state, repository name, and a safe link back to GitHub.

## Correct or remove a link

For editable GitHub artifacts such as issues and pull requests, remove or correct the `TP-<taskId>` reference in GitHub. When GitHub sends the newer update, Tailpoint synchronizes the links from that artifact.

Commit messages are immutable. A project Owner can use the explicit unlink operation when a commit or another immutable artifact contains a mistaken task reference. Deleting the Tailpoint task also removes its development links automatically.

Unlinking is a durable manual suppression: later webhook delivery or replay cannot silently restore that artifact/task pair. Moving development activity to another task suppresses the old association and creates the replacement together. The Development section shows provenance such as **Manually linked**, **From PR title**, **From commit message**, or **Inherited from pull request**.

Owners can review unresolved-reference diagnostics in GitHub repository settings. A diagnostic for an out-of-project reference deliberately says only that the token does not identify a task in the connected project; it does not reveal data from another project or workspace.

If a task is moved outside the connected project, its old project links are not displayed. Owners can explicitly unlink any obsolete artifact association.

## Understand connection health

Tailpoint shows the most recent delivery and a connection health state:

| State | Meaning |
| --- | --- |
| `pending` | Waiting for the first delivery. |
| `processing` | A delivery was accepted and queued. |
| `healthy` | The latest processed delivery completed successfully. |
| `failing` | Delivery processing has failed and may be retrying. |
| `silent` | No delivery has arrived within the configured silence window. |
| `disabled` | The Owner disabled the connection. |
| `archived` | The connection was removed; existing history is read-only. |

All project Owners—the permanent project creator plus every confirmed co-owner—receive notifications for repeated processing failures, unexpected delivery silence, repository renames, and an expired secret-rotation overlap that was not confirmed with the new secret.

Silence does not always mean the webhook is broken—a quiet repository may have no supported activity. Check GitHub's webhook delivery history before recreating the connection.

## Disable or remove a connection

- **Disable** temporarily rejects new deliveries while keeping the connection available to enable later.
- **Remove** archives the connection. It no longer accepts deliveries and disappears from the active connection list, but existing task Development history remains visible as archived/read-only.

Neither operation deletes GitHub artifacts or changes the repository.

## Rotate a webhook secret

1. In Tailpoint project settings, select **Rotate** for the repository.
2. Copy the new secret immediately; it is shown only once.
3. In GitHub, open **Settings → Webhooks** and edit the matching Tailpoint webhook.
4. Replace the old secret with the new secret and save the webhook.
5. Use GitHub's **Recent Deliveries → Redeliver** action or generate supported activity to confirm the new secret works.

Tailpoint accepts the previous secret for a limited overlap period, currently 60 minutes through the workspace UI. Once a delivery signed with the new secret is received, Tailpoint records the rotation as confirmed. If the overlap expires without confirmation, the project Owner is notified.

## Repository renames and transfers

Tailpoint binds a connection to GitHub's stable repository ID after the first signed delivery. If the repository is renamed or transferred, a later signed delivery updates the displayed `owner/repository` automatically and notifies project Owners.

Do not create a second connection solely because a repository was renamed. If the destination repository is already connected to the same Tailpoint project, Tailpoint rejects the conflicting rename and an Owner must resolve the duplicate connection.

## Troubleshooting

### No Development section or GitHub settings

- Confirm the workspace has `github_integration` enabled.
- Confirm you can view the task.
- Only project Owners can manage repository connections.

### Waiting for first delivery

- Confirm the GitHub webhook is active.
- Confirm the payload URL exactly matches the value generated by Tailpoint.
- Confirm the content type is `application/json`.
- Check GitHub **Recent Deliveries** for the ping and subsequent events.
- Generate one of the five supported event types; unsupported events are accepted safely but create no artifact.

### GitHub shows `401 Unauthorized`

- The configured secret does not match Tailpoint.
- Rotate the secret in Tailpoint, update it in GitHub, and redeliver an event during the overlap period.
- Ensure no proxy or middleware rewrites the request body, because the signature covers the exact JSON bytes.

### GitHub shows `404 Not Found`

- The connection may be disabled or archived.
- The workspace capability may have been disabled.
- Confirm the complete payload URL, including the webhook key.

### GitHub shows `429 Too Many Requests`

The connection/source rate limit was exceeded. Wait for the rate-limit window to reset and investigate repeated redeliveries or automation loops.

### GitHub shows `503 Service Unavailable`

Tailpoint could not durably queue the delivery. GitHub should retry it. Platform operators should check Redis and the GitHub delivery worker before manually redelivering.

### Delivery is processed but the task has no link

- Confirm the reference uses the complete token, such as `TP-123`.
- Confirm the task belongs to the same Tailpoint project as the connected repository.
- Put the reference in a supported field from the table above.
- Confirm the event action is supported. For example, arbitrary issue or pull-request actions may be safely ignored.

### Repository was renamed and deliveries fail

Check whether another connection in the project already uses the new repository identity. Otherwise, a correctly signed delivery with the same stable GitHub repository ID should reconcile the name automatically.

## Validate with synthetic pilot events

Platform operators can send correctly signed synthetic versions of all five supported events from the backend repository:

```bash
npm run github:pilot -- \
  --url https://api.example.com/api/github/webhooks/<key> \
  --secret '<one-time-secret>' \
  --repository owner/repository \
  --repository-id 123456789 \
  --task-id 123
```

Add `--dry-run` to validate arguments and payload generation without sending requests. A live run creates synthetic Development entries, so use a dedicated pilot project and task.

## Platform configuration

Production requires Redis-backed queueing and should use Redis-backed rate limiting:

```env
REDIS_ENABLED=true
QUEUE_DRIVER=redis
RATE_LIMIT_DRIVER=redis
GITHUB_FAILURE_ALERT_THRESHOLD=3
GITHUB_SILENCE_ALERT_HOURS=168
```

For optional defense-in-depth, populate `GITHUB_WEBHOOK_IP_ALLOWLIST` with the current comma-separated CIDRs from GitHub's published `hooks` ranges. HMAC verification remains mandatory. When the API is behind Render or another trusted reverse proxy, set `HTTP_TRUST_PROXY_HOPS` to the exact trusted proxy depth before enabling IP allowlisting.

## Security and privacy notes

- Webhook secrets are encrypted at rest and never returned after their one-time display.
- Tailpoint verifies `X-Hub-Signature-256` against the exact raw request body.
- Full webhook bodies, diffs, comments, file contents, and GitHub access tokens are not retained.
- Only bounded artifact summaries and extracted task IDs enter the delivery queue.
- Duplicate delivery IDs are idempotent.
- Delayed events are recorded but cannot overwrite state from a newer provider timestamp.
- Tailpoint makes no synchronous outbound call to GitHub and receives no write access to the repository.

## Current Phase 6A limitations

- No GitHub App or OAuth installation flow.
- No automatic repository discovery.
- No historical activity backfill.
- No two-way issue synchronization or GitHub writes.
- No GitLab support yet.

For implementation and acceptance details, see the [Phase 6A GitHub implementation plan](../phases/PHASE_6A_GITHUB_IMPLEMENTATION_PLAN.md).
