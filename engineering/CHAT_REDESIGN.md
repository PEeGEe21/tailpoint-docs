# Chat Redesign PRD

## Overview

This document defines the product and engineering requirements for the chat module redesign across `track-a-project` and `track-a-project-backend`.

The redesign is not just a UI refresh. It is a full chat-system upgrade covering:

- messaging reliability
- mobile responsiveness
- realtime presence
- attachments
- reactions
- conversation management
- profile and shared context
- read state accuracy
- call architecture

The goal is to build one coherent chat system, not a collection of partial improvements.

## Objective

Deliver a production-ready chat experience that:

- works well on mobile, tablet, and desktop
- feels fast because sending is optimistic
- remains correct under reconnects and multi-device usage
- supports modern messaging expectations like attachments, reactions, pinning, and archiving
- exposes real contextual information like profile, about, and shared projects
- creates a clean foundation for voice and video calling

## Current State

### Existing frontend surface

- `track-a-project/app/(dashboard)/chat/page.tsx`
- `track-a-project/app/actions/message.ts`
- `track-a-project/app/utils/sockets/messagesSocket.ts`
- `track-a-project/app/(dashboard)/chat/components/VideoCallModal.tsx`
- `track-a-project/app/(dashboard)/chat/components/IncomingCallModal.tsx`

### Existing backend surface

- `track-a-project-backend/src/messages/controllers/messages.controller.ts`
- `track-a-project-backend/src/messages/services/messages.service.ts`
- `track-a-project-backend/src/messages/messages.gateway.ts`
- `track-a-project-backend/src/typeorm/entities/Message.ts`
- `track-a-project-backend/src/typeorm/entities/Conversation.ts`
- `track-a-project-backend/src/typeorm/entities/ConversationParticipant.ts`
- `track-a-project-backend/src/typeorm/entities/MessageReaction.ts`
- `track-a-project-backend/src/typeorm/entities/MessageReadReceipt.ts`

### What already works

- direct conversation list
- direct conversation creation
- basic message fetch
- basic text send
- websocket broadcast for new messages
- typing events
- basic read receipt event broadcast
- placeholder profile sidebar

### What is incomplete

- attachments are modeled but not wired
- reactions exist in schema but not in chat flow
- unread counts are not correctly implemented
- online status depends on database fields and in-memory socket state
- profile/about/shared-projects are placeholder UI
- pin/archive/delete/star are not backed by participant-level persistence
- call UI exists, but backend call signaling does not
- mobile behavior is not treated as a primary layout requirement

## Product Principles

### 1. Mobile-first, not desktop-only

The chat module must be fully usable on small screens. Mobile responsiveness is a launch requirement.

### 2. Fast but correct

Chat should feel immediate through optimistic updates, but state must remain correct after reconnects, retries, refreshes, and multi-device use.

### 3. Participant-scoped behavior

Features like pinning, archiving, muting, deleting, and unread state belong to the participant layer, not the global conversation layer.

### 4. One system, not optional extras

Unread state, presence, retry behavior, pagination, search shape, and moderation constraints must be built into the core implementation of related features.

### 5. Schema before polish

We should not keep layering UI on top of incomplete data contracts. Message and conversation contracts need to be redesigned first.

## In Scope

### Messaging foundation

- optimistic sending
- retry failed messages
- unified message contract
- accurate delivery and read states
- typing indicator polish
- reverse infinite scroll and pagination

### Message features

- attachments
- reply-to support
- reactions
- starring messages
- edit and soft delete readiness

### Conversation features

- pin conversation
- archive conversation
- delete or hide conversation for a participant
- mute conversation
- mark unread readiness
- search and filtering readiness

### Presence and realtime

- Redis-backed online presence
- last seen fallback
- reconnect and resubscribe behavior
- multi-device awareness

### Context and profile

- real profile drawer
- about section
- shared projects
- workspace and role context

### Calls

- call signaling architecture
- video call completion
- voice-only call mode
- call lifecycle states

### UX and layout

- mobile responsiveness
- tablet responsiveness
- desktop responsiveness
- keyboard-safe composer behavior
- touch-friendly action surfaces

## Out of Scope For First Build

These should be schema-aware but do not need to fully ship in the first implementation pass unless we explicitly pull them in:

- full group-chat product flow
- threaded replies UI beyond basic reply-to
- advanced moderation/reporting UI
- attachment antivirus pipeline if infra is not ready yet

## User Problems To Solve

1. Sending a message feels slower than it should because the UI waits for server completion.
2. Online status cannot be trusted as fully realtime.
3. Mobile chat is not yet a first-class experience.
4. Users cannot send files even though the model suggests support exists.
5. The profile panel does not show real useful chat context.
6. The system lacks expected chat behaviors like reactions, pinning, archiving, and starred messages.
7. Read state and unread counts are not reliable.
8. Call-related UI exists without a full working backend path.

## Functional Requirements

## FR1. Responsive Layout

The chat module must support three primary layouts:

- desktop: conversation list, thread, and optional profile panel
- tablet: conversation list and thread, with profile in slide-over
- mobile: list view and thread view as separate states with clear back navigation

Requirements:

- the composer must remain usable when the mobile keyboard is open
- the header and composer must stay visible without breaking scroll
- action menus must remain touch-friendly
- the profile/about/shared-projects panel must become a drawer or bottom sheet on smaller screens

Acceptance criteria:

- chat is fully usable at common mobile widths without horizontal overflow
- users can navigate from list to thread and back on mobile
- typing and sending remain usable with the on-screen keyboard open
- attachment, emoji, and more-actions controls remain accessible on touch devices

## FR2. Optimistic Sending And Retry

Messages must appear immediately in the UI before server confirmation.

Requirements:

- each outgoing message includes a `clientMessageId`
- local pending messages render instantly
- failed sends transition to a failed state
- failed sends can be retried without creating duplicates
- server acknowledgement reconciles pending messages with persisted messages

Acceptance criteria:

- a sent message appears in the thread immediately
- network delay does not block local rendering
- failed messages can be retried inline
- duplicate messages are not created when retrying or reconnecting

## FR3. Attachments

Users must be able to send attachments in chat.

Requirements:

- support file selection from the composer
- upload flow must be separate from plain message create
- attachment preview must appear before send
- attachment messages must render appropriately in-thread
- upload failures must be recoverable

Acceptance criteria:

- a user can attach and send a file from the chat composer
- image attachments render as previews where appropriate
- non-image files render as downloadable attachment cards
- upload failure shows a recoverable error state

## FR4. Presence And Online Status

Presence must be powered by Redis rather than database `logged_in` state.

Requirements:

- websocket registration updates Redis-backed presence
- disconnect and heartbeat expiry clear presence
- online state is queryable across instances
- offline users may display last seen information where available

Acceptance criteria:

- online status updates without requiring a page refresh
- presence remains correct across reconnects
- the system does not depend on `logged_in` as the primary chat presence source

## FR5. Read State And Unread Counts

The system must support reliable unread state.

Requirements:

- participant-level read markers must be stored
- unread counts must be computed server-side
- opening a conversation should be able to mark visible messages as read
- read receipts must integrate with persisted state, not only socket events

Acceptance criteria:

- unread badge counts change correctly as messages are received and read
- reopening chat after refresh preserves read state
- read indicators do not rely only on temporary socket events

## FR6. Reactions

Users must be able to react to messages.

Requirements:

- add reaction
- remove reaction
- aggregate reaction counts
- show whether the current user reacted with a given emoji
- sync reaction changes in realtime

Acceptance criteria:

- a user can add a reaction to a message
- reacting again with the same emoji toggles or updates according to final product rule
- other participants see reaction updates without refresh

## FR7. Starred Messages

Users must be able to star important messages.

Requirements:

- star and unstar a message
- retrieve starred messages within a conversation
- support jump-to-message behavior

Acceptance criteria:

- a user can star a message
- starred messages persist across refresh
- starred messages can be listed and navigated back to

## FR8. Conversation Management

Users must be able to manage conversations at the participant level.

Requirements:

- pin conversation
- archive conversation
- hide or delete conversation for the participant
- mute conversation readiness

Acceptance criteria:

- pinning affects ordering for that participant only
- archiving removes a conversation from the default active list
- deleting or hiding a conversation does not globally destroy it for other participants unless explicitly intended

## FR9. Profile, About, And Shared Projects

The profile panel must show real chat context.

Requirements:

- real peer profile payload
- about text or bio
- shared project list between both participants
- quick navigation into shared projects
- organization and role context where relevant

Acceptance criteria:

- opening the info panel shows real peer data
- shared projects reflect actual backend relationships
- the panel is useful on both desktop and mobile

## FR10. Emoji Picker Behavior

The emoji picker must not close unexpectedly while the user is still composing.

Requirements:

- selecting an emoji should preserve input focus
- picker close behavior should be explicit or outside-click driven
- repeated emoji insertion should be smooth

Acceptance criteria:

- a user can add multiple emojis without the picker collapsing unexpectedly
- the cursor returns to the expected place in the composer

## FR11. Calls

The chat system must support a real call flow foundation.

Requirements:

- backend signaling for call start, accept, reject, end, and timeout
- conversation membership checks for call actions
- voice-only and video call modes
- missed and rejected state handling
- call UI must be responsive

Acceptance criteria:

- initiating a call triggers a working signaling flow
- recipients can accept or reject
- ended and missed call states are handled consistently
- call UI remains usable on small screens

## FR12. Pagination And Long Conversation Handling

Conversation history must scale.

Requirements:

- fetch newest chunk first
- support loading older messages
- preserve scroll position when loading history
- avoid rendering the full history at once

Acceptance criteria:

- large conversations remain usable
- loading older messages does not jump the viewport unexpectedly
- the frontend does not require the entire history to render a conversation

## Non-Functional Requirements

### Performance

- sends should feel immediate because of optimistic rendering
- long conversations should remain smooth to scroll
- attachment upload feedback should be visible

### Reliability

- reconnect should restore socket registration and conversation joins
- retries should not duplicate messages
- presence should survive multi-instance backend deployment

### Security And Validation

- message create payloads must be validated
- attachment types must be restricted
- participants must be authorized before message and call operations

### Maintainability

- fetch and socket message payloads should share one normalized shape
- participant-level preferences should avoid ad hoc workarounds
- direct chat implementation should not block future group chat support

## Data Model Requirements

### Message contract

Messages should converge on a normalized shape including:

- `id`
- `clientMessageId`
- `conversationId`
- `content`
- `messageType`
- `attachments`
- `replyTo`
- `sender`
- `createdAt`
- `updatedAt`
- `deliveryStatus`
- `reactions`
- `readBy`

### Participant-level conversation state

Conversation participant state should support:

- `isPinned`
- `pinnedAt`
- `isArchived`
- `archivedAt`
- `isDeleted`
- `deletedAt`
- `lastReadMessageId` or `lastReadAt`
- `draft`
- `isMuted`

### Presence model

Redis presence should support:

- per-user online state
- socket count or active session count
- TTL-based expiry
- optional last-seen metadata

## API And Transport Requirements

### REST responsibilities

- fetch conversations
- fetch paginated messages
- create message
- upload attachment metadata or upload target
- manage conversation preferences
- fetch profile/about/shared projects
- fetch starred messages

### WebSocket responsibilities

- register user presence
- join and leave conversation rooms
- new message events
- reaction updates
- typing updates
- read receipt updates
- call signaling events

### Redis responsibilities

- shared presence
- distributed realtime coordination where needed
- call/session helpers if required for multi-node behavior

## Dependencies Between Features

- unread correctness must be built together with read markers and optimistic sending
- socket resilience must be built together with presence
- attachments must be built together with validation and rendering rules
- pin/archive/delete must be built together with participant preference persistence
- mobile responsiveness must be built together with the component rewrite, not after
- calls must be built on top of a real signaling contract, not only frontend modal work

## Delivery Plan

## Phase 1. Foundation

- [Done] redesign message and conversation contracts
- [Done] add DTO validation
- [Done] add participant-level conversation state fields
- [Done] implement Redis-backed presence
- [Done] implement optimistic sending with `clientMessageId`
- [Done] implement retry and reconciliation rules
- [Done] implement responsive layout architecture
- [Done] harden reconnect and resubscribe behavior

Exit criteria:

- sending is optimistic and safe
- presence no longer depends on DB as primary realtime source
- mobile and desktop layout structure is established
- core contracts are stable enough for follow-on feature work

## Phase 2. Core Chat Features

- [Done] implement attachments end-to-end
- [Done] implement reactions
- [Done] implement read state and unread counts properly
- [Done] implement pin/archive/delete conversation
- [Done] implement starred messages
- [Done] implement real profile/about/shared projects

Exit criteria:

- core expected messaging features work end-to-end
- participant-specific conversation behavior persists correctly
- info panel is backed by real data

## Phase 3. Chat Experience Polish

Completed in Phase 3 so far:

- [Done] composer and keyboard polish on mobile
- [Done] responsive action sheets and drawers
- [Done] FR10. Emoji Picker Behavior

Remaining Phase 3 scope:

- [Done] typing indicator polish
- [Done] search structure for conversations and messages
- [Done] reverse infinite scroll
- [Done] deeper composer and keyboard polish on mobile
- [Done] deeper responsive action sheets and drawers

Recommended build order from here:

Phase 3 complete.

Exit criteria:

- large conversations remain usable
- mobile experience feels complete rather than adapted
- chat interactions feel polished

## Phase 4. Calls

Completed in Phase 4:

- [Done] implement call signaling on backend
- [Done] complete video call path
- [Done] add voice-only path
- [Done] add missed/rejected/ended states
- [Done] harden mobile call UI

Recommended build order from here:

Phase 4 complete.

Exit criteria:

- call UI is backed by working signaling
- common call lifecycle states behave correctly

## Media Layer Decision

**Chosen approach: LiveKit (self-hosted or LiveKit Cloud)**

Rationale:
- Backend-issued JWT tokens encode participant permissions natively
- Room lifecycle is controlled by our signaling layer, not inferred from room occupancy
- Supports both direct calls (Phase 4) and project group calls (Phase 5) under the same model
- Eliminates Jitsi moderator/join flow issues by design
- NestJS SDK available; webhook events for missed/rejected/ended state management

Rejected alternatives:
- Jitsi: meeting-product assumptions conflict with app-native call model; moderator flow is a known blocker
- Custom WebRTC/mediasoup: correct long-term but premature; SFU maintenance cost outweighs control benefits at this stage

Migration note:
- Phase 4 begins on LiveKit; Jitsi integration is retired, not patched

## Phase 5. Project Collaborator Group Calls

Current implementation pass:

- [Done] add group call support to the project detail messages tab
- [Done] scope group calls to project collaborators only
- [Done] add project-room membership validation on backend signaling
- [Done] support project-context call UI states for ringing, joined, rejected, missed, and ended
- [Done] use the same LiveKit-backed media model as direct calls

Verification note:

- manual multi-user validation is still recommended before marking Phase 5 complete in the roadmap

Exit criteria:

- project collaborators can start and receive group calls from the project detail messages tab
- non-collaborators cannot join or be signaled into project calls
- group call state is visible and consistent for all participants

## Open Product Decisions

These should be resolved before implementation in the affected areas:

1. Should message star be private per participant or shared across a conversation? per participant
2. Should delete conversation mean hide-for-me, soft delete, or destructive delete for all participants? soft delete
3. Should reactions toggle on repeat tap or allow multiple same-emoji reactions per user? toggle on repeat tap
4. What attachment size and file-type limits do we want at launch? permissive launch limits; avoid blocking normal team workflows
5. Do we want reply-to in phase 1 or phase 2? phase 2
6. Do we want direct-chat only at launch, while keeping group-chat-ready schema? yes

## Immediate Build Order

1. finalize the message contract and participant preference model
2. add validation DTOs and backend persistence changes
3. move presence to Redis
4. build optimistic send and retry
5. build responsive layout architecture
6. add unread state correctness
7. add attachments
8. add reactions, starring, and conversation management
9. replace placeholder profile/about/shared-projects
10. finish call signaling and responsive call UX

## Success Criteria

The redesign is successful when:

- chat works cleanly on mobile, tablet, and desktop
- sending feels immediate and recovers cleanly from failure
- presence is reliable and realtime
- unread counts and read state are correct
- attachments, reactions, starring, pinning, archiving, and deleting work end-to-end
- info panels show real profile and shared context
- call flows are backed by actual signaling logic
- the codebase is easier to extend instead of becoming more fragile
