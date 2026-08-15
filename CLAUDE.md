# CLAUDE.md

## Project Overview

`mlbserver` is a Node.js server that logs into MLB services and exposes a local web UI plus stream endpoints. No framework, no build step — nearly all logic lives in two files:

- `index.js`: HTTP server, routes, HLS proxy/rewriter, UI generation, multiview, ffmpeg helpers.
- `session.js`: session/auth, cache, MLB API calls, game discovery, guide/channel export, skip markers, Stream Finder, Game Changer.

## How To Run

```bash
npm install
node index.js
```

Docker: see `docker-compose.yml` and `docker-cli`. App on port `9999`, multiview on `10000`.

## CLI Flags

- `--env`: load config from environment variables instead of `credentials.json`
- `--cache` / `--session` / `--logout`: clear persisted state before starting
- `--ffmpeg_path`: override ffmpeg binary (default: `ffmpeg-static`)
- `--http_root`: mount app under a path prefix
- `--data_directory`: store all data/cache files at a custom path (Docker: `/mlbserver/data_directory`)

## Where To Make Changes

- Route behavior, playlist rewriting, UI, multiview → `index.js`
- Auth, API calls, game discovery, caching, export generation → `session.js`
- Deployment defaults → `Dockerfile`, `docker-compose.yml`

## Persisted Files

- `credentials.json`: MLB username/password, favorite teams
- `data/data.json`: tokens, device/session IDs, entitlements, ports
- `data/cookies.json`: cookie jar
- `cache/cache.json` + `cache/*.json`: API response cache
- `data/stream_finder_settings.json`: Stream Finder config

## Refactor Risk Areas

Be careful with:
- String-built playlist rewriting logic
- Auth/session persistence
- Cache expiry behavior
- ffmpeg command assembly
- Game Changer state transitions
- Audio-track fallback behavior
- Blackout filtering

These are tightly coupled and often rely on upstream response quirks.

## Important Notes

- **Upstream bugs**: many issues come from MLB API/auth changes, not local code. Always distinguish local proxy bugs from upstream changes.
- **`request` is deprecated** but deeply embedded — do not casually replace it.
- **No automated tests** — verify manually: homepage loads, a stream resolves via `/stream.m3u8`, `/channels.m3u` and `/guide.xml` generate valid output, multiview starts.

## Syncing with upstream

Upstream remote: `upstream` → https://github.com/tonywagner/mlbserver.git
Local patches are stored in `game-changer-patches.patch`.

Preferred: merge upstream (keeps history, lets git 3-way resolve most of it).

```bash
git fetch upstream
git merge upstream/master
# resolve conflicts, then verify local patches are still present:
node --check index.js && node --check session.js
git commit
# regenerate the patch file (exclude it from itself, or it grows every sync):
git diff upstream/master -- . ':(exclude)game-changer-patches.patch' > game-changer-patches.patch
```

Fallback (throw away local history, replay patch on top of upstream):

```bash
git reset --hard upstream/master
git apply game-changer-patches.patch   # or --3way if it conflicts
```

Verify the patch still reproduces the tree:
```bash
git worktree add --detach /tmp/patchtest upstream/master
git -C /tmp/patchtest apply --check game-changer-patches.patch
git worktree remove --force /tmp/patchtest
```

Check upstream changes relevant to Game Changer before syncing:
```bash
git diff upstream/master...master -- session.js
```

## Local Patches

These changes are not in the upstream repo and are maintained via `game-changer-patches.patch`.

### Game Changer: inning multiplier

**File:** `session.js`, in `getBestGame()`, just before `games.push(`.

Later innings weigh heavier when LI is comparable (6th: 1.1x, 9th: 2.0x, extra innings: 2.5x).

```js
// LOCAL PATCH: inning multiplier so later innings weigh heavier when LI is comparable
const INNING_MULTIPLIERS = [1.0, 1.0, 1.0, 1.0, 1.0, 1.1, 1.3, 1.6, 2.0]
const inning_multiplier = inning_num > 9 ? 2.5 : INNING_MULTIPLIERS[inning_num - 1]
let leverage_index = (LI_TABLE[inning_num_index][inning_half][runners_on_base][outs][run_differential_index] * inning_multiplier) + leverage_adjust
```

### Game Changer: fav team boost

**File:** `session.js`, in `getBestGame()`, after leverage_index is calculated.

Configurable LI bonus for favorite teams (default 0.3, stored in `credentials.json`, adjustable via slider on homepage). The boost is skipped while the game is in a break (`in_break`, set on inning changes / pitching changes), so a paused fav game does not out-rank live action elsewhere.

```js
// LOCAL PATCH: fav team boost - slightly lower the bar for favorite teams
const FAV_TEAM_BOOST = this.credentials.fav_team_boost
if ( !in_break && this.credentials.fav_teams && this.credentials.fav_teams.length > 0 && (this.credentials.fav_teams.includes(away_name_abbrev) || this.credentials.fav_teams.includes(home_name_abbrev)) ) {
  this.debuglog(game_changer_title + teams + ' fav team boost applied')
  leverage_index += FAV_TEAM_BOOST
}
```

The slider lives in `index.js` in the homepage handler: a `settingRow` inside the "Show more stream options" disclosure, with `favTeamBoostInfo` text in `settingsInfoText`. It reads/writes `fav_team_boost` via the query string and `session.setFavTeamBoost()`. When upstream reworks the options UI, this row has to be re-placed into the new markup.

### Game Changer: switch mid-at-bat when current game is boring

**File:** `session.js`, in `getBestGame()`, inside the `for (var i=0; i<best_games.length; i++)` loop.

Switches immediately (without waiting for a new batter) when current game's LI < 0.4.

```js
// LOCAL PATCH: also switch mid-at-bat if current game is really boring (LI < 0.4)
const really_boring = curr_game && (curr_game.leverage_index < 0.4)
if ( !curr_game || really_boring || (curr_game.new_batter && (large_leverage_diff || (curr_game_below_avg && game_better))) ) {
```

### README: Docker image

**File:** `README.md` — docker pull verwijst naar `ghcr.io/schellevis/mlbserver:latest` in plaats van upstream `tonywagner/mlbserver`.

### Crash fix: null cache_data from MLB schedule API (from daslicious/mlbserver)

**Files:** `index.js` (homepage handler), `session.js` (skip marker logic).

Prevents crash when `getDayData()` returns null (e.g. MLB API outage or no games scheduled).

`index.js` — after `getDayData()` call:
```js
if ( !cache_data ) {
  cache_data = { dates: [] }
}
```

`session.js` — tightened null check in skip marker logic:
```js
if ( cache_data && cache_data.dates && cache_data.dates[0] && cache_data.dates[0].games ) {
```

## Mental Model

- `index.js` answers HTTP requests and rewrites media for clients.
- `session.js` knows how MLB works.
- Disk state keeps login/session/cache stable across restarts.
- ffmpeg handles anything beyond plain HLS proxying.
- Most features are variations on "resolve a baseball event to a playable local URL".
