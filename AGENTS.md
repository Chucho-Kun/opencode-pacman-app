# AGENTS.md

## Ejecución
- Cero dependencias — sin `package.json`, build ni instalación. Abre `src/index.html` directo en el navegador.
- Servidor local (evita fallos de `file://`): `python3 -m http.server --directory src 8000`
- Sin linter/formatter/test runner/CI/`opencode.json` — verifica manual en el navegador.

## Stack
- Solo Vanilla JS + HTML + CSS. Sin bundler, módulos ni transpilación.
- Globales vía `window.*` — sin `import`/`export`. Todo archivo JS nuevo debe exponer su API en `window` y agregarse como `<script>` en `src/index.html` en orden de dependencias.

## Arquitectura
- Orden de carga `src/index.html:19-22`: `maze.js` → `game.js` → `render.js` → `main.js`. No lo cambies.
- `src/js/maze.js` — datos prístinos: `MAZE` (rejilla numérica 28×31), `MAZE_STR`, `TUNNEL_ROW=14`, `PACMAN_START`, `GHOST_STARTS`. Nunca mutes `MAZE`; `src/js/game.js:18-19` lo clona a `game.grid` en `createGame()`.
- `src/js/game.js` — estado y reglas (`createGame()`, `update()`), `DIRS`/`OPPOSITE` (`src/js/game.js:5-11`). Depende de los globales de `maze.js`.
- `src/js/render.js` — `draw()` lee `game.grid` (no `MAZE`) para que los puntos comidos desaparezcan; depende de `DIRS`.
- `src/js/main.js` — loop con `requestAnimationFrame`, teclado `Arrow*` → `nextDir`, pantallas del overlay.

## Laberinto / Rejilla
- Codificación `MAZE_STR` (`src/js/maze.js:42` `parseTile()`): `'#'`=muro(1), `'.'`=punto(2), `' '`=vacío(0), `'-'`=puerta(3, solo bloquea a Pac-Man vía `isWall()` en `src/js/game.js:56`).
- Dimensiones 28×31, `TILE=20` (`src/js/render.js:4`), canvas `560×620` (`src/index.html:11`).
- Túnel solo en `TUNNEL_ROW` (fila 14) — `wrapTunnel()`/`canMove()` en `src/js/game.js:66-81`.

## Modelo del Juego
- `game.state`: `start` → `playing` → `won`/`lost`. Ganas con `dotsRemaining===0`, pierdes con `lives<=0` (`src/js/game.js:178-195`).
- Movimiento en flotantes sub-celda: `PACMAN_SPEED=0.125`, `GHOST_SPEED=0.1`. Solo gira cuando está `aligned()` (celda entera) en `src/js/game.js:49-51`.
- Direcciones son strings `left`/`right`/`up`/`down` vía `DIRS`/`OPPOSITE`.
- `kind` de fantasma: `hunter` (persecución por distancia Manhattan) vs `random` — `decideGhost()` en `src/js/game.js:113`.

## Convenciones
- Un solo archivo CSS `src/css/style.css`; `#game-wrap` fijo en `560×620`.
- Proyecto de aprendizaje Spec-Driven (`README.md:11`) — si se agregan specs, la implementación debe seguir el spec.
