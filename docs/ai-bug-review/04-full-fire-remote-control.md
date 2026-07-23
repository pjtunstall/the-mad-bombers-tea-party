# Full-fire and remote-control interaction

## Finding

Most of the reported sequence is explained by the current rules: remote-control mode always takes precedence, and full-fire is attached to and consumed by the next remote bomb. The historical “invisible” rendering is not proven by the current code, but the protocol has no bomb identity or resynchronization mechanism, so a missing/stale client representation cannot be repaired.

Relevant code:

- `backend/server.js:260-266`, `645-675`
- `frontend/index.js:944-959`, `1032-1088`

## Current code

```js
socket.on("bomb", ({ y, x, index }) => {
  if (players[index].remoteControl) {
    plantRemoteControlBomb(y, x, index);
    return;
  }
  plantNormalBomb(y, x, index);
});
```

Remote placement consumes full-fire:

```js
player.remoteControlBombs.push({
  y,
  x,
  fireRange: player.fireRange,
  full: player.fullFire,
});

io.emit("plant remote control bomb", {
  y,
  x,
  index,
  full: player.fullFire,
});

if (player.fullFire) {
  full = true;
  player.fullFire = false;
  player.powerups = player.powerups.filter(
    (powerup) => powerup.name !== "full-fire"
  );
}
```

Problems:

- `full = true` assigns an undeclared variable and is unnecessary.
- The server trusts client coordinates and index.
- The plant event is the client's only source for bomb visuals and remote-control state.
- The state has no `bombId`, revision, acknowledgment, or recovery snapshot.
- The client decides whether Space may request detonation from the local boolean `isRemoteControlBombPlanted`, which can be stale.

## Proposed change

Make placement server-authoritative and return the accepted bomb:

```js
socket.on("bomb:place", (acknowledge) => {
  const player = getActivePlayer(socket.id);
  if (!player) {
    return acknowledge?.({ ok: false, reason: "not-active" });
  }

  const { y, x } = player.position;
  const result = placeBombForPlayer(player, y, x);

  if (!result.ok) {
    return acknowledge?.(result);
  }

  io.emit("bomb:placed", result.bomb);
  acknowledge?.({ ok: true, bomb: result.bomb });
});
```

Create one explicit bomb object:

```js
function placeBombForPlayer(player, y, x) {
  if (!canPlaceBomb(player, y, x)) {
    return { ok: false, reason: "blocked" };
  }

  const bomb = {
    id: crypto.randomUUID(),
    ownerId: player.id,
    y,
    x,
    mode: player.remoteControl ? "remote" : "timed",
    range: player.fullFire ? 13 : player.fireRange,
    full: player.fullFire,
  };

  consumeFullFire(player);
  registerBomb(bomb);
  return { ok: true, bomb };
}
```

Do not gate a valid server request on potentially stale client state:

```js
if (event.code === "Space") {
  event.preventDefault();
  socket.emit("bombs:detonate-remote");
}
```

The server can safely no-op if that player owns no remote bombs. A periodic or reconnect snapshot should include active bombs:

```js
socket.emit("game state", {
  revision,
  players: serializePlayers(),
  bombs: serializeActiveBombs(),
  grid: serializeGrid(),
});
```

## Product decision

The UI should explain whether full-fire is intended to:

1. strengthen the next remote-control bomb, as the current server does; or
2. temporarily override remote-control and place one timed full-fire bomb.

Either rule is valid, but it should be explicit in instructions and represented by one server-side state transition.

## Verification

Collect remote-control, then full-fire. Place and detonate the bomb under normal and throttled networking. All clients should render the same bomb ID, range, mode, and consumed powerup state. Reconnecting during the planted state should reconstruct the bomb from the snapshot.
