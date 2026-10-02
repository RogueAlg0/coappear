# Presence Layer: Scope and Architecture Plan

Status: draft plan for maintainer review. The v1 scope in section 2 is locked.
Open decisions are collected in section 9; nothing else in this doc should be
treated as decided until the maintainer signs off.

## 1. Pitch

Any website becomes a shared place with one script tag. Visitors appear as
live cursors, each carrying a warm, empowering, randomly assigned name like
"Brave Sparrow" (no real identity, just a kind stranger), plus a visitor
count, and anyone can pin
shared note boxes onto the page that everyone else sees instantly. No signup,
no dashboard, no accounts: the site owner pastes a snippet, and presence is
just there, the way Figma cursors or a whiteboard feel alive. The layer is
invisible by design: no chrome, no branding, no UI to learn, it should feel
like a natural property of the host page.

One-line description: presence-as-a-script-tag, live cursors and
shared notes for any website, with zero signup.

## 2. v1 scope (locked)

- Product: "presence-as-a-script-tag". One tiny `<script>` embed; visitors see
  live cursors plus a visitor count, and can create shared note
  boxes visible to everyone on that site. Each visitor is assigned an
  empowering, warm, neutral random name (adjective plus noun, e.g. "Brave
  Sparrow", "Kind Lighthouse") shown on their cursor: anonymous in identity,
  human in feel. Names are ephemeral, assigned per connection, never stored.
- No signup, no dashboard, no accounts. Per-site namespaces keyed by domain.
- 5 notes per visitor. Big notes supported: per-note cap proposed at 100KB
  (see section 3 for justification).
- Strict profanity filter, including lookalike and unicode-trick normalization.
- Per-site persistent board state in SQLite (single file). Reloading restores
  the last state.
- Privacy-first: no cookies, no accounts, no stored IPs. Rate limits live in
  memory only, never persisted.
- Invisible by design (core principle): minimal visuals, no chrome. The layer
  should feel like a natural property of the host page, intuitive with no UI
  to learn.
- Single small server plus static frontend. Must run comfortably on a free
  tier.
- Public repo, MIT license, launched with the maintainer's standard playbook:
  README with live demo and GIF, CI green from day one, branch protection,
  issue templates, contributing guide, OpenSSF best-practices badge,
  Scorecard.

Explicit non-goals for v1:

- Accounts, profiles, or persistent identity of any kind. (Ephemeral assigned
  names are labels, not identity: they are random per connection and never
  stored.)
- Direct messages between visitors.
- Note edit history or versioning (last-write-wins only).
- Images, file uploads, or rich media in notes (plain text only).
- A moderation UI. Emergencies are handled by a secret-key admin wipe
  endpoint that clears a site's board.
- Per-note ownership or permissions. The board is a public whiteboard: any
  visitor can move, edit, or delete any note. This is a deliberate
  simplification, not an oversight. It removes the need for identity,
  sidesteps attribution privacy questions, and matches the invisible-by-design
  goal. If abuse shows this was wrong, ownership is a v2 discussion.

## 3. Architecture

### 3.1 Embed script design

Two parts. The site owner pastes an inline loader snippet (target: under
1KB) that injects the full client asynchronously with `async` and `defer`
semantics, so it never blocks the host page. The full client is vanilla JS,
zero dependencies.

Size budget proposal: **20KB gzipped** for the full client bundle, enforced
in CI with a build step that fails the build if the bundle exceeds it.
Justification: this puts the client in the same weight class as a typical
analytics snippet, which site owners already accept without thinking. At
20KB gzipped, transfer time is well under 100ms on ordinary broadband and the
parse cost is negligible next to any modern framework. A dependency-free
client (websocket handling, cursor rendering, note overlay CRUD, reconnect
logic) fits comfortably in this budget; anything larger suggests a framework
crept in and should be questioned. Keeping the client small also keeps
free-tier bandwidth trivial even if many sites embed it.

Client responsibilities:

- Open one websocket per page load to the presence server.
- Render remote cursors as small, unobtrusive markers that fade after a few
  seconds of inactivity. No names, no labels, no avatars (there is no
  identity to show).
- Show a small visitor count. Placement should be ambient (a corner dot with
  a number), never a panel.
- Note overlay: double-click (or a documented gesture) creates a note box;
  drag to move, type to edit, small affordance to delete. Autosize with a
  large maximum; styling is deliberately plain so notes feel native to any
  host page.
- Throttle outbound cursor messages (suggest: max 10 per second, and only
  when the pointer actually moved beyond a small delta).
- Reconnect with backoff (see 3.4).

### 3.2 Websocket protocol sketch

Transport: `wss`, one connection per page load. The site namespace comes from
the page origin, validated server-side against the `Origin` header and the
embedding site's registered domain. All messages are JSON objects with a
`type` field. Payloads stay small; cursor messages are a few dozen bytes.

Client to server:

- `join`: `{ site }`. First message on a new connection. Server responds with
  a snapshot.
- `cursor`: `{ x, y }`. Throttled client-side. Coordinates are fractions of
  the viewport (0 to 1), so cursors stay meaningful across different screen
  sizes.
- `note.create`: `{ id, x, y, w, h, text }`. `id` is client-generated
  (UUID); geometry is viewport fractions; text capped at 100KB.
- `note.update`: `{ id, x, y, w, h, text }`. Full replacement, not a diff.
- `note.delete`: `{ id }`.
- `ping`: keepalive.

Server to client:

- `welcome`: `{ visitorId, count, notes }`. The full board snapshot plus the
  caller's ephemeral id. Sent once after `join`.
- `presence`: `{ count }`. Broadcast on every join and leave.
- `peer.join` / `peer.leave`: `{ visitorId }`. Lets clients add or remove a
  cursor marker.
- `cursor`: `{ visitorId, x, y }`. Relayed, never stored.
- `note.created` / `note.updated` / `note.deleted`: the note payload, or the
  id for deletes. Broadcast to the whole room.
- `error`: `{ code, message }`. Codes include `rate_limited`,
  `quota_exceeded`, `profanity_blocked`, `payload_too_large`,
  `invalid_message`. The client surfaces these quietly (a note that failed
  validation simply does not appear, plus a subtle inline hint where the
  note was being composed).
- `pong`: keepalive reply.

Concurrency rule: last-write-wins by server receive timestamp. No operational
transform, no conflict UI in v1. Simultaneous edits to the same note resolve
to whichever update the server processed last, and every client converges on
the broadcast.

### 3.3 Server design

Single process, single writer. Language is not prescribed by this plan, but
the shape is: an event loop holding all rooms in memory, one SQLite
connection in WAL mode. One process means no distributed state, no lock
contention on the database, and trivial deployment. Horizontal scaling is
explicitly out of v1; if one box stops being enough, that is a good problem
and a v2 redesign.

Per-site rooms: an in-memory map keyed by normalized domain. Each room holds
the set of live connections, each tagged with an ephemeral visitor id
(random, per connection) and a small rate-limit state. Rooms are created on
first join and can be dropped from memory when empty; the board state itself
lives in SQLite, so dropping a room loses nothing.

SQLite schema sketch:

- `sites`: `id` (integer primary key), `domain` (text, unique),
  `created_at`. One row per embedding site, created lazily on first join.
- `notes`: `id` (text primary key, client UUID), `site_id` (foreign key),
  `x, y, w, h` (reals, viewport fractions), `text` (text, max 100KB
  enforced before insert), `created_at`, `updated_at`. Every create, update,
  and delete writes through to disk immediately, so a server restart loses
  at most in-flight, unacknowledged messages.
- A per-site note cap (propose 500 notes per site, open for confirmation in
  section 9): when the cap is hit, the oldest note is evicted on the next
  create. This bounds database growth and broadcast snapshot size without any
  background jobs.

Per-note cap justification (100KB): the locked scope says big notes are
supported, so the cap must be generous, but unbounded text breaks three
things: profanity-filter CPU time (the filter scans the whole note on every
update), snapshot broadcast size on every new join, and database growth from
a single abusive client. 100KB is roughly fifty printed pages of text, far
beyond any legitimate sticky note, while keeping worst-case filter and
broadcast cost trivial. Typical notes will be under 1KB, so this cap should
never bind real use. It is a guardrail, not a feature limit.

5-notes-per-visitor enforcement: quota is tracked per live connection against
the ephemeral visitor id, in memory only. This is the honest tradeoff of the
privacy model: because there are no cookies or accounts, a visitor who
reconnects gets a fresh quota. Accept it (see section 5), document it, and
rely on the per-site note cap and the admin wipe endpoint as the backstops.

Admin: one secret-key endpoint (key from environment variable, never in the
repo) that wipes a single site's board: `POST /admin/wipe { site, key }`.
No listing endpoint, no other admin powers in v1.

### 3.4 Reconnection behavior

The client reconnects with exponential backoff and jitter (suggest 1s, 2s,
5s, 10s, capped at 30s). On rejoin the server sends a fresh `welcome`
snapshot, so the client always converges to current board state; there is no
attempt to replay missed cursor movements (cursors are ephemeral by design)
or missed note edits (last-write-wins plus snapshot covers it). A reconnect
gets a new ephemeral visitor id and a fresh note quota, which is the accepted
quota tradeoff noted above. The server detects dead connections with
ping/pong timeouts (suggest 30s without a pong) and broadcasts
`peer.leave` so stale cursors disappear promptly.

## 4. Privacy model

What is collected, and where it lives:

- Ephemeral visitor id: random per connection, held in server memory only,
  discarded on disconnect. Used for cursor attribution within a session and
  for the per-connection note quota. Never written to disk, never logged.
- Cursor positions: relayed to other visitors in the same room, never stored,
  never logged.
- Note text and geometry: persisted in SQLite per site. This is the product
  (a persistent shared board), and it is anonymous: no author, no IP, no
  user agent is stored alongside notes.
- Rate-limit counters: in-memory token buckets per connection. IP addresses
  may be used transiently as a key for a per-IP connection cap, but they are
  never persisted and never written to logs.

What never touches disk:

- IP addresses, user agents, cursor trails, visitor ids, rate-limit state.
  Server access logging must be disabled or piped to null; a default web
  server access log would silently violate this model, so the deployment
  checklist needs an explicit log-suppression step.
- Cookies are never set. Local storage is not used for identity.

Residual privacy notes, stated plainly: note content is visible to anyone who
visits the site, by design (it is a public whiteboard). TLS protects data in
transit. The server operator can read the SQLite file; self-hosting is the
answer for anyone who does not trust the public instance, and the README
should say so.

## 5. Abuse model

Threats, mitigations, and accepted residual risk:

1. Profanity filter bypass (unicode lookalikes, zero-width characters,
   leetspeak, splitting a word across two updates). Mitigation: normalize
   before matching (section 6), re-filter the full note text on every create
   and update, strict block on any match. Residual: novel slurs not in the
   list will pass until the list is updated; this is an arms race and the
   blocklist will never be complete. Accepted.
2. Spam floods (connection floods, message floods). Mitigation: per-IP cap
   on concurrent connections, per-connection token buckets (suggest: cursor
   messages 10/sec, note operations a handful per minute), maximum websocket
   frame size with immediate close on violation. Residual: slow, distributed,
   low-rate spam looks like normal use. Accepted; the wipe endpoint exists
   for visible damage.
3. Note stuffing (many large notes). Mitigation: 5-note quota per connection,
   100KB per-note cap, per-site total note cap with oldest-first eviction.
   Residual: reconnecting resets the per-connection quota, so a determined
   actor can fill a site's board up to the site cap. The site cap bounds the
   damage and the wipe endpoint clears it. Accepted as the price of having no
   identity.
4. Websocket abuse (malformed JSON, schema violations, slow reads holding
   connections). Mitigation: strict message validation, drop and close on
   first violation, ping/pong timeouts, read timeouts. Cheap to implement,
   do it from the start.
5. Hostile embedding (someone puts the tag on a shock or spam site).
   Mitigation: namespaces are keyed by domain, so damage is contained to
   that site's board; there is no central directory of sites in v1, so there
   is nothing to deface globally. The operator can wipe or block a domain at
   the server level. Residual: the public demo instance could get
   embarrassing boards; the wipe endpoint plus a small domain blocklist
   covers it.
6. Data theft or deanonymization. There is nothing to steal: no accounts, no
   IPs, no cursor history. Note text is public by design. This is the payoff
   of the privacy model and should be stated as a feature.

What remains accepted risk, in one place: no identity means no bans, only
rate limits and wipes; quotas reset on reconnect; the filter is blocklist
based and will miss novel terms. All three are conscious prices paid for
privacy and invisibility, and the doc should keep saying so rather than
letting them become surprises.

## 6. Profanity filter approach

Strict blocklist, deny on any match. No machine learning in v1: a blocklist
is deterministic, auditable, fast, and its failures are legible (a missed
word is a missing list entry, not a model mystery).

Normalization pipeline, applied identically on every create and update
before matching:

- Unicode NFKC normalization, then a confusable mapping that folds
  lookalike codepoints (Cyrillic, Greek, and other script lookalikes for
  Latin letters) to their ASCII equivalents.
- Strip zero-width and invisible characters, strip or fold combining marks,
  casefold, collapse whitespace and punctuation between letters so that
  spaced-out or punctuated insertions still match.
- Match against the blocklist with word-boundary awareness to avoid false
  positives on innocent substrings.

Performance notes: compile the list once at startup into a single efficient
matcher; the scan is one pass over at most 100KB of text, which bounds worst
case cost and keeps per-note check time in the low milliseconds on modest
hardware. Filtering runs server-side as the source of truth; the client may
run the same check locally for instant feedback while typing, but the server
re-checks everything and its verdict wins.

The doc and the repo must never contain actual profane words, including in
examples or tests. Tests should use obvious placeholders and synthetic
evasion patterns (for example, a placeholder token with zero-width
characters inserted), never real terms.

Open: the source of the word list (section 9). Whatever the source, check
its license before vendoring, and keep the list in a separate file so it can
be updated without touching code.

## 7. Hosting

Constraints: one small process, one SQLite file on a persistent volume,
websocket-friendly (long-lived connections, so no platform that kills idle
connections aggressively), near zero cost.

- fly.io (recommended for the public demo): at time of writing, the free
  allowance covers a small shared VM plus a persistent volume and modest
  outbound transfer, which fits this project exactly: one VM, one 1GB
  volume for the SQLite file, websockets work natively. Verify current
  terms before launching; free tiers change.
- render: the free web-service tier sleeps after inactivity and, more
  importantly, its free disks are ephemeral, so the SQLite board would not
  survive restarts. Not recommended for the persistent demo unless on a paid
  tier.
- A cheap VPS (for example, Hetzner-class, a few euros a month) as the
  non-free fallback: full control over logs (needed for the privacy model),
  trivial SQLite backups by copying the file.

Why SQLite fits: the entire dataset is per-site note boards, tiny by any
measure; a single writer process means no lock contention; WAL mode gives
crash safety; backup is a file copy; there is nothing to configure, patch,
or pay for. Postgres would add operational cost for zero v1 benefit.

## 8. Repo-launch checklist

Follow the maintainer's standard playbook, in this order:

- MIT license file, plus a short privacy note in the README stating what is
  stored (anonymous note text per site) and what is not (no cookies, no IPs,
  no accounts).
- README structure: one-line description, live demo link with an animated
  GIF of cursors and notes in action, the embed snippet (copy-paste ready),
  self-hosting instructions (single binary or container plus the SQLite
  file), privacy summary, link to this scope doc.
- CI green from day one: lint, format check, unit tests for the filter and
  protocol handling, the 20KB client bundle budget check, and a smoke test
  that boots the server and round-trips a note over a real websocket.
- Branch protection on main: PR required, no direct pushes, no force pushes,
  required CI checks.
- Issue templates (bug report, feature request) and a pull request template.
- Contributing guide: how to run locally, the no-profanity-in-tests rule,
  the bundle budget rule, DCO or signed commits per maintainer preference.
- OpenSSF best-practices self-certification, badge in README.
- Scorecard workflow, with findings triaged (token permissions, pinned
  dependencies, branch protection checks).
- Live demo instance on the chosen free tier, with the admin wipe key set
  from the environment and access logging disabled.

## 9. Open decisions for the maintainer

1. Project name. 25 candidates were checked against npm, GitHub, and general
   web results; the namespace is crowded and most good names are taken.
   Five proposals, with honest collision notes:
   - `coappear`: clean, no package or repo found under this name (only the
     ordinary English word appearing in docs). Slightly technical, but
     descriptive of shared presence.
   - `commonsroom`: cleanest compound found; only weak hits (a dormant
     strategy doc, a decades-old newspaper mention). Evokes a shared public
     room. Longer to type.
   - `othercursors`: descriptive and on the nose for the core visual; only
     weak hits (demo component names, a university exercise repo). Less
     brandable, very literal.
   - `hearthside`: warm and memorable, but weak collisions exist (a care
     app, an agent cookbook). Usable, not clean.
   - `murmuration`: evocative (starling flocks moving as one), but
     crowded-ish (music visualizers, swarm sims). Least recommended of the
     five.
   Recommendation: pick from the top three, then do a final check of npm,
   PyPI, GitHub org names, and a trademark search before locking it.
2. Per-note size cap confirmation: 100KB proposed, justification in section
   3.3. Confirm or adjust.
3. Filter list source: vendor a license-compatible open list, hand-curate
   from scratch, or start from an open list and curate additions. Check the
   license before vendoring; keep the list in its own file.
4. Board reset and expiry policy: do boards live forever until wiped, or
   should untouched sites expire after N days of no visits? Expiry bounds
   disk growth and stale content; permanence is simpler to explain. Decide
   before launch, since changing it later surprises site owners.
5. Admin wipe key distribution: environment variable on the host is enough
   for one operator; if more people need it, decide how the key is shared
   and rotated without ever landing in the repo.

## 10. Milestone sketch (v1 build order)

- M1: Protocol and server skeleton. Websocket server, per-site rooms,
  join/leave/presence/cursor relay, ephemeral ids, ping/pong. No notes yet.
- M2: Embed client. Loader snippet, cursor rendering, visitor count,
  throttling, reconnect with backoff. Bundle budget check in CI from this
  point on.
- M3: Notes. Client overlay (create, move, edit, delete), server broadcast,
  SQLite persistence with the schema in 3.3, snapshot on join.
- M4: Safety. Profanity filter with the normalization pipeline, rate limits,
  quotas, per-site note cap, admin wipe endpoint, access-log suppression.
- M5: Invisible-design pass. Strip every visual down until the layer feels
  native to arbitrary host pages; test the embed on several real sites for
  style collisions (namespaced CSS, shadow DOM if needed).
- M6: Launch readiness. Demo site, README with GIF, issue templates,
  contributing guide, CI hardening, OpenSSF self-cert, Scorecard triage,
  branch protection, live demo instance on the free tier.
- M7: Public launch.

Suggested working order within each milestone: protocol shape first, then
server, then client, then tests. Keep the repo public from the first commit
so the launch playbook items (CI, templates, badges) accumulate naturally
instead of landing as a big bang.
