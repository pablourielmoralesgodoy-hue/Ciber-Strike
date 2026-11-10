# AGENTS.md

## Arquitectura
Sitio estático de un solo archivo: `index.html` contiene todo el juego. `netlify.toml` publica la raíz sin comando de build. No hay backend: el online es P2P con PeerJS.

Scripts dentro de `index.html`, en orden:
1. Cargador de Three.js con mirrors → `window.__threeReadyPromise`.
2. Menús (dispositivo → online/offline → menú principal) y sesión online `window.__csOnlineSession` (PeerJS; el anfitrión retransmite mensajes a los invitados).
3. Motor base dentro de `__threeReadyPromise.then(...)`: escena, `Player`, `ShadowBot`, `GameState`, `startGame`, `resetGame`.
4. Bloque de "expansión Synthwave" ejecutado en `window.onload` dentro de `try { (() => { ... })() }`. Sobrescribe funciones del motor base por reasignación (`spawnInitialBots = function(){...}`, `ShadowBot.prototype.die = ...`). Las definiciones posteriores ganan.
5. Script de parche final (audio): se ejecuta en ámbito global, así que NO puede ver `Player`, `GameState`, `botTargets` ni `spawnInitialBots` (son locales del motor).

## Decisiones no obvias
- Un error en cualquier punto del bloque de expansión aborta todo lo que sigue (16 bots, online). Tras editar, verificar que en consola aparece `CYBER-STRIKE 3D: 16 bots (8 por planta)...` y no `CYBER-STRIKE upgrade error`.
- Funciones definidas dentro de `buildSecondFloor()` no existen fuera; para usarlas fuera se exportan por referencia (ver `__csUpperCrateRef`).
- Spawns de bots: `__lowerBotSpawnPoints` (8, perímetro planta baja, y=0) y `__upperBotSpawnPoints` (8, celdas del borde del laberinto, y=12). `GameState.getRandomBotSpawn` equilibra pisos en las reapariciones; la función local `getRandomBotSpawn` se redirige a ella. En Captura, `__botTeamSpawn` da a cada equipo su mitad del perímetro (azul sur, rojo norte), 4+4 por planta.
- `__csOnlineSession.onData` recibe `(peerId, msg)`. No envolverlo con otra firma: eso descartaba todos los mensajes y la partida online nunca empezaba.
- El botón `btn-start` se clona y se reemplaza en el bloque online; listeners añadidos antes se pierden.
- En online no hay bots; cada jugador recibe un índice de spawn de las 16 zonas.
- HUD online: `#cs-online-hud` es el marcador superior (tiempo de partida desde `net._csMatchStartedAt` + bajas, o AZUL/ROJO y zonas en Captura). Mientras se juega online, `body.cs-online-playing` oculta `#team-score-hud` y `#gun-game-leaderboard` para no tapar la mira.
- Daño online: la víctima calcula el impacto (`__csApplyNetworkShot`) y responde con `hit-confirm` al tirador (marcador + número de daño + aviso de baja). La víctima ve un arco rojo (`#cs-dmg-dir`) hacia el origen del disparo. El anfitrión retransmite `hit-confirm`.
- Menú durante la partida online: cada jugador avisa `state.inMenu`. Si el anfitrión y todos los invitados están en el menú ~1,5 s, el anfitrión termina la partida (`reason:'all-menu'`) sin pantalla final y la sala sigue abierta para elegir otro modo. Si solo algunos salen, pueden cambiar arsenal y pulsar VOLVER A LA PARTIDA.
