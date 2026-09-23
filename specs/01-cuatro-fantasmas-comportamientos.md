# SPEC 01 — Cuatro fantasmas con comportamientos individuales

> **Status:** Implemented
> **Depends on:** —
> **Date:** 2026-09-23
> **Objective:** Ampliar de 2 a 4 fantasmas, cada uno con IA distinta, donde uno (hunter) persigue agresivamente a Pac-Man.

## Scope

**In:**

- Expandir `GHOST_STARTS` en `src/js/maze.js:54` de 2 a 4 entradas, todas dentro de la casa (`y=13-14`), con `kind` distintos: `hunter`, `ambusher`, `patrol`, `random`.
- Asignar velocidad por fantasma: `hunter` a `0.125` (igual que `PACMAN_SPEED`), resto a `GHOST_SPEED=0.1` (`src/js/game.js:14`).
- Extender `decideGhost()` en `src/js/game.js:113` a 4 ramas: `hunter` (Manhattan greedy a Pac-Man), `ambusher` (Manhattan greedy a 4 celdas delante de Pac-Man con clamp), `patrol` (regla horaria recto→derecha→izquierda→vuelta), `random` (actual).
- Mapear colores en `src/js/render.js:147` `GHOST_COLORS`: `hunter` rojo `#ff0000`, `ambusher` rosa `#ffb8ff`, `patrol` cyan `#00ffff`, `random` naranja `#ffb852` (orden indexado a `game.ghosts`).
- Salida inmediata de casa: los 4 atraviesan `'-'` (puerta `3`) sin delay, usando `isWall(..., 'ghost')` actual (`src/js/game.js:56`).

**Out of scope (for future specs):**

- Modo asustado / power-pellets / fantasmas comestibles.
- Ciclos scatter/chase temporizados (clásico).
- Salida escalonada de casa por tiempo o por dots comidos.
- Pathfinding óptimo BFS/A* (se mantiene Manhattan greedy).
- Cambios a `MAZE`, `MAZE_STR`, `TUNNEL_ROW`, `PACMAN_START`, `TILE` o `src/css/style.css`.
- Nuevo archivo JS (se reutilizan `maze.js`/`game.js`/`render.js`).

## Data model

```js
// src/js/maze.js
const GHOST_STARTS = [
  { x: 13, y: 14, kind: 'hunter' },   // agresivo — rojo
  { x: 14, y: 14, kind: 'ambusher' }, // emboscada — rosa
  { x: 13, y: 13, kind: 'patrol' },   // patrulla — cyan
  { x: 14, y: 13, kind: 'random' },   // aleatorio — naranja
];

// src/js/game.js — createGame()
ghosts: GHOST_STARTS.map(g => ({
  x: g.x, y: g.y,
  dir: 'up',
  speed: g.kind === 'hunter' ? 0.125 : 0.1,
  kind: g.kind, // 'hunter' | 'ambusher' | 'patrol' | 'random'
}));

// decideGhost() — lógica por kind
// hunter:  target = (round(pacman.x), round(pacman.y))
// ambusher: target = pacman + DIRS[pacman.dir]*4, clamp a [0,W-1]x[0,H-1] y
//           si cae en muro (grid[y][x]===1), buscar celda transitable más cercana
//           por Manhattan; luego elegir dir que minimiza dist a target
// patrol:   en intersección, probar en orden horario: recto > derecha > izquierda > vuelta
//           (derecha/izquierda relativas a g.dir via DIRS/OPPOSITE)
// random:   elección uniforme entre choices (actual)

// src/js/render.js
const GHOST_COLORS = ['#ff0000', '#ffb8ff', '#00ffff', '#ffb852'];
// índice 0..3 corresponde 1:1 a GHOST_STARTS / game.ghosts
```

Coordenadas: origen arriba-izquierda, `x in [0,27]`, `y in [0,30]`. Velocidades en celdas/frame. `aligned()` (`src/js/game.js:49`) sigue gateando giros.

## Implementation plan

1. Editar `src/js/maze.js:54-57` — reemplazar `GHOST_STARTS` de 2 a 4 entradas con posiciones `(13,14) hunter`, `(14,14) ambusher`, `(13,13) patrol`, `(14,13) random`. Verificación: abrir `src/index.html` en `python3 -m http.server --directory src 8000`, `createGame().ghosts.length === 4` en consola.
2. Editar `src/js/game.js:39-45` — en `createGame()`, asignar `speed` por `kind` (`hunter` → `0.125`, resto `GHOST_SPEED`). Verificación: `createGame().ghosts[0].speed === 0.125` y resto `0.1`.
3. Editar `src/js/game.js:113-142` — extender `decideGhost()` con ramas `ambusher` (cálculo target + clamp + Manhattan greedy) y `patrol` (orden horario usando `DIRS`/`OPPOSITE` y `canMove(..., 'ghost')`). Mantener `hunter` y `random` intactos. Verificación: cada fantasma cambia de dirección en intersecciones según su regla; `hunter` acorta distancia, `ambusher` corta por delante, `patrol` gira en sentido horario, `random` no revierte salvo callejón.
4. Verificar `src/js/render.js:147` — confirmar `GHOST_COLORS` en orden rojo/rosa/cyan/naranja; ajustar si el orden actual difiere. Verificación: 4 fantasmas visibles con colores distintos al iniciar partida.
5. Prueba de regresión manual: mover a Pac-Man contra muros/puerta, túnel fila 14 (`wrapTunnel`), comer dots, colisión con cada fantasma (`collides` <0.5), estados `won`/`lost`. Verificación: sin errores en consola, `dotsRemaining` descuenta, `lives` y `resetPositions()` funcionan.

## Acceptance criteria

- [ ] `createGame().ghosts.length === 4` y cada uno tiene `kind` distinto en `{hunter, ambusher, patrol, random}`.
- [ ] `GHOST_STARTS` contiene 4 posiciones todas con `y` en `13` o `14` (dentro de la casa) y `x` en `13` o `14`.
- [ ] El fantasma `hunter` tiene `speed === 0.125` y los otros tres `speed === 0.1`.
- [ ] En una intersección abierta, el `hunter` elige la dirección que minimiza `|nx-px|+|ny-py|` hacia Pac-Man (Manhattan greedy), sin considerar `OPPOSITE[g.dir]` salvo callejón.
- [ ] El `ambusher` apunta a `Pac-Man + 4*DIRS[pacman.dir]` clampado a la grilla; si el target es muro, persigue la celda transitable más cercana a ese target; verificable moviendo a Pac-Man en línea recta y observando que el rosa se adelanta.
- [ ] El `patrol` en intersección sigue prioridad `recto > derecha > izquierda > vuelta` (horario relativo) entre opciones transitables; verificable en cruce de 4 vías.
- [ ] El `random` mantiene elección uniforme entre `choices` (no `hunter`).
- [ ] Los 4 fantasmas atraviesan la puerta `'-'` (`grid===3`) y salen de la casa inmediatamente al iniciar `playing` (sin delay).
- [ ] `draw()` renderiza 4 fantasmas con colores `#ff0000`, `#ffb8ff`, `#00ffff`, `#ffb852` en orden correspondiente a `GHOST_STARTS`.
- [ ] No hay regresión: túnel `TUNNEL_ROW=14`, `isWall` para Pac-Man vs ghost, `aligned()`, comer dots `score +=10`, `won`/`lost`.
- [ ] El juego carga sin errores en consola vía `python3 -m http.server --directory src 8000` y `src/index.html`.

## Decisions

- **Sí:** 4 roles `hunter/ambusher/patrol/random` simples y distinguibles. Razón: cumple pedido "comportamiento individual y diferente" sin complejidad de Inky/Clyde clásicos; más fácil de verificar manualmente.
- **No:** Clonar Blinky/Pinky/Inky/Clyde fiel. Razón: requiere targeting con offset de otro fantasma (Inky) y modo scatter, fuera de alcance.
- **Sí:** `hunter` a `0.125` igual que Pac-Man, resto `0.1`. Razón: presión agresiva sin ser inalcanzable; mantiene constantes existentes (`PACMAN_SPEED`, `GHOST_SPEED`).
- **No:** `hunter` más rápido que Pac-Man (`0.13`). Razón: descartado por usuario — igualar velocidades es más balanceado para MVP.
- **Sí:** Manhattan greedy existente para `hunter` y `ambusher`. Razón: ya implementado, O(1), suficiente para laberinto 28×31; BFS/A* sería overengineering en este spec.
- **No:** BFS/A* para persecución perfecta. Razón: descartado explícitamente.
- **Sí:** `ambusher` con clamp a grilla + corrección a celda transitable más cercana si target es muro. Razón: evita target inválido que haría al fantasma vagar; mantiene persecución coherente.
- **Sí:** `patrol` por regla horaria `recto>derecha>izquierda>vuelta`. Razón: simula patrulla sin waypoints hardcodeados; determinista y testeable.
- **No:** Waypoints de esquinas. Razón: frágil si cambia `MAZE`, requiere lista de coordenadas.
- **Sí:** Los 4 arrancan dentro de la casa `(13-14,13-14)` y salen inmediato. Razón: pedido del usuario y simplicidad; no requiere timers ni contadores de dots.
- **No:** Salida escalonada por tiempo/dots. Razón: va a otro spec si se quiere fidelidad arcade.
- **Sí:** Reutilizar `DIRS`/`OPPOSITE`/`canMove`/`aligned`/`wrapTunnel`. Razón: respeta arquitectura (`AGENTS.md`) y evita duplicar lógica de movimiento sub-celda.
- **Sí:** Sin nuevo archivo JS, sin cambiar orden de `<script>` en `src/index.html:19-22`. Razón: `AGENTS.md` exige globales vía `window.*` y orden fijo; nuevo archivo añadiría carga sin necesidad.

## Risks

| Risk | Mitigation |
|------|------------|
| `ambusher` target fuera de grilla o en muro causa elección errática | Clampar a `[0,W-1]x[0,H-1]` y corregir a celda transitable más cercana antes de calcular distancias; fallback a `random` si no hay `choices` |
| `patrol` se queda oscilando en pasillo recto | Regla horaria solo aplica en intersecciones (`aligned` true); en pasillo recto `choices` tiene 1 opción, avanza recto |
| 4 fantasmas en la misma celda de casa causan colisión inmediata con Pac-Man si `PACMAN_START` cercano | Casa en `y=13-14` lejos de `PACMAN_START (13,23)` (9 filas); `resetPositions` separa posiciones iniciales |
| Velocidad `0.125` no alinea cada 8 frames si se mezcla con `0.1` | `aligned()` usa tolerancia `1e-3`; ambos son múltiplos de `1/40`, alinean eventualmente; no mezclar en el mismo ghost |
| `GHOST_COLORS` desalineado con `GHOST_STARTS` | Documentar mapeo 1:1 por índice; test visual: cada `kind` tiene color esperado |

## What is **not** in this spec

- Power-pellets, modo asustado, fantasmas comestibles ni parpadeo.
- Scatter/chase temporizado ni nivel de dificultad progresivo.
- Salida de casa con delay, llave de casa ni reaparición tras ser comido.
- BFS/A* ni raycasting.
- Cambios de layout del laberinto, HUD nuevo, sonidos o persistencia.

Cada uno de esos, si llega, va en su propio spec.
