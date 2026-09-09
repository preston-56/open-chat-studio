---
status: active
---

# Session/Message Sync API (v2)

> Design doc for `GET /api/v2/sessions/` and `GET /api/v2/messages/`. Read endpoints for
> ETL-style consumers (data warehouse, BI tool, CRM) that need to sync OCS session and message
> data incrementally instead of re-paging the full dataset on every pull. From #3904.

## TL;DR

Two new read-only endpoints, cursor pagination ordered by `updated_at`, and a `since` filter for
incremental pulls. Most of the infrastructure this needs already exists in the codebase, so the
work is mainly wiring, not new sync machinery. Nine decisions shape the work; D1 needs Simon's
confirmation before the rest proceed, since it touches an accepted ADR.

## Context

### The use case

A client wants to keep an external system (warehouse, CRM) in sync with OCS session and message
data without re-fetching everything on every pull. v1's `GET /api/v1/sessions/` supports this
poorly: no `updated_at` filter, no flat message list (`GET /sessions/{id}/` only), and while
pagination happens to already be cursor-based (DRF default), it's ordered by `created_at`, so an
edited record between pulls never surfaces.

### Existing landscape (what we reuse)

- `apps/api/pagination.py::CursorPagination`: the DRF-wide default (`DEFAULT_PAGINATION_CLASS`),
  `(-created_at, -pk)`, with a `count` field on the first page. Already used by v1 sessions.
- `apps/utils/models.py::BaseModel.updated_at`: `auto_now=True`. Inherited by `ExperimentSession`
  and `ChatMessage`, neither overrides it. No migration needed for `since`.
- `apps/experiments/filters.py::ExperimentSessionFilter`: has `ParticipantFilter`,
  `ChannelsFilter`, `ChatMessageTagsFilter` already, confirming the underlying model fields to
  filter on directly.
- `config/settings.py::OAUTH_CLIENT_CREDENTIALS_SCOPES`: declares `sessions:read` already.
- `config/settings.py::DEFAULT_THROTTLE_CLASSES`: `apps.api.throttling.APIRateThrottle` applies
  to every DRF view already, `api` scope, 2000/5m. No new throttle work needed.
- `apps/api/v2/usage/`: closest sibling package (`views.py`/`serializers.py`/`permissions.py`/
  `services.py` split). Its `CanViewUsage` permission class grants client-credentials tokens
  automatically and checks a Django permission for user-delegated tokens; the same shape applies
  here with `experiments.view_experimentsession` and `chat.view_chatmessage`.

## Decisions

| Decision | Notes |
|---|---|
| D1: flat, not nested, session/message routes | contradicts [ADR-0023](../adr/0023-rename-experiment-to-chatbot-in-v2.md), needs sign-off |
| D2: `updated_at`-ordered cursor pagination | new subclass |
| D3: drop the proposed `next_since` field | cursor's `next` already covers it |
| D4: one scope, `sessions:read`, for both endpoints | already declared, unused |
| D5: filter directly in `get_queryset()` | not via `ExperimentSessionFilter` |
| D6: `/messages/` takes a `participant` filter | resolves SmittieC's open question |
| D7: IDs follow [ADR-0026](../adr/0026-identify-resources-by-primary-key.md) | public identifier where one exists, PK otherwise |
| D8: include `metadata` on messages | matches v1, carries `compression_marker` |
| D9: batch-prefetch attachments per page | `get_attached_files()` isn't safe at flat-list scale |

### D1: Flat, not nested, session/message routes

**Decision.** `/api/v2/sessions/` and `/api/v2/messages/` as flat, top-level, cursor-paginated
collections, with `chatbot` as an optional filter, not path segments.

**Context.** [ADR-0023](../adr/0023-rename-experiment-to-chatbot-in-v2.md) already decided
sessions live at `/api/v2/chatbots/{id}/sessions/`, nested under their chatbot. That route
doesn't exist yet, so this ticket would be its first implementation. ADR-0023 is a naming ADR
(`experiment` renamed to `chatbot` throughout v2); its Context and Alternatives sections argue
only that. The nested route is one clause riding along with the rename, not an independently
reasoned structural decision, no alternative resource shape is discussed or rejected anywhere in
it. Simon (its author) already sketched the flat shape himself on the issue thread when he asked
SmittieC for feedback, without cross-referencing ADR-0023.

**Consequences.** A sync consumer polls once per team instead of once per chatbot per cycle,
which is the actual reason to diverge. If approved, this needs a new ADR superseding 0023 for
these two resources. If rejected, both endpoints move under `/api/v2/chatbots/{id}/sessions/` and
`/api/v2/chatbots/{id}/messages/`, and the standalone `chatbot` filter param goes away; D2 through
D9 are unaffected either way.

**Alternatives considered.** Keep the nested shape as ADR-0023 states it: rejected for this use
case specifically, not as a general objection to the convention, since incremental sync needs one
team-wide pull, not N calls for N chatbots.

### D2: `updated_at`-ordered cursor pagination

**Decision.** A new `CursorPagination` subclass, `ordering = ("updated_at", "pk")`, ascending.

**Context.** The existing default orders by `(-created_at, -pk)`, which misses an edited (not
newly created) row on an incremental pull.

**Consequences.** Two small subclasses, one per endpoint, under `apps/api/v2/sync/pagination.py`.

**Alternatives considered.** Reuse the default ordering and have clients re-fetch everything
periodically to catch edits: rejected, defeats the point of an incremental sync API.

### D3: Drop the proposed `next_since` response field

**Decision.** No `next_since` field in the response envelope.

**Context.** The issue proposes it, but it duplicates what the cursor's `next` URL already
encodes.

**Consequences.** A client stores `next`, not a timestamp it re-derives itself; one resumable
pointer instead of two.

**Alternatives considered.** Return both: rejected, one client-visible pointer is simpler and a
raw timestamp comparison on the client side risks missing or duplicating rows at the boundary in
a way cursor pagination already avoids.

### D4: One scope, `sessions:read`, for both endpoints

**Decision.** `required_scopes = ["sessions:read"]` on both endpoints, no separate `messages`
scope.

**Context.** `sessions:read` is already declared in `OAUTH_CLIENT_CREDENTIALS_SCOPES`, unused by
any view yet. Messages are always synced in the context of their session.

**Consequences.** This ticket is the scope's first real consumer, no new scope to register.

**Alternatives considered.** A dedicated `messages:read` scope: rejected, no existing resource
in this codebase splits sub-resources into their own scope, and nothing here lets a client read
messages without also being trusted to read the sessions they belong to.

### D5: Filter directly in `get_queryset()`

**Decision.** Implement `chatbot`/`participant`/`channel`/`tags` as plain queryset filters in
each view, not through `ExperimentSessionFilter`.

**Context.** That filter class is built for the dynamic filter widget's `f_<field>`/`op_<field>`
query param shape and HTML filter UI, not a public API's `?participant=<id>` shape. `channel`
maps to `.filter(platform=<value>)`, confirmed against `ChannelsFilter.column`.

**Consequences.** Three direct `.filter()` calls, independent of an internal UI component's
param contract.

**Alternatives considered.** Adapt `ExperimentSessionFilter` to accept the public param shape:
rejected, more code than reimplementing three filters directly, and couples a public API's
contract to an internal widget's.

### D6: `/messages/` takes a `participant` filter

**Decision.** `/api/v2/messages/` accepts `participant` (`public_id`), same as `/sessions/`.

**Context.** Resolves SmittieC's open question on the issue thread: looking up a participant's
messages without a session id first.

**Consequences.** One query shape works whether the client already has the participant id or the
session id.

**Alternatives considered.** A separate `participant_identifier` string-lookup param, as
suggested on the thread: rejected, `participant` alone covers the case without a second param.

### D7: IDs follow ADR-0026

**Decision.** `session.id` sources `external_id`. `chatbot_id` sources `experiment.public_id`.
`participant_id` sources `participant.public_id`. `message.id` is the raw primary key.

**Context.** [ADR-0026](../adr/0026-identify-resources-by-primary-key.md) says reuse an existing
public identifier where one exists, PK otherwise. Sessions, chatbots, and participants already
have one (matches v1 and `TriggerBotMessageResponse`'s `session_id` field,
`ParticipantDataEntrySerializer`'s `chatbot_id` field). Messages don't, and this endpoint is
authenticated and team-scoped, the same reasoning ADR-0026 already applied to inspect's nested
resources.

**Consequences.** No new public-ID fields anywhere, consistent with every other v2 serializer.

**Alternatives considered.** Add a `public_id` to `ChatMessage`: rejected, ADR-0026 already
rejected this generally and nothing here changes that reasoning.

### D8: Include `metadata` on messages

**Decision.** `metadata` is a message response field.

**Context.** v1's `MessageSerializer` already exposes it; it's where `compression_marker` lives
(from #4012).

**Consequences.** Sync consumers keep parity with what v1 callers already see.

**Alternatives considered.** Drop it for a smaller payload: rejected, a step back for a sync
consumer, not a simplification.

### D9: Batch-prefetch attachments per page

**Decision.** Prefetch attachments once per page of messages, not via a per-message
`get_attached_files()` call.

**Context.** That method issues its own query per call, fine for a session's small embedded
message list (v1's existing behavior), not fine for up to 1500 messages per page on the new flat
list.

**Consequences.** New helper, same shape as `attach_chat_tagged_items`
(`apps.annotations.prefetch`), which already does this for tags.

**Alternatives considered.** Call `get_attached_files()` per message as v1 does: rejected, real
N+1 risk at this endpoint's scale that v1 never had to handle.

## Endpoint shape

```
GET /api/v2/sessions/
GET /api/v2/sessions/{id}/
GET /api/v2/messages/
```

## Response shape

### `GET /api/v2/sessions/`

| Param | Type | Notes |
|---|---|---|
| `chatbot` | UUID | `public_id`, filter to one chatbot |
| `participant` | UUID | `public_id`, filter to one participant |
| `channel` | slug | Platform, e.g. `web`, `whatsapp` |
| `tags` | string | Filter by tag name |
| `since` | ISO 8601 | Only rows created or updated after this time |
| `cursor` | opaque | Pagination cursor from a previous response |
| `page_size` | int | Default 100, max 1500 |

| Field | Source | Notes |
|---|---|---|
| `id` | `external_id` | Existing public identifier, matches v1 |
| `chatbot_id` | `experiment.public_id` | |
| `participant_id` | `participant.public_id` | |
| `created_at` / `updated_at` | direct | |
| `ended_at`, `state`, `tags`, `url` | direct | |

### `GET /api/v2/sessions/{id}/`

Same fields as above, plus embedded `messages[]` (same shape as `/messages/` below).

### `GET /api/v2/messages/`

| Param | Type | Notes |
|---|---|---|
| `chatbot` | UUID | `public_id` |
| `session` | UUID | `external_id`, filter to one session |
| `participant` | UUID | `public_id` |
| `since` | ISO 8601 | |
| `cursor` | opaque | |
| `page_size` | int | Default 100, max 1500 |

| Field | Source | Notes |
|---|---|---|
| `id` | primary key | No existing public identifier for messages, see D7 |
| `session_id` | `external_id` | |
| `role`, `content`, `metadata` | direct | `metadata` carries `compression_marker`, see #4012 |
| `created_at` / `updated_at` | direct | |
| `attachments[]`, `tags[]` | direct | Batch-prefetched per page, see D9 |

## Implementation plan

1. `apps/api/v2/sync/pagination.py`: the `updated_at`-ordered `CursorPagination` subclass (D2).
2. `apps/api/v2/sync/serializers.py`: session and message serializers, ID mapping per D7.
3. `apps/api/v2/sync/views.py`: two views, `get_queryset()` applying the filters above directly
   (D5, D6), batch attachment prefetch (D9).
4. `apps/api/v2/sync/permissions.py`, only if `usage/permissions.py`'s `CanViewUsage` pattern
   isn't reusable as-is.
5. Wire into `apps/api/v2/urls.py`, add `@extend_schema` for OpenAPI.

### Test plan

- First full pull, incremental pull with `since`, empty incremental pull.
- Cursor stability under a concurrent write between pages.
- Each filter (`chatbot`, `participant`, `channel`, `tags`, `session`) and the scope gate.
- Attachment prefetch query count on a page with several messages (guards D9).

## Open questions

- Should `/sessions/{id}/` paginate its embedded messages for very long sessions, or stay
  unpaginated like v1's retrieve endpoint? Leaning toward matching v1 unless real session sizes
  make that painful.

## Related issues

- [#3904](https://github.com/dimagi/open-chat-studio/issues/3904): origin of this design.
- [#4012](https://github.com/dimagi/open-chat-studio/issues/4012): source of the `compression_marker` metadata surfaced in D8.
