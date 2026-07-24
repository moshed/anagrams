# Anagrams — live word-stealing game

A single-file, static web game of **Anagrams** (a.k.a. Snatch / Word Stealing).
Everyone plays on their own phone against one shared table. Everything lives in
`index.html` (HTML + CSS + vanilla JS, no build step) — same shape as
`/Users/moshe/Apps/spellingbee`.

Live: **https://anagrams.dancykier.com** · Repo: `moshed/anagrams` (public,
GitHub Pages from `main` root). Deploy = edit `index.html` + `git push`.

## Files
- `index.html` — the whole game (home screen, table, rules engine, sync layer).
- `words.json` — 115k-word dictionary, copied verbatim from the spellingbee
  repo (SCOWL-derived, a–z, length ≥ 4 — which is exactly the min word length
  here). Fallback URL in `loadDict()` is the jsDelivr copy of the same file in
  `moshed/spellingbee`.
- `CNAME` — `anagrams.dancykier.com`.
- `og.png` — 1200×630 social card (tiles spelling ANAGRAMS on green felt),
  generated with the PIL snippet in git history.

## The rules as implemented
- **Bag** = the Bananagrams distribution, 144 tiles (`TILE_COUNTS`), shuffled
  at game start. Anyone can hit **Flip** to turn one tile face-up into the
  middle (500 ms global cooldown, blocked while a claim/penalty is live).
- **Min word length 4** (`MIN_LEN`).
- **Calling a word** (`I HAVE A WORD!`) takes an exclusive lock on the table:
  `state.claim = {pid, start, deadline}`. Everyone else's buttons go dead —
  that's the digital stand-in for shouting first.
- **Timer**: 5 s to start typing (`CLAIM_START`), **+2 s per letter typed**
  (`CLAIM_PER`), hard ceiling 30 s per claim (`CLAIM_CAP`). Only *growth* of
  the input adds time (backspacing then retyping doesn't farm seconds).
- **Making a word**: either it's spelled from the middle tiles alone, or it's a
  **steal** — use **all** letters of a word already on the table (anyone's,
  including your own) **plus at least one** middle tile.
- **Root must change** (`sameRoot()`): a steal is rejected if the new word is
  just an inflection — S / ES / IES / ED / D / ING / ER / EST / EN, handling
  e-drop (bake→baked), consonant doubling (run→running) and y→i (city→cities).
  So CAT→CATS ✗, TEAM→TEAMS ✗, but TEAM→STREAM ✓ and BREAD→BEARDS ✓.
  Deliberately strict on ER/EST so "just add a suffix" never wins.
- **Multiple legal steals?** `findMove()` picks the **longest source word**
  (the biggest steal).
- **Penalty for a miss**: bad word, gave up, or clock expired → the caller must
  give up one of *their* words and its letters go **back to the middle**
  (`startPenalty` → `giveUpWord`). 20 s to choose or their shortest word is
  auto-picked. Nothing to lose if they have no words yet.
- **Winner** = most **tiles** (sum of word lengths), shown when anyone ends the
  game from the ⋯ menu (the bag counter tells you when it's empty).

## Sync model (the interesting part)
- One Postgres row per room: `public.ana_games(room text pk, state jsonb,
  version int, updated_at)` in the **Misc** Supabase project
  (`atqhfbaurrmivjarowco`, anon key hard-coded — RLS is open, it's a party
  game). Table prefix convention: this game owns `ana_`.
- **Every mutation is an optimistic, version-checked UPDATE**:
  `PATCH /ana_games?room=eq.X&version=eq.N` with `Prefer: return=representation`.
  0 rows back = someone beat us → re-pull and retry (`Store.mutate`). Verified:
  two simultaneous claims at the same version → exactly one wins. This is what
  makes "who called it first" resolve identically on every device.
- **Fresh re-validation on commit.** `submitWord()` checks the word locally for
  instant feedback, then `findMove()` runs **again inside the mutation** against
  whatever the table looks like at commit time — so a word that was legal 200 ms
  ago but just got stolen turns into a penalty, not a phantom score.
- **Live updates** = Supabase realtime (`postgres_changes` on the room) with a
  1.5 s polling safety net (`POLL_MS`) for flaky phone networks. Realtime is
  loaded lazily from esm.sh; if that import fails the game still works on
  polling alone.
- **Clock skew**: deadlines are absolute epoch ms, so every device syncs to the
  server's `Date` response header (`Store.noteTime` → `SKEW` → `nowS()`).
  Never use raw `Date.now()` for anything shared.
- **Refereeing expired timers** is done by *whichever* device notices first
  (`refereeTimers()` every 100 ms); the version check guarantees the penalty is
  applied exactly once.
- Rooms are 4 letters, no I/O (they read as 1/0). Invite link is `#r=CODE`.
- **Solo practice** mode runs the identical engine with `Store.online = false`
  (mutations just apply locally) — handy for testing rules without a room.

## Identity
`ana_pid` (random) + `ana_name` in localStorage. No accounts. A joiner without a
saved name gets the name dialog before being announced, so nobody shows up as
"Player".

## Schema (already provisioned in Misc)
```sql
create table public.ana_games (
  room text primary key,
  state jsonb not null default '{}'::jsonb,
  version int not null default 0,
  updated_at timestamptz not null default now());
alter table public.ana_games enable row level security;
create policy ana_read   on public.ana_games for select using (true);
create policy ana_insert on public.ana_games for insert with check (true);
create policy ana_update on public.ana_games for update using (true) with check (true);
alter table public.ana_games replica identity full;
alter publication supabase_realtime add table public.ana_games;
```
No delete policy on purpose (clients can't nuke a room). Housekeeping:
`delete from public.ana_games where updated_at < now() - interval '7 days';`

## Testing without a browser
The rules engine is pure and can be pulled straight out of the HTML:
slice `index.html` between `const MIN_LEN` and `const Store = {`, `eval` it in
Node with `document`/`localStorage` stubs, then set `DICT.set` from
`words.json`. That's how `sameRoot()` / `findMove()` were checked. NB: assign
the eval result to one object (`const S = eval(...)`) — destructuring clashes
with the function declarations that sloppy-mode eval hoists into scope.

## Hosting
GitHub Pages from `main` root, custom domain via Namecheap CNAME
`anagrams` → `moshed.github.io.` (see the global memory note: `setHosts`
replaces every record, so always re-send them all **and** pass `EmailType=OX`
or Moshe's email breaks). HTTPS enforced once the cert provisions.
