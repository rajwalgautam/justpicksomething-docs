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
| Pay a small API bill | Breaks the strict free-only rule. Keep it bounded: a Haiku-class model, short answers, and the hard caps in [API access and spend cap](#api-access-and-spend-cap). |
| A free-tier gateway | Neon offers an AI Gateway. Its free-tier allowance is **unconfirmed**; check the pricing page before relying on it. |
| Don't ship it | Fine too. It's a nice-to-have, not core to picking. |

Record the outcome in the [decisions log](project-plan.md#decisions-log).

## API access and spend cap

If we pay for the API, the backend gets its own access with a hard monthly cap
enforced by Anthropic, not just by our code.

**A claude.ai subscription (Pro/Max) doesn't cover API usage.** API calls are
billed separately in the [Claude Console](https://platform.claude.com) with
prepaid credits. The same login works there.

### Setup (one time, by hand)

1. Sign in to the Console, add billing, and buy a small amount of credit.
2. Create a workspace `justpicksomething` (Settings → Workspaces). Don't use the
   Default Workspace: spend limits can't be set on it.
3. On that workspace, set **Spend limits**: the monthly cap, plus alerts at
   e.g. 50% and 80%.
4. Optionally set **workspace rate limits** (requests, input tokens and output
   tokens per minute), so a burst of abuse can't use the whole month's budget
   in an hour.
5. Create a **service account** (Settings → Service accounts), add it to the
   workspace, and create a key scoped to that workspace. A service account key
   isn't tied to a personal user, and a workspace-scoped key can only spend
   against this workspace's cap.
6. Put the key in Vercel as `AI_API_KEY` (Production only) and in the local
   `.env`. Never commit it.
7. Leave **auto-reload** of credits off, so spend also can't pass the credit
   bought. (Check this setting on the Billing page; it isn't covered in the
   docs we read.)

Record the cap amount in the [decisions log](project-plan.md#decisions-log)
once it's picked.

### Layers of the cap

| Layer | Where | What happens when it's hit |
| --- | --- | --- |
| Our rate limits | Postgres table ([Limits](#limits)) | We return `429` before calling the API |
| Workspace spend limit | Console → Workspaces | API returns `400` `invalid_request_error`, message starts with "You have reached your specified workspace API usage limits" |
| Organization spend limit | Console → Settings → Billing | Same as the workspace limit, message without "workspace" |
| Tier spend cap | Set by Anthropic (Start tier: $500/month) | API returns `429` with `error.details.error_code: "enforced_spend_limit_reached"` and no `retry-after` |
| Prepaid credit | Console → Billing | Calls fail once credit runs out (auto-reload off) |

Spend limits reset at the start of each calendar month (00:00 UTC on the 1st).

### Backend handling

- Treat a spend-limit `400`, or a `429` with `enforced_spend_limit_reached`,
  as "AI is off until next month": return `503` so the frontend hides the Ask
  box, and log it once.
- Don't retry those errors. Retrying can't succeed until the limit resets, so
  turn off the SDK's automatic retries for them.
- Ordinary `429`s with a `retry-after` header are normal rate limits; return
  `429` to the client.

Sources: Anthropic docs on
[rate limits and spend limits](https://platform.claude.com/docs/en/api/rate-limits)
and [workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces).

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
- App-wide: a daily cap, so one bad day can't use the whole month's budget.
  Once it's hit, return `503` until the next day. The hard monthly backstop is
  the Console spend limit ([API access and spend cap](#api-access-and-spend-cap)).

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
