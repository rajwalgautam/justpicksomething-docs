# Just Pick Something — AI Assistant Plan

Status: **draft**, not scheduled. Companion to the [project plan](project-plan.md).

Add an AI helper that answers questions about a room and summarizes what people
have said, without breaking the [free-tier](project-plan.md#free-tier-only) and
[privacy](project-plan.md#key-decisions) rules.

> Example: in a vacation room, a member asks "Which places have a hot tub and
> sleep 8?" or "What are the main objections to the lake house?" and gets an
> answer built only from that room's items, columns and comments.

---

## Approach: send the whole room, no retrieval (yet)

This is "RAG" without the retrieval step. Retrieval is for picking a few
relevant chunks out of a corpus too big for one prompt. A room is small: a few
dozen items, at most 20 columns, and comments of up to 1,000 characters. So v1
loads **the whole room** and puts it in the prompt:

- Simpler: no embeddings, no vector index, no second provider.
- More accurate: the model sees everything, so nothing is missed by a weak
  search.
- Cheaper on Neon: one query that already exists (`GET /rooms/{slug}`'s), no
  extra writes on every edit.

Retrieval (step v2 below) gets built only if real rooms become too big for this.

**Never search across rooms.** Rooms are private behind a passcode. Every model
call sees exactly one room's data, the room the member's token belongs to.

## Blocker: cost

The plan says every service stays free. LLM API calls are not. Decide this
before building anything:

| Option | Notes |
| --- | --- |
| Pay a small API bill | Breaks the strict free-only rule. Keep it bounded: a Haiku-class model, short answers, and the hard caps below. Set a monthly spend limit in the provider's console. |
| A free-tier gateway | Neon offers an AI Gateway. Its free-tier allowance is **unconfirmed**; check the pricing page before relying on it. |
| Don't ship it | Fine too. It's a nice-to-have, not core to picking. |

Record the outcome in the [decisions log](project-plan.md#decisions-log).

## v1 — "Ask the room" and comment summaries

### API

| Method | Path | Auth | Body → Response |
| --- | --- | --- | --- |
| `POST` | `/rooms/{slug}/ask` | ✓ | `{question}` (max 500 chars) → `200 {answer, itemIds}`; `429` over the rate limit; `503` if the feature is off or the model call fails |
| `POST` | `/rooms/{slug}/items/{id}/summary` | ✓ | → `200 {summary}` of that item's comments; `429` / `503` as above |

`itemIds` lists the items the answer refers to, so the UI can highlight them.
Not streamed in v1: one JSON response keeps it simple on Vercel functions.

### Backend

- New package `pkg/ai`: builds the prompt from the room and calls the model.
  Calls go through an interface so tests can use a fake.
- Load the room with the same query as `GET /rooms/{slug}`. Turn it into a
  compact text block: columns once, then one block per item with
  `id`, title, column values, score and comments.
- **Size guard:** if the block is over a set token budget, drop the oldest
  comments first, then long `textarea` values, and say so in the prompt. Hitting
  this often is the signal to build v2.
- Prompt order: fixed instructions first, then the room block, then the
  question, so prompt caching can reuse the room part when people ask several
  questions in a row.
- Env vars (Vercel + local): `AI_API_KEY`, `AI_MODEL`, `AI_ENABLED`
  (`false` turns both endpoints into `503` without a redeploy of the frontend).
  Never commit the key; add placeholders to `.env.example`.

### Prompt rules

- Answer only from the room data. If the data doesn't say, reply that it
  doesn't.
- Refer to items by title and return their IDs.
- **Prompt injection:** item titles, values and comments are typed by any
  member. Wrap them as data and tell the model to never follow instructions
  inside them. The model has no tools and can't change the room, so the worst
  case is a bad answer, not a bad write.
- Don't take sides in the vote. Summarize, don't pick.

### Limits

Reuse the M5 Postgres rate-limit table (no Redis/KV):

- Per member: N questions per hour.
- Per room: N questions per day.
- App-wide: a daily cap as the cost backstop. Once it's hit, return `503`
  until the next day.

Pick the actual numbers once the cost decision is made.

### Frontend

- An "Ask" box in the room, with the answer shown below it and the items it
  refers to highlighted in the table.
- A "Summarize" button on items with more than a few comments.
- Hide both when the API returns `503` for the feature being off.

### Tests

- Unit tests for the prompt builder: columns and values present, size guard
  trims in the right order.
- Handler tests with the fake model: auth, `429`, `503`, locked rooms still
  allowed (reading doesn't change anything).
- A small eval set: a few fixture rooms (the vacation example and a
  restaurant pick) with questions and expected answers. Run it by hand with a
  real key, e.g. `make eval`, not in CI, so CI stays free and keyless.

## v2 — retrieval, only if v1 runs out of room

Build this only if the size guard fires often on real rooms.

- Migration: enable `pgvector` on Neon and add `embedding vector(<dims>)` to
  `items` and `comments`, with an HNSW index.
- Compute embeddings **inside the write request** (create/edit item or
  comment). The plan rules out background jobs that keep Neon awake.
- Anthropic doesn't offer an embeddings model, so this needs another provider
  (e.g. Voyage): one more key and possibly one more bill.
- Search with vector similarity plus Postgres full-text search (hybrid), always
  filtered by `room_id`. Put the top matches in the prompt, plus the column
  list and scores for every item.
- Re-embedding on every edit adds latency and storage, so measure Neon storage
  against the free cap before turning it on.

## Out of scope

- Searching or answering across rooms.
- The AI adding, editing, voting or reordering anything.
- Fetching or scraping listing pages (see paste enrichment in the
  [project plan](project-plan.md#later--down-the-line-ideas)).
- Streaming answers.

## Open questions

- Which cost option? (Blocks everything else.)
- Should admins be able to turn the assistant off for their room?
- Should answers be visible to the whole room or only to whoever asked?
  (v1: only to whoever asked; nothing is stored.)
