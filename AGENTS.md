# AGENTS.md

## Run
- No `package.json`, no build, no deps. Open `src/index.html` directly in a browser.
- For local server (avoids `file://` quirks): `python3 -m http.server --directory src 8000`

## Stack
- Vanilla JS + HTML + CSS only. No bundler, no modules, no transpilation.
- All JS uses global scope via `window.*` — no `import`/`export`.

## Architecture
- Load order matters — `src/index.html:19-22` loads `maze.js` → `game.js` → `render.js` → `main.js`. Keep this order.
- `src/js/maze.js` — defines `MAZE` (28×31 numeric grid), `MAZE_STR`, `TUNNEL_ROW=14`, `PACMAN_START`, `GHOST_STARTS`. Pristine data; never mutate `MAZE` directly.
- `src/js/game.js` — state + rules (`createGame()`, `update()`). Clones `MAZE` into `game.grid` on `createGame()` (`src/js/game.js:18-19`). Depends on globals from `maze.js`.
- `src/js/render.js` — canvas drawing (`draw()`). Reads `game.grid` (not `MAZE`) so eaten dots disappear. Depends on `DIRS` from `game.js`.
- `src/js/main.js` — game loop, keyboard (`Arrow*` → `nextDir`), overlay screens. Entrypoint via `requestAnimationFrame`.

## Maze / Grid Conventions
- String encoding in `MAZE_STR`: `'#'`=wall(1), `'.'`=dot(2), `' '`/`' '`=empty(0), `'-'`=door(3). Parsed by `parseTile()` (`src/js/maze.js:42`).
- Numeric tiles: `0` empty, `1` wall, `2` dot, `3` ghost-house door (blocks pacman only — `isWall()` at `src/js/game.js:56`).
- Grid dims: 28 cols × 31 rows. Canvas `560×620` with `TILE=20` (`src/js/render.js:4`).
- Tunnel wraparound only on `TUNNEL_ROW` (row 14) — `wrapTunnel()` / `canMove()` in `src/js/game.js:66-81`.

## Game Model
- `game.state`: `start` → `playing` → `won`/`lost`. Win when `dotsRemaining===0`, lose when `lives<=0` (`src/js/game.js:178-195`).
- Movement is sub-cell floats (`PACMAN_SPEED=0.125`, `GHOST_SPEED=0.1`). Turns only when `aligned()` (integer cell) — `src/js/game.js:49-51`.
- Directions are strings `left`/`right`/`up`/`down` via `DIRS`/`OPPOSITE` (`src/js/game.js:5-11`).
- Ghost `kind`: `hunter` (Manhattan chase) vs `random` — `decideGhost()` (`src/js/game.js:113`).

## Gotchas
- No linter/formatter/test runner configured. Verify manually in browser.
- Adding a new JS file: append a `<script>` tag in `src/index.html` in dependency order and expose API on `window`.
- CSS is single file `src/css/style.css`; `#game-wrap` is fixed `560×620`.

## Project Context
- Spec Driven Development learning project. Specs (if added) should drive implementation — see `README.md:11`.
