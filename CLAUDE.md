# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A brick breaker browser game (UI text in Japanese). The whole game is one self-contained file, `index.html` (HTML + CSS + JS, no dependencies besides Google Fonts). It is deployed with GitHub Pages from `master` at https://ion000.github.io/brock-game/.

There is no build step, package manager, linter or test suite. To run it, open `index.html` in a browser.

## Testing

All game code lives inside one IIFE, so nothing is reachable from outside. To drive the game headlessly (Playwright is available in the cloud container), copy `index.html` to a scratch file and inject a debug hook just before the final `loadLevel(0);` line, then script it:

```bash
sed 's|^loadLevel(0);$|window.__t = { loadLevel, newGame, endGame, hit: hitBrick, bricks: () => bricks, set level(v){ level = v; }, set score(v){ score = v; } };\nloadLevel(0);|' index.html > /path/to/scratch/test.html
```

Never commit the hook. Check for `pageerror` / console errors and take screenshots.

Do not submit test scores to the live ranking. The Firestore rules forbid deletes, so test data cannot be removed from the game.

## Architecture (`index.html`)

- **Layout**: the game uses a fixed logical resolution of 480×720 (`W`, `H`). `#stage` is CSS-scaled to fit the window, and the canvas backing store is `W*H*scale*dpr`, drawn with `setTransform(renderScale)`. All game coordinates are logical, and pointer input is mapped back with `toLogical()`.
- **Canvas vs DOM**: the play field (bricks, paddle, balls, particles, stage banner, combo, power-up timers) is drawn on the canvas. The HUD (`#hud`) and all menus are DOM overlays. The menus are `section.screen`: `#scrTitle`, `#scrPause`, `#scrOver` and `#scrRanking`, toggled by `setScreen(name)` through the `hidden` attribute. A global `[hidden]{display:none!important}` rule is required because some elements set `display` themselves.
- **State machine**: `state` is one of `title`, `serve`, `play`, `paused`, `clear`, `gameover` or `victory`.
  - `update(dt)` branches on `state`. While serving, balls stick to the paddle.
  - Clearing a stage sets `clear`, and after a delay `nextLevel()` runs.
  - When the last life is lost, or after the last stage, `endGame(victory)` runs.
  - `session` is incremented on every new game and on returning to the title, so stale async ranking callbacks are ignored.
- **Levels**: `LEVELS` holds strings of 12 columns. The characters are:
  - `.`: empty
  - `1`–`6`: normal bricks, 1 hit, solid colored style
  - `H`: 2 hits, frosted glass
  - `S`: 3 hits, thick glass
  - `X`: indestructible, metal

  `loadLevel()` turns the strings into brick objects (`hp`, `maxHp`, `seed`, appear animation). Rendering and sound depend on the type:
  - `X` → `drawMetal` and `Sound.metal`, plus sparks.
  - `maxHp > 1` → `drawGlass`, `Sound.glassCrack` / `Sound.glassBreak`, plus `spawnShards`.
  - Otherwise → `drawSolid` and `Sound.brick`.

  A stage is cleared when no non-`X` bricks remain.
- **Ball physics**: `updateBall` uses sub-steps of about 3px to avoid tunneling, circle-vs-rect collision, and reflection along the axis of least penetration. Where the ball hits the paddle sets its bounce angle (±62°). A minimum vertical component prevents horizontal loops.
- **Power-ups**: `POWERUPS` / `POWERUP_WEIGHTS` define the drops. Timed effects live in `effects` with durations in `EFFECT_MAX`, and `applyPowerup` applies them.
- **Sound**: everything is synthesized with WebAudio in the `Sound` object (`tone`, `noise`). There are no audio files. The `AudioContext` is created on the first user gesture (`Sound.init()`).
- **Persistence**: `localStorage` is accessed through the `store` helper, under the keys `bb.best`, `bb.name`, `bb.muted` and `bb.ranking` (the local fallback ranking).

## Ranking / Firebase

- `Ranking` talks to Firestore through its REST API using `fetch`, with no Firebase SDK.
  - It reads with `:runQuery` (ordered by `score` descending, limit `RANK_SIZE` = 10).
  - It writes by POSTing to `/scores`.
  - The config is the `FIREBASE` object (`apiKey`, `projectId: 'block-kuzushi'`). The API key is a public client key.
- If `FIREBASE` is empty, or a request fails, the game falls back to the local `bb.ranking` list and the board is labeled accordingly.
- On game over, `qualifies()` decides whether the score made the top 10. If so, the name form appears. Names pass through `cleanName()` (control characters stripped, max 12 characters). The board is built with `textContent`; keep it that way, because the names are user input.
- `firestore.rules` is the source of truth for server-side validation:
  - Anyone can read.
  - `create` is allowed only with exactly the fields `name`, `score`, `stage` and `createdAt`.
  - `name` must be 1–12 characters, `score` must be from 1 to 5,000,000, and `stage` must be from 1 to 8.
  - Updates and deletes are not allowed.

  If you change the submitted fields, the name length, the number of stages (`LEVELS.length`) or the possible score range, update `firestore.rules` to match. Tell the user they must paste it into the Firebase console (Firestore → Rules → Publish), because committing the file does not deploy it.
