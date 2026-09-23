# AGENTS.md

## Run
- Zero deps — no `package.json`, build, or install. Open `src/index.html` directly.
- Local server (avoids `file://` quirks): `python3 -m http.server --directory src 8000`
- No linter/formatter/test runner/CI/`opencode.json` — verify manually in browser.

## Stack
- Vanilla JS + HTML + CSS only. No bundler, modules, or transpilation.
- Globals via `window.*` — no `import`/`export`. New JS files must expose on `window` and be added as `<script>` in `src/index.html` in dependency order.

## Architecture
- Load order `src/index.html:19-22`: `maze.js` → `game.js` → `render.js` → `main.js`. Do not reorder.
- `src/js/maze.js` — pristine data: `MAZE` (28×31 numeric grid), `MAZE_STR`, `TUNNEL_ROW=14`, `PACMAN_START`, `GHOST_STARTS`. Never mutate `MAZE`; `src/js/game.js:18-19` clones to `game.grid` on `createGame()`.
- `src/js/game.js` — state + rules (`createGame()`, `update()`), `DIRS`/`OPPOSITE` (`src/js/game.js:5-11`). Depends on `maze.js` globals.
- `src/js/render.js` — `draw()` reads `game.grid` (not `MAZE`) so eaten dots disappear; depends on `DIRS`.
- `src/js/main.js` — `requestAnimationFrame` loop, keyboard `Arrow*` → `nextDir`, overlay screens.

## Maze / Grid
- Encoding `MAZE_STR` (`src/js/maze.js:42` `parseTile()`): `'#'`=wall(1), `'.'`=dot(2), `' '`=empty(0), `'-'`=door(3, blocks pacman only via `isWall()` at `src/js/game.js:56`).
- Dims 28×31, `TILE=20` (`src/js/render.js:4`), canvas `560×620` (`src/index.html:11`).
- Tunnel wraparound only on `TUNNEL_ROW` (row 14) — `wrapTunnel()`/`canMove()` at `src/js/game.js:66-81`.

## Game Model
- `game.state`: `start` → `playing` → `won`/`lost`. Win `dotsRemaining===0`, lose `lives<=0` (`src/js/game.js:178-195`).
- Movement is sub-cell floats: `PACMAN_SPEED=0.125`, `GHOST_SPEED=0.1`. Turns only when `aligned()` (integer cell) at `src/js/game.js:49-51`.
- Directions are strings `left`/`right`/`up`/`down` via `DIRS`/`OPPOSITE`.
- Ghost `kind`: `hunter` (Manhattan chase) vs `random` — `decideGhost()` at `src/js/game.js:113`.

## Conventions
- Single CSS file `src/css/style.css`; `#game-wrap` fixed `560×620`.
- Spec-driven learning project (`README.md:11`) — if specs are added, drive implementation from spec.
