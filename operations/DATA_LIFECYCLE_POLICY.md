# Baseline Data Lifecycle Policy

This policy defines the Phase 1 lifecycle contract for sensitive records. Product features may apply a shorter organization policy later, but may not retain content beyond these defaults.

| Record type | Classification | Default retention | Export authority | Deletion authority |
| --- | --- | --- | --- | --- |
| Decisions and content-bearing history | Sensitive workspace record | Retained with its project until an authorized hard deletion | Any project member who may view the record | Project Editor or Owner |
| Note audio and transcripts | Sensitive user recording | Retained until the owning user deletes the audio or note | Owning user | Owning user |
| Lifecycle events | Content-free governance metadata | Retained with the organization | Same authority as the source record | Removed only with organization deletion |

## Deletion contract

- Decision deletion is a hard delete. Links and content-bearing decision-history snapshots cascade with the source record. The preserved lifecycle event contains only identifiers, status, actor, time, and deletion method.
- Note-audio deletion removes the object-storage source first, then clears its URL, path, MIME type, duration, transcript, transcript status, and consent fields. Deleting the note performs the same source-file cleanup.
- Search currently reads decisions and transcripts from their source tables, so hard deletion or transcript clearing removes the content from search. AI request audits never contain raw content, and generated drafts are sent with provider storage disabled.
- The current providers do not expose a separate persisted-artifact identifier. If a future provider does, its adapter must delete that artifact before lifecycle deletion reports completion.

## Preservation and expiry

- There is no legal-hold feature in the baseline release. Do not claim that a record is legally preserved. Legal hold, configurable retention schedules, and disposition approval remain enterprise work.
- Application deletion is immediate. Infrastructure backups expire according to the operator's backup schedule and must not be restored as live content except during disaster recovery. Restored deletions must be replayed from content-free lifecycle events.
- Recording requires affirmative consent plus a notice-version identifier before upload or transcription. Consent, export, deletion, and access-history actions create content-free lifecycle events.
