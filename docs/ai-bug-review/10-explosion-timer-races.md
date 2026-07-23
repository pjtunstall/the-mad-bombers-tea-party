# Explosion timers can overwrite newer cell or game state

## Finding

Every fire application schedules an unconditional transition to `"walkable"` one second later. The callback does not verify that the same fire instance still owns the cell. If explosions overlap, an older timer can clear newer fire early. Timers are also not tied to a game generation, so callbacks can survive phase changes or restarts.

Relevant code:

- `backend/server.js:382-430`, `615-755`, `787-804`

## Offending code

```js
function addFire(y, x, style, origin) {
  // ...
  grid[y][x].type = "fire";
  setTimeout(() => {
    grid[y][x].type = "walkable";
    io.emit("remove fire", { y, x, style });
  }, 1000);
}
```

Example:

1. Explosion A sets a cell to fire and schedules removal at T+1000ms.
2. Explosion B reaches that cell at T+800ms and schedules removal at T+1800ms.
3. A's callback runs at T+1000ms and marks the cell walkable while B should still be active.

Bomb, chain-reaction, death, and fire timers are stored inconsistently. `restart()` clears only `playAgainTimeoutId`, and game over clears only the game-loop interval.

## Proposed change

Give transient cell effects identity and expiry:

```js
let nextEffectId = 0;

function addFire(y, x, style) {
  const cell = grid[y][x];
  const effect = {
    id: ++nextEffectId,
    gameId: currentGameId,
    expiresAt: Date.now() + 1000,
    style,
  };

  cell.fireEffects ??= new Map();
  cell.fireEffects.set(effect.id, effect);
  cell.type = "fire";

  io.emit("fire:added", { y, x, effect });

  registerGameTimeout(() => {
    removeFireEffect(y, x, effect);
  }, 1000);
}

function removeFireEffect(y, x, effect) {
  if (effect.gameId !== currentGameId) return;

  const cell = grid[y]?.[x];
  if (!cell?.fireEffects?.delete(effect.id)) return;

  if (cell.fireEffects.size === 0) {
    cell.type = "walkable";
  }

  io.emit("fire:removed", { y, x, effectId: effect.id });
}
```

Manage every timeout through one game-scoped registry:

```js
const gameTimeouts = new Set();

function registerGameTimeout(callback, delay) {
  const id = setTimeout(() => {
    gameTimeouts.delete(id);
    callback();
  }, delay);
  gameTimeouts.add(id);
  return id;
}

function clearGameTimers() {
  for (const id of gameTimeouts) clearTimeout(id);
  gameTimeouts.clear();
  clearInterval(gameLoopId);
}
```

Call `clearGameTimers()` before replacing `grid`, increment `currentGameId`, and make all remaining asynchronous callbacks verify their captured game ID.

For simpler rules, fire can use one `fireUntil` timestamp per cell; an earlier callback should only clear when `Date.now() >= cell.fireUntil`.

## Verification

Create overlapping explosions 100–900ms apart and assert that the cell remains fire until the latest expiry. End/restart a game with bombs, fire, and death animations pending; no old callback should mutate the new grid or emit an event carrying the old `gameId`.
