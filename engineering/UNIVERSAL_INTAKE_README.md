# Universal Intake

Universal Intake provides one governed pipeline for creating project tasks from API/SDK calls, CSV or Excel files, authenticated webhooks, and inbound email. It also records durable intake events and supports optional, review-only AI suggestions.

## Contents

- [How intake works](#how-intake-works)
- [Prerequisites](#prerequisites)
- [API and SDK intake](#api-and-sdk-intake)
- [CSV and Excel imports](#csv-and-excel-imports)
- [Authenticated webhooks](#authenticated-webhooks)
- [Email-to-task](#email-to-task)
- [Intake events and retries](#intake-events-and-retries)
- [AI intake suggestions](#ai-intake-suggestions)
- [Security and idempotency](#security-and-idempotency)
- [Deployment checklist](#deployment-checklist)

## How intake works

All channels converge on the same processing boundary:

```text
API / SDK / CSV / Excel / Webhook / Email
                      |
                      v
              Normalized intake event
                      |
          validation + idempotency + dedupe
                      |
             accepted / rejected / failed
                      |
                 Project task
                      |
          optional AI suggestion and review
```

The durable event is the authoritative record of what was received and what happened. Channel adapters authenticate and parse input, but task creation is performed by the normalized pipeline. Task creation and the accepted event-to-task link are committed together.

Event states are:

- `received`: stored but not yet processed.
- `validated`: validation completed.
- `accepted`: a task was created or an idempotent existing result was returned.
- `rejected`: input failed a business or validation rule.
- `quarantined`: content requires review, such as spam or an attachment.
- `failed`: an operational error occurred and the event may be retryable.

## Prerequisites

1. Enable the `universal_intake` capability for the organization.
2. Configure a default intake status and create the required channel source in the project's **Intake Operations** tab.
3. Enable `ai_assistance` separately if AI suggestions are required.
4. Use HTTPS for all public endpoints outside local development.

Authenticated workspace endpoints require:

```http
Authorization: Bearer <user-access-token>
x-organization-id: <organization-uuid>
```

Public channel endpoints use their own authentication described below and do not accept workspace identity from the sender.

## API and SDK intake

### Endpoint

```http
POST /v1/ingest/tasks
Authorization: Bearer trk_live_... # or trk_test_...
Content-Type: application/json
```

The API key determines the organization and destination project. Create and revoke project ingestion keys from the project's Intake Operations settings.

### Request

```json
{
  "source": "api",
  "title": "Checkout API is returning 500",
  "description": "Failures started after the latest deployment.",
  "severity": "high",
  "priority": 1,
  "dedupeKey": "checkout-api-500",
  "idempotencyKey": "delivery-01J8Y8M6A4",
  "occurredAt": "2026-08-11T01:30:00.000Z",
  "metadata": {
    "environment": "production"
  },
  "assigneeEmails": ["owner@example.com"],
  "customFields": [
    { "fieldId": "12", "value": "Payments" }
  ]
}
```

Supported `source` values are `api`, `sdk`, `sentry`, and `manual`. Supported severities are `low`, `medium`, `high`, and `critical`.

`idempotencyKey` identifies one delivery and should remain unchanged across transport retries. `dedupeKey` is a separate, optional business key used to group repeated occurrences into a task.

### Example

```bash
curl -X POST "$API_BASE_URL/v1/ingest/tasks" \
  -H "Authorization: Bearer $INGESTION_API_KEY" \
  -H "Content-Type: application/json" \
  --data '{
    "source": "api",
    "title": "Checkout API is returning 500",
    "severity": "high",
    "idempotencyKey": "delivery-01J8Y8M6A4"
  }'
```

New tasks return HTTP `201`; idempotent or deduplicated outcomes return HTTP `200`. Responses retain the original task result and include the durable intake event identifier.

The Tailpoint SDK uses this same endpoint. Configure `TAILPOINT_INGESTION_ENDPOINT` and `TAILPOINT_INGESTION_KEY`; SDK retries reuse one stable idempotency key.

## CSV and Excel imports

Use **Project → Intake Operations → Imports** for the normal workflow:

1. Download the CSV or Excel template.
2. Fill it in and upload it.
3. Review the detected headers and preview rows.
4. Map the title, description, severity, priority, dedupe key, assignees, and Custom Fields.
5. Remove unwanted staged rows.
6. Process the remaining rows.
7. Download the error CSV, correct failed rows, and retry without duplicating accepted rows.

Import constraints:

- CSV and first-sheet XLSX files are supported.
- Maximum file size is 10 MB.
- Maximum batch size is 5,000 rows.
- Every row has a stable identity and is validated independently.
- The assignee column accepts project-member email addresses.
- Accepted rows are skipped safely on retry.

### Import endpoints

```text
GET    /projects/:projectId/intake/imports
GET    /projects/:projectId/intake/imports/template?format=csv|xlsx
POST   /projects/:projectId/intake/imports/preview
GET    /projects/:projectId/intake/imports/:batchId
GET    /projects/:projectId/intake/imports/:batchId/rows
DELETE /projects/:projectId/intake/imports/:batchId/rows/:rowId
POST   /projects/:projectId/intake/imports/:batchId/process
GET    /projects/:projectId/intake/imports/:batchId/errors.csv
```

The preview upload uses multipart form data with a `file` field. Processing receives column names, not row values:

```json
{
  "title": "Title",
  "description": "Description",
  "severity": "Severity",
  "priority": "Priority",
  "dedupeKey": "Dedupe Key",
  "assignees": "Assignees",
  "customFields": [
    { "fieldId": "12", "column": "Team" }
  ]
}
```

## Authenticated webhooks

Create a named source under **Project → Intake Operations → Webhooks**. Creation returns:

- A public key used in the endpoint URL.
- A signing secret beginning with `whsec_`, displayed only once.
- A JSON mapping that uses safe dotted paths such as `incident.title`.

### Public endpoint

```http
POST /public/intake/webhooks/:publicKey
Content-Type: application/json
x-tailpoint-timestamp: <Unix seconds>
x-tailpoint-delivery: <unique provider delivery ID>
x-tailpoint-signature: sha256=<hex digest>
```

Sign the exact request bytes using:

```text
HMAC-SHA256(secret, timestamp + "." + rawRequestBody)
```

### Node.js example

```js
import crypto from "node:crypto";

const body = JSON.stringify({
  incident: {
    title: "Production API is returning 500",
    description: "The projects endpoint failed.",
    severity: "high",
    priority: 1
  }
});

const timestamp = Math.floor(Date.now() / 1000).toString();
const delivery = crypto.randomUUID();
const signature = crypto
  .createHmac("sha256", process.env.TAILPOINT_WEBHOOK_SECRET)
  .update(timestamp)
  .update(".")
  .update(body)
  .digest("hex");

const response = await fetch(
  `${process.env.API_BASE_URL}/public/intake/webhooks/${process.env.TAILPOINT_WEBHOOK_PUBLIC_KEY}`,
  {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "x-tailpoint-timestamp": timestamp,
      "x-tailpoint-delivery": delivery,
      "x-tailpoint-signature": `sha256=${signature}`
    },
    body
  }
);

console.log(await response.json());
```

The timestamp must be within five minutes. The delivery ID is the source-scoped idempotency key. Reusing it returns the original result rather than creating another task.

Example response:

```json
{
  "accepted": true,
  "eventId": "d621ee8d-a65a-4d5f-b429-ad7c2d3d5720",
  "taskId": 42,
  "idempotent": false
}
```

Webhook secrets are encrypted at rest. Set `WEBHOOK_SECRET_ENCRYPTION_KEY` to a stable, private production value. Rotating a secret can retain the previous secret for a bounded overlap window so senders can transition safely.

Management endpoints:

```text
GET   /projects/:projectId/intake/webhooks
POST  /projects/:projectId/intake/webhooks
PATCH /projects/:projectId/intake/webhooks/:sourceId
POST  /projects/:projectId/intake/webhooks/:sourceId/rotate-secret
```

## Email-to-task

Email intake currently uses SendGrid Inbound Parse. Create an address under **Project → Intake Operations → Email**. The generated opaque address routes messages to that project, for example:

```text
4f2c...@inbound.example.com
```

Configure:

```env
INBOUND_EMAIL_DOMAIN=inbound.example.com
SENDGRID_INBOUND_ACCESS_TOKEN=replace-with-a-long-random-token
```

Then configure DNS/MX and SendGrid Inbound Parse for that domain, posting multipart requests to:

```http
POST /public/intake/email/sendgrid
Authorization: Bearer <SENDGRID_INBOUND_ACCESS_TOKEN>
```

Processing rules:

- Recipient determines the project.
- Subject becomes the task title; an empty subject becomes `Email request`.
- Plain text or sanitized HTML becomes the description.
- SendGrid's parsed `Message-ID` is required and prevents duplicate tasks on retry.
- Sender information is attribution only and grants no workspace access.
- Messages over the address's spam threshold are quarantined and create no task.
- PNG, JPEG, PDF, TXT, CSV, and XLSX attachments are accepted for quarantine review.
- Up to 10 attachments of 10 MB each are accepted.
- Attachments are not automatically attached to the task until a scanning/release workflow is added.

Rotating an address creates a new recipient token immediately; the old address stops resolving. Revoking or disabling an address stops delivery without exposing whether the project exists.

Management endpoints:

```text
GET   /projects/:projectId/intake/email-addresses
POST  /projects/:projectId/intake/email-addresses
PATCH /projects/:projectId/intake/email-addresses/:id
POST  /projects/:projectId/intake/email-addresses/:id/rotate
```

## Intake events and retries

The Intake Operations event history is the main operational view. It shows channel, state, task, validation diagnostics, failure information, and processing attempts.

```text
GET  /projects/:projectId/intake/events
GET  /projects/:projectId/intake/events/:eventId
POST /projects/:projectId/intake/events/:eventId/retry
POST /projects/:projectId/intake/events/:eventId/reprocess
```

The list accepts `page`, `limit`, `state`, and `channel` query parameters.

- **Retry** is for retryable operational failures and preserves event identity and attempt history.
- **Reprocess** explicitly processes an otherwise terminal event again under current project rules.
- Replaying an accepted event returns its existing task.
- Rejected or quarantined events never create tasks unless explicitly and successfully reprocessed.

## AI intake suggestions

AI suggestions are optional and review-only. They run after an intake event has created a task; AI never creates, assigns, routes, or merges a task autonomously.

Both `universal_intake` and `ai_assistance` must be enabled. Configure one supported provider:

```env
AI_TEXT_PROVIDER=openai
OPENAI_API_KEY=...
OPENAI_AI_MODEL=gpt-5.6-sol
```

Alternatively, use `AI_TEXT_PROVIDER=huggingface` with the corresponding provider credentials and model configuration.

### Suggestion lifecycle

1. Generate a suggestion for an accepted event.
2. Review before/after values, reasons, and field confidence.
3. Select only the fields to apply, or dismiss the suggestion.
4. Confirm routing and duplicate merging separately when either is selected.
5. The backend revalidates the task, event fingerprint, destination, duplicate, assignee, permissions, and tenant ownership at apply time.

Suggestion states are `pending`, `applied`, `dismissed`, and `stale`.

### Endpoints

```text
GET  /ai/projects/:projectId/intake/events/:eventId/suggestions
POST /ai/projects/:projectId/intake/events/:eventId/suggestions/generate
POST /ai/projects/:projectId/intake/suggestions/:suggestionId/dismiss
POST /ai/projects/:projectId/intake/suggestions/:suggestionId/apply
```

Apply selected fields:

```json
{
  "fields": ["title", "priority", "assigneeId"],
  "confirmRouting": false,
  "confirmDuplicateMerge": false
}
```

Allowed fields are `title`, `category`, `priority`, `duplicateTaskId`, `assigneeId`, and `destinationProjectId`. Routing and duplicate merging require their corresponding confirmation flags.

The model sees only bounded, redacted intake content and authorized candidate labels/IDs. Candidate matching is restricted to projects, members, and tasks the reviewing actor may view. Provider output is treated as untrusted structured data and validated before it is stored or applied.

## Security and idempotency

- API keys, webhook secrets, provider tokens, authorization headers, and raw secrets are never stored in event snapshots.
- API/SDK idempotency uses the caller's stable `idempotencyKey`.
- Webhook idempotency uses `x-tailpoint-delivery`, scoped to its configured source.
- Email idempotency uses `Message-ID`, scoped to its inbound address.
- Import idempotency uses stable batch row identities.
- Idempotency prevents transport retries from creating duplicates.
- The optional task `dedupeKey` groups repeated business occurrences and is not a replacement for idempotency.
- Webhook signatures are checked against the exact raw body with a five-minute replay window.
- Public failures are deliberately neutral to avoid revealing projects or source configuration.
- Cross-organization and cross-project references are revalidated server-side.

## Deployment checklist

- [ ] Run all database migrations.
- [ ] Enable `universal_intake` for the pilot organization.
- [ ] Enable `ai_assistance` only for organizations using AI review.
- [ ] Configure each project's default intake status.
- [ ] Generate and securely store API keys and webhook secrets.
- [ ] Set a stable production `WEBHOOK_SECRET_ENCRYPTION_KEY`.
- [ ] Configure the AI provider, quotas, and provider credentials.
- [ ] Configure `INBOUND_EMAIL_DOMAIN`, DNS/MX, SendGrid Inbound Parse, and its bearer token.
- [ ] Add `INBOUND_EMAIL_DOMAIN` as a GitHub production environment variable.
- [ ] Add `SENDGRID_INBOUND_ACCESS_TOKEN` and `WEBHOOK_SECRET_ENCRYPTION_KEY` as GitHub production environment secrets.
- [ ] Configure those same values in the hosting provider's backend runtime environment; this repository's deploy hook does not transfer GitHub job environment variables to the host.
- [ ] Perform one signed webhook smoke test.
- [ ] Perform one live inbound-email smoke test and verify message idempotency.
- [ ] Upload a sample CSV/XLSX, remove a staged row, process it, and verify the error report.
- [ ] Verify accepted, rejected, quarantined, and failed events in Intake Operations.
- [ ] Generate, selectively apply, and dismiss an AI suggestion.

## Operational notes

- Universal Intake and AI Assistance are default-off pilot capabilities.
- Webhooks are limited to 60 requests per minute per public route policy.
- SendGrid inbound processing is limited to 30 requests per minute per public route policy.
- API ingestion uses `RATE_LIMIT_INGESTION_MAX`, `RATE_LIMIT_INGESTION_WINDOW_MS`, and `INGESTION_MAX_BODY_KB`.
- A live provider/DNS email test is required in each deployed environment even though the application-level email adapter is covered by automated tests.
