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

### UI facelift: dark theme, light mode, controls panel

**File:** `index.js`, in the homepage handler. The CSS is built as a string concatenation in three `body +=` blocks near the top of the page builder — search for `<style type="text/css">`.

**What:** replaces the plain black/gray CSS with a structured dark theme, automatic light mode via `prefers-color-scheme`, improved buttons/table/tooltips, and a controls panel with flex-based label/value rows.

**Key things to re-apply:**

1. **`@media (prefers-color-scheme: light){...}` block** — add before the main CSS. Contains all light-mode overrides.

2. **Main CSS block** — replace the content of `<style type="text/css">`. Key additions vs upstream:
   - `body`: `#0f1117` bg, `#e2e8f0` text, system font stack, `padding: 0 12px 24px`
   - `button`: dark surface, border, border-radius, hover state; `button.default`: blue (`#2563eb`)
   - `th`: uppercase, muted color, dark surface bg
   - `tbody tr:hover td`: subtle highlight
   - `select`, `input[type=date]`, `textarea`, `input[type=number]`: dark surface styling
   - `.controls`: card wrapper with border-radius and subtle bg
   - `.controls p`: `display:flex; align-items:center; flex-wrap:wrap; gap:5px; border-bottom`
   - `.controls p>.tooltip:first-child`: fixed `width:115px`, muted color — creates label column
   - `.hint`: muted small text, no border-bottom
   - `.blackout`: adds `opacity:.5`

3. **Highlights/modal CSS block** — dark modal (`#1a1d2e` bg, border-radius, `#60a5fa` links).

4. **Tooltip CSS block** — dark tooltip card with border, shadow, better positioning (`top:100%; left:0; margin-top:4px`).

5. **Controls wrapper HTML** — wrap the controls section in `<div class="controls">`:
   - Add `<div class="controls">` after `</h1>` (and change hint `<p>` to `<p class="hint">`)
   - Add `</div>` just before `<table>` (the games table)

6. **Level/Org split** — Level buttons and Org dropdown are on the same `<p>` upstream. Split them:
   - Find: `body += ' or <span class="tooltip">Org`
   - Replace: `body += '</p>\n<p><span class="tooltip">Org`

## Mental Model

- `index.js` answers HTTP requests and rewrites media for clients.
- `session.js` knows how MLB works.
- Disk state keeps login/session/cache stable across restarts.
- ffmpeg handles anything beyond plain HLS proxying.
- Most features are variations on "resolve a baseball event to a playable local URL".
