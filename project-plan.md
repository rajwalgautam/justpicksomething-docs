# Just Pick Something — Project Plan

A "why don't you just pick" app: someone creates a **room**, shares a link +
passcode, and everyone in the room drops **items** in, compares them across
custom **columns**, and **votes** until the group lands on a pick.

> Example: planning a group vacation. Each item is an Airbnb/VRBO link; columns
> are `link`, `bedrooms`, `bathrooms`, `hot tub (yes/no)`; everyone votes.

**Strategy:** get one thin slice working end to end, deployed on the real
domains, before adding breadth. Every milestone after that ships to production
on its own.

---

## Repos

| Repo | Purpose | Deploys to |
| --- | --- | --- |
| [`Gauteam/justpicksomething-frontend`](https://github.com/Gauteam/justpicksomething-frontend) | TypeScript + React UI | `pick.rajwalgautam.com` (Vercel) |
| [`Gauteam/justpicksomething-backend`](https://github.com/Gauteam/justpicksomething-backend) | Go HTTP API (Vercel Go functions) + SQL migrations | `api.pick.rajwalgautam.com` (Vercel) |
| [`rajwalgautam/justpicksomething-docs`](https://github.com/rajwalgautam/justpicksomething-docs) | This plan, API contract, decisions | — |

The two code repos are **public**: Vercel Hobby won't deploy private repos owned
by a GitHub organization. Secrets live only in Vercel env vars and git-ignored
`.env` files, never in the repos.

Use the workout builder (`workouts-backend`, `workouts-ui`) as the reference for
Vercel + Go + Neon + Next.js setup, with the changes noted under
[Lessons from the workout builder](#lessons-from-the-workout-builder).

## Tech stack

- **Frontend:** Next.js (App Router) + TypeScript + React, Tailwind. Same
  toolchain as `workouts-ui`. The UI talks to the API only over HTTP — no Next
  API routes or server actions, so the backend stays the single source of truth.
- **Backend:** Go on Vercel serverless functions. One entry point
  (`api/index.go`) with a `net/http` `ServeMux` (Go 1.22+ path patterns, e.g.
  `GET /rooms/{slug}`), and a `vercel.json` rewrite sending every path to it.
  This avoids one-file-per-endpoint sprawl and query-param routing.
- **Database:** Neon Postgres via `pgx/v5` `pgxpool` (small pool, serverless
  tuning as in `workouts-backend/pkg/db/db.go`). Migrations with
  `golang-migrate`, same `Makefile` pattern.
- **Realtime:** polling, within the [polling budget](#polling-budget). Vercel
  functions don't hold WebSockets well, and long-lived connections would burn
  free-tier execution time, so polling it is.

## Constraints

### Free tier only

Every service stays on its free plan: Vercel Hobby, Neon Free, GitHub Free,
Cloudflare Turnstile (free) if needed. The only paid piece is the domain
(`rajwalgautam.com`, already owned); `pick.` and `api.pick.` are subdomains
pointed at Vercel with DNS records at no extra cost. What that means for the
design:

- **Vercel Hobby is for personal, non-commercial use.** No ads or paid features
  unless we move to a paid plan.
- **Function invocations and execution time are capped per month.** Polling is
  the main cost, so it must be conservative (see [Polling budget](#polling-budget)).
- **Neon Free has a small storage cap and limited compute hours, and its compute
  suspends when idle.** The first request after a quiet period is a cold start
  (roughly a second or more), and every request wakes the compute and uses up
  hours. So: no background jobs that keep the database awake, and delete old
  rooms (M5) to stay under storage.
- **No paid add-ons:** no Vercel KV/Redis or paid firewall features. Rate
  limiting uses a Postgres table.
- Check current limits on each provider's pricing page before relying on a
  specific number. They change, so this plan doesn't hardcode them.

### Polling budget

- Poll `GET /rooms/{slug}` only while the tab is visible (`document.visibilityState`).
- Start at 5s; back off to 30s after a couple of minutes with no changes; reset
  on any local action.
- The response includes an `updatedAt` per room. The client sends
  `?since=<updatedAt>` and gets a tiny `200 {"unchanged":true}` when nothing
  changed. Not `304`: Vercel strips CORS headers from 304 responses, so
  browsers reject them. The database lookup still happens, though.
- If the free limits are hit anyway, the fallback is a manual "refresh" button,
  not a paid plan.

### API access: HTTPS, only from `pick.rajwalgautam.com`

- **HTTPS only.** Vercel serves custom domains over HTTPS with automatic
  certificates and redirects HTTP to HTTPS. The API also sends
  `Strict-Transport-Security: max-age=31536000; includeSubDomains`. As a guard,
  it rejects any request whose `X-Forwarded-Proto` isn't `https` (skipped
  locally).
- **Only the frontend's origin.** Two layers in the router middleware:
  1. **CORS:** `Access-Control-Allow-Origin: https://pick.rajwalgautam.com`
     exactly (never `*`). Preflight `OPTIONS` from any other origin gets `403`.
  2. **Origin check:** every non-`GET` request must carry
     `Origin: https://pick.rajwalgautam.com`, or it gets `403`. `GET` requests
     must either carry that origin or have no `Origin` header (some browsers omit
     it on same-site `GET`s).
- **Allowed origins come from an env var** (`ALLOWED_ORIGINS`, comma-separated):
  - Production: `https://pick.rajwalgautam.com`
  - Local dev: `http://localhost:3000` (set only in the local `.env`, never in
    Vercel Production)
  - Vercel Preview: leave unset, or set it to the preview frontend's URL if we
    want preview deployments to work end to end.
- **What this does not do:** the `Origin` header only constrains browsers. Any
  script or `curl` can set it to anything. It stops other websites from using
  the API from their visitors' browsers, but it doesn't stop direct API calls.
  The real protection is the room passcode, member tokens, and rate limits
  (M5). Don't treat the origin check as authentication.

## Key decisions

- **No user accounts.** Identity is per-room. Joining a room (slug + passcode +
  display name) returns a random **member token**; the frontend stores it in
  `localStorage` keyed by room and sends it as `Authorization: Bearer <token>`.
  The server stores only a hash of the token.
- **Room creator is admin.** Admins can change settings, lock voting, and
  promote others.
- **Passcode is auto-generated** (e.g. 6 characters, no ambiguous letters), with
  an optional override field at creation. Stored hashed (bcrypt).
- **Room link** is `pick.rajwalgautam.com/r/{slug}`, where `slug` is a short
  random base62 ID (not the room name), so rooms aren't guessable.
- **Item column values live in JSONB** (`items.values`, keyed by column ID).
  Columns are user-defined per room, so a fixed schema doesn't fit; JSONB keeps
  "add a column" a single-row insert.

## Data model (target)

Built up across milestones; M1 only needs the starred fields.

```
rooms        id*, slug*, name*, passcode_hash*, created_at*,
             max_items, vote_mode ('thumbs' | 'budget'), votes_per_member,
             allow_duplicate_votes, everyone_admin, locked_at
members      id*, room_id*, display_name*, token_hash*, is_admin*, created_at*
columns      id, room_id, name, type ('text' | 'boolean' | 'textarea'), position
items        id*, room_id*, title*, fields (jsonb), position, created_by*, created_at*
votes        id*, room_id*, item_id*, member_id*, value* (+1 / -1 in thumbs mode;
             1..N votes placed on the item in budget mode), created_at*, updated_at*
comments     id, room_id, item_id, member_id, body, created_at, edited_at
```

Constraints worth encoding in SQL: `unique(room_id, display_name)` on members;
`unique(item_id, member_id)` on votes in every mode (one row per member per
item; in budget mode its `value` is how many votes they've stacked there, and
the per-member budget is checked in code). Comments get an index on
`(item_id, created_at)` and `on delete cascade` from items and members.

---

## Milestones

### M0 — Walking skeleton (infra only) ✅ Done 2026-10-29

Prove the pipes work before writing features.

- [x] Backend: Go module, `api/index.go` router, `GET /health` that runs
      `SELECT 1` against Neon. `DATABASE_URL` read from the environment.
- [x] Backend: `vercel.json` rewrite, deployed, `api.pick.rajwalgautam.com`
      attached.
- [x] Backend: middleware implementing
      [API access](#api-access-https-only-from-pickrajwalgautamcom) — HTTPS
      check, HSTS, CORS and origin check driven by `ALLOWED_ORIGINS`. Do this in
      M0 so every later endpoint gets it for free.
- [x] Neon project + `DATABASE_URL` set in Vercel (Production + Preview) and a
      local `.env` (git-ignored). First migration creates an empty schema.
- [x] Frontend: Next.js app scaffolded, `NEXT_PUBLIC_API_URL` env var, home page
      shows the `/health` result. Deployed, `pick.rajwalgautam.com` attached.

**Done when:** visiting `pick.rajwalgautam.com` shows "API: ok" fetched live
from `api.pick.rajwalgautam.com/health`, and:

- `curl -i http://api.pick.rajwalgautam.com/health` redirects to HTTPS.
- `curl -i -X POST -H "Origin: https://evil.example" https://api.pick.rajwalgautam.com/health`
  returns `403`.
- A `fetch` to the API from a page on any other origin is blocked by CORS.

Verified on production 2026-10-29: `/health` → `200 {"db":"ok"}` with HSTS;
`http://` → `308` to HTTPS; `POST` and `GET` from another origin → `403`;
preflight from the frontend → `204` allowing only `https://pick.rajwalgautam.com`;
unknown path → `404`; the site shows "API: ok".

### M1 — Vertical slice: create → join → add → vote

The smallest version of the whole product. Items are just a title; voting is
thumbs up/down only; no settings.

Backend endpoints:

| Method | Path | Notes |
| --- | --- | --- |
| `POST` | `/rooms` | body `{name, displayName}` → `{slug, passcode, token}`; creator becomes admin member |
| `POST` | `/rooms/{slug}/join` | body `{passcode, displayName}` → `{token}` |
| `GET` | `/rooms/{slug}` | auth; returns room, members, items with vote tallies and the caller's own vote |
| `POST` | `/rooms/{slug}/items` | auth; body `{title}` |
| `PUT` | `/rooms/{slug}/items/{id}/vote` | auth; body `{value: 1 \| -1}` sets the caller's vote |
| `DELETE` | `/rooms/{slug}/items/{id}/vote` | auth; clears the caller's vote |

Votes use `PUT` with the absolute value, not "add a vote", so a retried or
double-sent request can't count twice.

Frontend pages:

- `/` — "Create a room" form (room name, your name). On success, show the share
  link + passcode with copy buttons, then go to the room.
- `/r/[slug]` — if no token for this room, show join form (passcode, name).
  Otherwise show the item list with 👍/👎 counts, your vote highlighted, an
  "add item" input, sorted by score. Poll `GET /rooms/{slug}` per the
  [polling budget](#polling-budget).

**Done when:** on production, person A creates a room, sends the link + passcode
to person B on another device, both add items and vote, and each sees the
other's changes within a few seconds.

### M2 — Columns, comparison, and comments

- [ ] Admin can add/rename/delete columns: `text` (single line), `boolean`,
      `textarea`.
- [ ] Items get editable values per column (stored in `items.fields`).
- [ ] Room view becomes a comparison table (items as rows, columns as columns),
      with a card layout on mobile.
- [ ] Drag to reorder columns and items (`@dnd-kit`, as in `workouts-ui`);
      persist `position`.
- [ ] Text values that look like URLs render as links.
- [ ] **Sort items by any column** by clicking its header: ascending, then
      descending, then back to manual (drag) order. Also sort by score, title,
      and date added.
  - `text`: case-insensitive natural sort, so `2` sorts before `10` (handles
    "bedrooms" typed as text without needing a number type yet).
  - `boolean`: yes first when ascending. `textarea`: not sortable.
  - Empty values always sort last.
  - Sorting runs in the browser on data it already has, so it costs no API
    calls. Each viewer's sort is their own (kept in `localStorage`), not shared
    with the room. Drag-to-reorder is disabled while a sort is active.

#### Comments

Basic, flat comment threads on each item. No replies, reactions, mentions, or
formatting; plain text, with URLs shown as links.

- [ ] Any member can comment on an item while the room is unlocked. Max 1,000
      characters per comment.
- [ ] Authors can edit and delete their own comments; admins can delete any.
      Edited comments show "(edited)".
- [ ] Comment count shows on each item row/card. Clicking it opens the thread
      (side panel on desktop, sheet on mobile), oldest first.
- [ ] Removing a member or item deletes their comments.

Endpoints:

| Method | Path | Notes |
| --- | --- | --- |
| `GET` | `/rooms/{slug}/items/{id}/comments` | auth; full thread |
| `POST` | `/rooms/{slug}/items/{id}/comments` | auth; body `{body}` |
| `PUT` | `/rooms/{slug}/comments/{commentId}` | auth; author only; body `{body}` |
| `DELETE` | `/rooms/{slug}/comments/{commentId}` | auth; author or admin |

Free-tier fit: `GET /rooms/{slug}` returns only a `commentCount` per item, not
comment bodies, so the polled payload stays small. The thread is fetched when
opened and refreshed by the same polling loop only while it's open. Comments
update the room's `updatedAt`, so other members' counts update on their next
poll.

### M3 — Room settings and voting modes

- [ ] Settings panel (admin only), editable after creation:
  - room name
  - max number of items (empty = no limit; can't go below the current count)
  - vote mode: **thumbs** (up/down per item) or **budget** ("X votes out of Y":
    each member gets N votes to spread, 1–100)
  - allow duplicate votes on one item (budget mode only)
- [ ] Budget mode reuses the vote endpoint: `PUT .../vote` with `{count: n}`
      sets how many of your votes are on that item (still idempotent);
      `count` > 1 only when duplicates are allowed; the server rejects totals
      over the member's budget.
- [ ] **Lock in:** admin locks the room → options, votes, columns and comments
      are frozen, and the top pick (or a tie) is highlighted. Admin can unlock.
      Settings can't change while locked.
- [ ] Backend enforces all of the above (not just the UI).
- [ ] "Add several at once" becomes a table with the room's columns; rows can
      be pasted from a spreadsheet (tab-separated).

### M4 — Admins and members

- [ ] Member list in the room; admin can promote/demote admins and remove members.
- [ ] Room setting "everyone is admin" (default on for small casual rooms).
- [ ] Admin can regenerate the passcode (existing members stay joined).

### M5 — Hardening

- [ ] Rate-limit room creation and join attempts per IP with a Postgres counter
      table (free; no Redis/KV). Add Cloudflare Turnstile (free) to the create
      form if that's not enough. Answers "how do I stop people spamming room
      creation". Also cap total rooms per day as a backstop for the Neon
      storage limit.
- [ ] Input validation and size limits on every write.
- [ ] Go tests for handlers against a Neon branch or local Postgres; GitHub
      Actions running `go test` and the frontend `lint` + `build`.
- [ ] Expire/delete rooms inactive for N days. Run the cleanup lazily inside
      `POST /rooms` (delete a small batch of expired rooms), or as a daily
      Vercel cron if the Hobby plan's cron allowance permits. Keeps the database
      under Neon's free storage cap.

### Later — "down the line" ideas

- **Templates** (e.g. "Vacation rental" with link/bedrooms/bathrooms/hot tub
  pre-defined). Start with a few built-in ones; a separate templates tab and
  user-submitted templates after that. Rooms created from a template can still
  add and edit columns.
- **Paste enrichment:** pasting an Airbnb or VRBO link pre-fills columns. Needs
  research — neither offers a public listing API, and scraping is fragile and may
  break their terms, so start with Open Graph tags (title, image) only.
- Realtime upgrade (SSE or hosted pub/sub) if polling isn't enough.
- **mTLS for API connections.** Require a client certificate on connections
  to `api.pick.rajwalgautam.com`, so only our own code can reach it. Needs
  design first:
  - Today the browser calls the API directly, and every visitor's browser would
    need its own installed client certificate, which isn't workable. mTLS only
    fits if browsers stop calling the API directly: the Next.js server (or a
    proxy) calls the API with one client certificate, and the browser talks
    only to the Next.js server.
  - That moves API traffic through Vercel functions on the frontend project
    too, roughly doubling function invocations against the free-tier limits.
  - Check whether Vercel can enforce client certificates on a custom domain
    at all (not confirmed). If not, the API needs a host or front proxy that
    can, e.g. Cloudflare's mTLS rules.
  - Until then, the protections are HTTPS, the origin check, room passcodes,
    member tokens and the M5 rate limits.

---

## Lessons from the workout builder

- `workouts-backend/pkg/db/db.go` has the Neon connection string hardcoded and
  committed. Here, read `DATABASE_URL` from the environment from day one and keep
  `.env` files out of git.
- `workouts-backend` routes by query param (`?id=`) with one function per
  resource. The single-router approach above gives real REST paths.
- CORS there is `*`; here, only `https://pick.rajwalgautam.com` (see
  [API access](#api-access-https-only-from-pickrajwalgautamcom)).

## Decisions log

- **2026-10-29 — Vote API:** `PUT` with an absolute value (`{value: ±1}`, later
  `{count: n}`), `DELETE` to clear. Idempotent, so retries are safe.
- **2026-10-29 — Rejoining:** display names are unique per room. Joining from a
  new device means picking a new name. Revisit in M4.

- **2026-10-29 — Bulk add (added to M1):** paste options separated by new
  lines or commas; one request, max 50, duplicates (ignoring case) skipped.
- **2026-10-29 — M2 permissions:** only admins add/rename/delete/reorder
  columns and set the manual item order, since those are shared by everyone.
  Any member edits item titles and values.
- **2026-10-29 — Default sort:** most votes first. "Manual order" is one of the
  sort options; dragging only works in it.
- **2026-10-29 — Switching vote mode clears all votes:** thumbs votes (±1) and
  budget votes (counts) don't convert into each other. Turning off duplicate
  votes caps everyone at one vote per option. Lowering the budget below what
  someone has used keeps their votes; they just can't add more until they take
  some back.
- **2026-10-29 — Bulk add is a table:** with columns in the room, a list of
  titles isn't enough, so bulk add takes rows with column values. Pasted text
  splits on tabs and new lines only (not commas), so titles can contain commas.
- **2026-10-29 — `items.fields`:** column values live in `fields`, not
  `values` (a reserved word in SQL).

## Open questions

- Do non-admins get to add columns, or only admins? (Plan assumes admins only.)
- Should vote tallies be hidden until the room is locked, to avoid
  bandwagoning? Could be a room setting in M3.
