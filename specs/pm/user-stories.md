# Quote Platform — User Stories

**Version:** 0.1 (Draft)
**Source:** `FeatureRequirements.md`, `TechnicalRequirements.md`, and `ArchitectureDecisionRecords.md`
**Status:** Ready for backlog refinement

## Story format

Each story follows this format:

> As a [user], I want [capability], so that [outcome].

Acceptance criteria use Given/When/Then language. Stories are intentionally sized for refinement; large stories should be split into implementation tasks during sprint planning.

## MVP release scope

The MVP includes authentication, collections, collection membership and roles, quote capture and editing, speakers, personal tierlists, reactions, search, blocking, moderation, and Discord-to-application ingestion. Real-time updates, quote-level visibility overrides, bidirectional Discord publishing, and Elasticsearch/OpenSearch are excluded from v1.

## Epic 1: Authentication and account access

### US-001 — Sign in with OAuth

**Priority:** Must have

As a visitor, I want to sign in with OAuth, so that I can use the application without managing a separate password.

**Acceptance criteria**

- Given I am unauthenticated, when I select the configured OAuth provider, then I am redirected to the provider's authorization flow.
- Given authorization succeeds, when the callback is processed, then an account is created or the existing account is signed in.
- Given authorization fails or is cancelled, then no account is created and I receive an actionable error.
- Provider access tokens and secrets are never exposed to the browser or written to application logs.

### US-002 — Manage my session

**Priority:** Must have

As a signed-in user, I want my session to remain secure and expire appropriately, so that my account is protected on shared or lost devices.

**Acceptance criteria**

- The application can identify the signed-in user on subsequent requests.
- I can sign out and the session is invalidated.
- Expired or invalid sessions are rejected and do not expose protected resources.
- Browser requests use CSRF protection and the application enforces an explicit CORS policy.

## Epic 2: Collections and access

### US-003 — Create a collection

**Priority:** Must have

As a user, I want to create a collection, so that I can organize related quotes in one place.

**Acceptance criteria**

- Given I am signed in, when I submit a valid collection name, then a collection is created and I become its Owner.
- A collection has a unique identifier, name, visibility, creator, and creation timestamp.
- A collection can be marked public or private.
- Invalid or blank names produce field-level validation feedback.

### US-004 — Browse a collection

**Priority:** Must have

As a user, I want to view a collection I can access, so that I can read and interact with its quotes.

**Acceptance criteria**

- Public collections are visible to users who are not members, subject to blocking rules.
- Private collections are visible only to authorized members.
- The collection view shows its name, visibility, accessible quotes, and the current user's role.
- Unauthorized access does not reveal private collection content through direct URLs or search.

### US-005 — Manage collection membership

**Priority:** Must have

As a collection Owner or authorized Moderator, I want to add, remove, and manage members, so that access remains controlled.

**Acceptance criteria**

- Membership is scoped to one collection; a user's role in one collection does not change their role elsewhere.
- Owners can assign the baseline roles Owner, Moderator, Member, and Viewer according to the permission model.
- Unauthorized users cannot add, remove, or change members.
- Removing a member immediately prevents access that depended on that membership.
- A collection always retains an Owner, and ownership transfer is explicit and audited.

### US-006 — Manage collection settings

**Priority:** Must have

As a collection Owner, I want to update collection settings or delete the collection, so that I can manage its lifecycle.

**Acceptance criteria**

- Only an Owner can change collection name or visibility and initiate deletion.
- Deletion requires an explicit confirmation.
- Deleted collection content is not returned by normal reads or search.
- The deletion event is recorded for authorized audit or moderation workflows.

## Epic 3: Quotes and speakers

### US-007 — Create a quote

**Priority:** Must have

As a collection Member, I want to add a quote to a collection, so that memorable statements can be preserved.

**Acceptance criteria**

- A quote belongs to exactly one collection in v1.
- Members and users with equivalent write permission can create quotes; Viewers cannot.
- A quote requires non-empty text and supports either a linked collection speaker or an unlinked speaker name.
- The creator and creation timestamp are recorded.
- The initial quote version is recorded in the quote changelog.

### US-008 — Add and manage speakers

**Priority:** Must have

As a collection member, I want to add speakers and use display names, so that quotes can be attributed without exposing unnecessary identity details.

**Acceptance criteria**

- A speaker can be added to a collection with a display/fake name.
- A quote can reference an existing collection speaker or retain an unlinked/free-text speaker name.
- An unlinked speaker is not automatically matched to a collection speaker by name.
- Speaker names are subject to validation and collection permissions.

### US-009 — Relink an unlinked speaker

**Priority:** Must have

As a collection Moderator, I want to explicitly relink an unlinked speaker to a collection speaker, so that attribution can be corrected safely.

**Acceptance criteria**

- Only an authorized Moderator or Owner can perform the relink.
- The user must explicitly confirm which collection speaker will receive the quotes.
- The relink updates the selected quotes and records a quote revision and audit event for each affected quote.
- No other quotes are changed by the operation.

### US-010 — Edit a quote

**Priority:** Must have

As a quote author or authorized Moderator, I want to edit a quote, so that errors can be corrected while preserving its history.

**Acceptance criteria**

- A user can edit only quotes they are authorized to edit.
- An edit records the editor, timestamp, changed fields, previous values, and new values.
- The current quote shows the latest values while its prior revisions remain available to authorized users.
- Editing a quote does not change its collection unless the user performs an explicit move operation.

### US-011 — View quote history

**Priority:** Should have

As an authorized user, I want to view a quote's revision history, so that I can understand how its text or attribution changed.

**Acceptance criteria**

- History is ordered newest-first or oldest-first consistently and shows the ordering clearly.
- Each revision identifies the actor, timestamp, changed fields, and before/after values.
- Unauthorized users cannot use revision history to discover hidden or deleted content.

### US-012 — Remove or restore a quote

**Priority:** Must have

As an authorized Moderator, I want to hide and restore quotes, so that inappropriate or incorrect content can be moderated without losing its audit trail.

**Acceptance criteria**

- Authorized Moderators can soft-delete/hide a quote with a reason.
- Hidden quotes are excluded from normal collection views and search.
- Authorized moderation users can view the hidden state and restore the quote.
- Hide, restore, actor, timestamp, and reason are recorded in the audit history.

## Epic 4: Ranking and reactions

### US-013 — Rank quotes personally

**Priority:** Must have

As a collection Member, I want to place quotes into my personal tierlist, so that I can rank them according to my own preferences.

**Acceptance criteria**

- Each user's ranking is independent from every other user's ranking.
- A user can move a quote between tiers and reorder quotes within a tier.
- A user can remove a quote from their tierlist without deleting the quote.
- Only quotes from collections the user can access can be ranked.
- The user's ranking persists across sessions.

### US-014 — React to a quote

**Priority:** Must have

As a collection Member, I want to react to a quote, so that I can quickly express my response.

**Acceptance criteria**

- The interface exposes only the fixed, application-defined reaction types.
- A user can add or remove their reaction for a quote.
- A user can have at most one reaction of each type on a quote.
- Reaction totals respect collection visibility and blocking rules.
- Viewers cannot react unless the permission model explicitly grants them interaction access.

## Epic 5: Friends and blocking

### US-015 — Send and respond to friend requests

**Priority:** Should have

As a user, I want to send, accept, decline, and cancel friend requests, so that I can manage social connections.

**Acceptance criteria**

- A user can send a request to another eligible user and see its pending state.
- The recipient can accept or decline the request.
- The sender can cancel a pending request.
- Duplicate pending requests are prevented.
- Friendship does not automatically grant access to private collections.

### US-016 — Block a user

**Priority:** Must have

As a user, I want to block another user, so that I can prevent direct interaction and hide their content from my views.

**Acceptance criteria**

- Blocking prevents direct interaction covered by the product's blocking policy.
- The blocked user's quotes and reactions are hidden from the blocker's views and search results.
- Blocking does not remove the blocked user's content for other users.
- The blocker can unblock the user.
- Block and unblock actions do not expose private information about either user.

## Epic 6: Search

### US-017 — Search accessible quotes

**Priority:** Must have

As a user, I want to search quote text and speaker names, so that I can quickly find relevant quotes.

**Acceptance criteria**

- Search works within a selected collection and across collections the user can access.
- Search covers quote text and speaker names.
- Results exclude inaccessible, blocked, soft-deleted, and hidden content.
- Results support consistent pagination.
- Empty, malformed, or overly long queries receive useful validation feedback.

## Epic 7: Moderation and audit

### US-018 — Report a quote

**Priority:** Must have

As a user, I want to report a quote, so that inappropriate or problematic content can be reviewed.

**Acceptance criteria**

- A user can submit a report with a reason and optional explanation.
- The report identifies the quote, reporter, collection, status, and timestamps.
- A user cannot create duplicate unresolved reports for the same quote and reason.
- The reporter can see whether their report is received or resolved without seeing restricted moderator notes.

### US-019 — Review moderation reports

**Priority:** Must have

As a collection Moderator, I want to review reports from my collections, so that I can take appropriate action.

**Acceptance criteria**

- Collection Moderators see only reports for collections they moderate.
- Platform Admins can see reports across collections.
- A reviewer can mark a report open, resolved, dismissed, or escalated according to the moderation policy.
- Available actions are permission-checked and recorded with actor, timestamp, action, and reason.
- Moderation actions use soft deletion or hiding where applicable.

### US-020 — Audit administrative actions

**Priority:** Must have

As a platform administrator, I want administrative and moderation actions audit-logged, so that decisions can be investigated and appealed.

**Acceptance criteria**

- Logs include actor, action, target, timestamp, reason when supplied, and relevant before/after state.
- Audit records cannot be changed through normal user workflows.
- Audit records are visible only to authorized platform or collection administrators.
- Secrets, access tokens, and API keys never appear in logs.

## Epic 8: Discord integration

### US-021 — Install and authorize the Discord bot

**Priority:** Must have

As a collection Owner, I want to connect a Discord guild and authorize the bot, so that quotes can be captured from configured channels.

**Acceptance criteria**

- The installation flow identifies the Discord guild and the application user authorizing it.
- The user must have sufficient Discord and application permissions to complete the connection.
- A guild can be connected to multiple collections.
- Connection and authorization changes are audit-logged.

### US-022 — Map a Discord channel to a collection

**Priority:** Must have

As a collection Owner or authorized Moderator, I want to map a Discord channel to a collection, so that incoming quotes are routed correctly.

**Acceptance criteria**

- A source channel can be mapped to one configured collection at a time.
- The mapping belongs to a guild and collection and can be changed or removed by an authorized user.
- Messages from unmapped channels are not imported as quotes.
- Mapping changes are recorded for troubleshooting and audit purposes.

### US-023 — Ingest a quote from Discord

**Priority:** Must have

As a Discord user, I want to submit a quote through the configured bot workflow, so that it appears in the mapped collection without re-entering it in the web app.

**Acceptance criteria**

- A supported Discord command or message format creates a quote in the channel's mapped collection.
- The imported quote records its Discord guild, channel, message ID, author, and source timestamp where available.
- The bot rejects messages from unmapped channels or unauthorized workflows with a useful response.
- Duplicate Discord events do not create duplicate quotes.
- Import failures are retried or placed in an observable failure state without losing the source event.

### US-024 — Respect Discord and application limits

**Priority:** Must have

As a platform operator, I want Discord commands and outbound bot actions to respect rate limits, so that the integration remains reliable and does not get blocked.

**Acceptance criteria**

- Discord command limits are independent from web-session and API-key limits.
- The bot honors Discord rate-limit responses and retry timing.
- Repeated application rate-limit violations produce a clear user-facing response.
- Rate-limit events and failed retries are observable without logging secrets.

## Epic 9: API access

### US-025 — Create and manage an API key

**Priority:** Should have

As an authorized user, I want to create, rotate, and revoke scoped API keys, so that I can integrate with the platform safely.

**Acceptance criteria**

- A key has an explicit read or write scope and an identifiable name.
- The plaintext key is shown only at creation time; only a secure hash is stored.
- I can revoke a key and see its last-used timestamp.
- A key cannot access resources or perform actions beyond the issuing user's current permissions.
- API-key creation, rotation, revocation, and use are audit-logged without logging the secret value.

## Epic 10: Reliability and usability

### US-026 — Receive actionable errors

**Priority:** Must have

As a user, I want clear error messages when an action fails, so that I know how to correct the problem.

**Acceptance criteria**

- API errors use a consistent Problem Details response shape.
- Validation errors identify the affected field and provide a stable machine-readable code.
- Rate-limit errors return HTTP 429 and a `Retry-After` value.
- The frontend presents errors in the relevant workflow instead of silently discarding them.

### US-027 — Use the primary workflows accessibly

**Priority:** Must have

As a user with accessibility needs, I want the primary workflows to be accessible, so that I can create, browse, search, and moderate quotes using supported assistive technology.

**Acceptance criteria**

- Primary workflows target WCAG 2.1 AA.
- Forms have associated labels, keyboard access, and visible validation errors.
- Interactive controls expose meaningful names and states to assistive technology.
- Focus is managed when dialogs, validation errors, or navigation changes occur.

### US-028 — Use the application on mobile devices

**Priority:** Must have

As a mobile user, I want the primary workflows to work on my phone or tablet, so that I can capture and browse quotes wherever I am.

**Acceptance criteria**

- Quote browsing, quote creation, search, authentication, reactions, and tierlist ranking work at narrow phone and tablet viewport widths.
- Primary workflows do not require horizontal scrolling or desktop-only hover interactions.
- Essential actions remain visible or reachable through an accessible mobile navigation pattern.
- Forms and dialogs fit within the viewport and remain usable when the on-screen keyboard is open.
- Touch targets have sufficient size and spacing for reliable use on touch devices.
- Responsive behavior is tested at representative phone, tablet, and desktop viewport sizes.

### US-029 — Use responsive moderation tools

**Priority:** Should have

As a moderator using a mobile device, I want to review reports and take moderation actions responsively, so that urgent issues can be handled away from a desktop.

**Acceptance criteria**

- Moderators can view report details, apply permitted actions, and enter reasons on a phone or tablet.
- Tables, action menus, and report details adapt to narrow screens without hiding essential information.
- Destructive actions remain clearly labeled and require an accessible confirmation step.
- Validation and authorization errors are visible in the mobile layout.

## Deferred stories

These are intentionally excluded from v1 and should not be pulled into the MVP backlog without a scope decision:

- Bidirectional posting from the application back to Discord
- Quote-level visibility overrides inside a collection
- Elasticsearch/OpenSearch search
- Real-time updates via WebSockets or Server-Sent Events
- Additional OAuth providers beyond the initial provider selection

## Cross-story definition of done

A story is complete when:

- Acceptance criteria are covered by automated tests where practical.
- Authorization and visibility behavior is tested for both allowed and denied users.
- API changes are reflected in the OpenAPI specification.
- Database changes include a reviewed Liquibase migration.
- User-facing errors are actionable and field-level where applicable.
- Metrics, structured logs, and audit events are added for operationally significant workflows.
- The change does not expose secrets or bypass blocking, collection visibility, or role permissions.
