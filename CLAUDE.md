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

## Local Patches

These changes are not in the upstream repo and must be re-applied after updating mlbserver.

### Game Changer: switch mid-at-bat when current game is boring

**File:** `session.js`, in `getBestGame()`, inside the `for (var i=0; i<best_games.length; i++)` loop.

**What:** adds `really_boring` condition so Game Changer switches immediately (without waiting for a new batter) when the current game's leverage index is very low.

**Find this line:**
```js
if ( !curr_game || (curr_game.new_batter && (large_leverage_diff || (curr_game_below_avg && game_better))) ) {
```

**Replace with:**
```js
// LOCAL PATCH: also switch mid-at-bat if current game is really boring (LI < 0.4)
const really_boring = curr_game && (curr_game.leverage_index < 0.4)
if ( !curr_game || really_boring || (curr_game.new_batter && (large_leverage_diff || (curr_game_below_avg && game_better))) ) {
```

## Mental Model

- `index.js` answers HTTP requests and rewrites media for clients.
- `session.js` knows how MLB works.
- Disk state keeps login/session/cache stable across restarts.
- ffmpeg handles anything beyond plain HLS proxying.
- Most features are variations on "resolve a baseball event to a playable local URL".
