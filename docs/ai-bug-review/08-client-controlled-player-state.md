# The server trusts client-controlled identity and coordinates

## Finding

Movement and bomb events include a player index chosen by the client. Bomb placement also includes client-chosen coordinates. The server uses those values directly instead of deriving identity and position from the socket's authenticated player.

This is both a robustness and security problem. An honest but stale client can affect the wrong player after reindexing; a crafted client can control another player, detonate their bombs, or force invalid grid accesses.

Relevant code:

- `backend/server.js:221-276`, `615-675`
- `frontend/index.js:935-975`

## Offending code

```js
socket.on("move", ({ index, key }) => {
  players[index].direction.key = key;
  // ...
});

socket.on("bomb", ({ y, x, index }) => {
  if (players[index].remoteControl) {
    plantRemoteControlBomb(y, x, index);
    return;
  }
  plantNormalBomb(y, x, index);
});

socket.on("detonate remote control bombs", (index) => {
  for (const bomb of players[index].remoteControlBombs) {
    // ...
  }
});
```

The `stop` handler already uses the closure's `player`, showing that the index is unnecessary:

```js
socket.on("stop", (index) => {
  player.direction.y = 0;
  player.direction.x = 0;
});
```

## Proposed change

Send only the requested action:

```js
socket.emit("player:move", { key: event.key });
socket.emit("player:stop");
socket.emit("bomb:place");
socket.emit("bombs:detonate-remote");
```

Resolve and validate the actor on the server:

```js
function getActivePlayerForSocket(socket) {
  const player = playersBySocketId.get(socket.id);
  if (!player) return null;
  if (gameState.phase !== "playing") return null;
  if (player.lives <= 0 || player.deathInProgress) return null;
  return player;
}

socket.on("player:move", ({ key } = {}) => {
  const player = getActivePlayerForSocket(socket);
  if (!player || !MOVEMENT_KEYS.has(key)) return;
  setDirection(player, key);
});

socket.on("player:stop", () => {
  const player = getActivePlayerForSocket(socket);
  if (!player) return;
  player.direction = { y: 0, x: 0, key: "" };
});
```

Use the authoritative position for bombs:

```js
socket.on("bomb:place", () => {
  const player = getActivePlayerForSocket(socket);
  if (!player) return;

  const { y, x } = player.position;
  if (!canPlaceBomb(player, y, x)) return;

  placeBomb(player, y, x);
});

socket.on("bombs:detonate-remote", () => {
  const player = getActivePlayerForSocket(socket);
  if (!player) return;
  detonateRemoteBombsOwnedBy(player.id);
});
```

Indexes may still be included in server-to-client render events as presentation data, but they should never authorize a client action. Stable player IDs are preferable because array positions change.

Validate payload shape and rate-limit input so malformed or excessively frequent events are cheap no-ops.

## Verification

Using a Socket.IO test client, send missing, negative, huge, string, and another player's indexes and arbitrary bomb coordinates. None should affect identity or position. Reindex players after a disconnect and verify that the original socket still controls only its mapped player.
