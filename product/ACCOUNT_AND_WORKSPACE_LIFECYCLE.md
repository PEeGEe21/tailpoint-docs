# Account and Workspace Lifecycle

**Priority:** P0 — Critical  
**Status:** Ready for implementation planning  
**Decision date:** 2026-09-09  
**Surfaces:** Backend API, web workspace, mobile app

## Problem

Tailpoint currently couples account registration to creation of the first
organization. The public `signup/create-organization` operation rejects an
email that already belongs to a user, so a signed-out existing user cannot use
that flow to create another workspace. Authentication can also appear to
depend on access to an existing workspace.

This conflates two different concepts:

- An **account** identifies and authenticates a person.
- A **workspace membership** authorizes that account within an organization.

## Resolutions

1. A person signs in to their Tailpoint account, not to a workspace.
2. The same account and email may belong to multiple organizations.
3. Losing or leaving every organization must not prevent account
   authentication. After sign-in, a user with no active memberships is sent to
   a create-or-join workspace state.
4. An authenticated user creates an additional organization through a new
   authenticated organization endpoint. The client asks only for workspace
   details; it must not submit the signup form or recreate the account.
5. The creator receives an active organization-admin membership, and the new
   organization becomes the active organization. Tokens/session context must
   be issued or rotated accordingly.
6. Account-scoped sessions without an active organization may access only the
   small set of account, workspace-list, workspace-create, workspace-join,
   recovery, and sign-out operations. They may not access organization data.
7. A globally suspended or deleted account cannot bypass that restriction by
   creating a workspace.
8. New-account registration requires email ownership verification before the
   account and first workspace are finalized. A verified existing account does
   not repeat email verification for every new workspace unless a separately
   approved step-up-authentication policy requires it.
9. Web and mobile must implement the same lifecycle, terminology, API
   contracts, and recovery states.

## Required backend work

- Separate account authentication from active-organization selection.
- Allow successful login with zero active organization memberships and return
  an explicit next step such as `create_or_join_organization`.
- Introduce an authenticated organization-creation endpoint, for example
  `POST /api/organizations`, with authorization suitable for an account-scoped
  session.
- Create the organization, initial subscription, admin membership, and active
  session context atomically.
- Implement a general email-verification lifecycle for signup. Do not reuse
  password-reset verification state.
- Define expiry, resend, attempt limits, throttling, one-time consumption, and
  non-enumerating responses for verification codes.
- Preserve tenant isolation for tokens that have no active organization.
- Add audit records for organization creation and membership assignment.

## Required web and mobile work

- Keep public registration for genuinely new accounts only.
- After authentication, route zero-membership accounts to a dedicated
  create-or-join workspace screen.
- Provide "Create another workspace" to authenticated users without asking for
  email, password, or personal details again.
- Add email-code entry, resend, expiry, loading, error, and recovery states to
  new-account registration.
- Refresh the organization list and enter the new workspace after successful
  creation.
- Explain account conflicts by directing an existing email owner to sign in or
  recover their password rather than suggesting that the email cannot be used
  for another workspace.

## Acceptance criteria

- A verified account can own or join more than one organization using one
  email address.
- A signed-out existing user can authenticate even when they have no active
  workspace membership.
- A zero-membership user can create a workspace or join one by invitation, but
  cannot query any organization-scoped resource beforehand.
- Creating an additional workspace never creates a duplicate user and never
  calls a signup endpoint.
- A new user cannot complete account/first-workspace registration until their
  email code is verified.
- Verification codes expire, are single-use, are attempt- and resend-limited,
  and do not expose whether arbitrary email addresses are registered.
- Workspace creation and initial membership are transactional; failures leave
  neither an orphan organization nor a partial membership.
- After creation, both clients show the new workspace as active and include all
  other active memberships in the switcher.
- Backend authorization tests and web/mobile end-to-end tests cover new-user,
  existing-user, zero-membership, removed-member, suspended-account, expired
  code, duplicate submission, and rollback cases.

## Current implementation gap

The current public `POST /api/auth/signup/create-organization` operation is an
account-registration endpoint and correctly rejects an existing email for that
specific purpose. It must not be broadened to silently reuse an existing user.
The separate authenticated organization-creation operation and general signup
email verification are currently missing.
