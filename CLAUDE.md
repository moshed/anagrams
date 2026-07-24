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
- **Bag** = one of three tile sets (`TILE_SETS`), shuffled at game start and
  recorded in `state.settings.bagMix`; picked on the home screen or in
  ⋯ → Table settings (stored per browser in `ana_bag`, applied to the NEXT game):
  - `scrabble` — 98 tiles, 43% vowels — **the default**
  - `bananagrams` — the real 144-tile Bananagrams mix, 42% vowels
  - `lean` — 87 tiles, 36% vowels — Scrabble's consonants with the vowels
    thinned; the only mix that actually plays drier (the other two are
    within a point of each other, which surprises people)
  Anyone can hit **Flip** to turn one tile face-up into the middle (500 ms
  global cooldown, blocked while a claim/penalty is live).
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
  give up one of *their* words and its letters are shuffled back **into the bag,
  unflipped** (`returnToBag()`), never onto the table — otherwise the next player
  just picks the same word straight back up. 20 s to choose or their shortest
  word is auto-picked. Nothing to lose if they have no words yet.
- **Flip approval** (⋯ → Flip settings, stored in `state.settings`): a flip can
  require a raw number of players or a percentage of them to ask
  (`flipThreshold()`); asks are `state.flipVotes`, shown on the Flip button and
  cleared whenever the table changes. Default is 1 = anyone flips instantly.
  ⚠️ `settings` holds both `bagMix` and the flip rule — always MERGE when writing
  it (`Object.assign`), never replace the object, or you wipe the other setting.
- **Live source preview** (`previewMove()`): as you type, the claim box shows a
  glowing mini-tile per letter taken from the middle and an orange chip for the
  word you'd take, named with its owner. Makes "all of it or nothing" obvious.
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
  Never use raw `Date.now()` for anything shared. The header only has 1-second
  resolution, so a *correct* device would pick up ~1s of phantom skew and referee
  a countdown early/late — `noteTime` therefore ignores offsets under 3s.
- **A room row that vanishes is rebuilt** (`Store.missing` → `Store.reseat()`),
  so housekeeping or a stray delete can't brick a game in progress.
  ⚠️ Never run a bare `delete from public.ana_games` — scope it by `updated_at`.
- **Refereeing expired timers** is done by *whichever* device notices first
  (`refereeTimers()` every 100 ms); the version check guarantees the penalty is
  applied exactly once.
- Rooms are 4 letters, no I/O (they read as 1/0). Invite link is `#r=CODE`.
- **Solo practice** mode runs the identical engine with `Store.online = false`
  (mutations just apply locally) — handy for testing rules without a room.

## Identity & resume
`ana_pid` (random) + `ana_name` in localStorage — the browser IS the account. No
accounts, no passwords. A joiner without a saved name gets the name dialog before
being announced, so nobody shows up as "Player".

The home screen lists **your games** (`renderResume()` → `myGames()`): rooms this
browser has played (`ana_rooms` in localStorage) merged with a server scan of the
last 7 days for any room whose `state.players` still contains this player id. ⋯ →
**My player ID** copies the id, or adopts one pasted from another device — that's
how a game follows you between phone and laptop.

## Layout rules that matter
Opponents render 3-across **above** the table (`#playersTop`, wrapping past
three), you render **below** it next to the buttons (`#playersBottom`). Word
lists never scroll or clip — cards grow, the middle gives up height and
`sizeTiles()` shrinks the tiles to fit. Past 8 words a card goes `.dense`.
The claim sheet is hidden with `visibility/opacity`, **never `display:none`** —
iOS won't focus an input that wasn't rendered at tap time, and that's the
difference between the keyboard appearing instantly and not at all.

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
