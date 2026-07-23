# Remote bombs remain in stale server/client state

## Finding

When `kill()` force-detonates a player's remote bombs, it neither removes those bombs from `player.remoteControlBombs` nor correctly decrements `player.plantedBombs`. The mismatched detonation event also prevents clients from clearing their corresponding state.

Relevant code:

- `backend/server.js:268-277`, `841-857`
- `frontend/index.js:1079-1099`

## Offending code

```js
function kill(player, isNotDisconnected = true) {
  // ...
  for (const bomb of player.remoteControlBombs) {
    detonate(bomb.y, bomb.x, bomb.full ? 13 : bomb.fireRange);
    if (player.plantedBombs > 0) {
      players.plantedBombs--;
    }
    io.emit("detonate remote control bomb", player.index);
  }
  // remoteControlBombs is never emptied.
}
```

`players.plantedBombs--` modifies a property on the array object, producing invalid count state. Repeated Space events can revisit stale bomb coordinates.

The normal manual path clears the list only after iterating it:

```js
for (const bomb of players[index].remoteControlBombs) {
  detonate(/* ... */);
}
players[index].remoteControlBombs.length = 0;
```

That implementation is duplicated in two places and has already diverged.

## Proposed change

Centralize detonation and remove bombs from authoritative storage before producing effects:

```js
function detonateRemoteBombs(player) {
  const bombs = player.remoteControlBombs.splice(0);
  if (bombs.length === 0) return [];

  const detonatedIds = [];

  for (const bomb of bombs) {
    if (!activeBombs.delete(bomb.id)) continue;

    player.plantedBombs = Math.max(0, player.plantedBombs - 1);
    detonateBomb(bomb);
    detonatedIds.push(bomb.id);
  }

  io.emit("bombs:remote-detonated", {
    ownerId: player.id,
    bombIds: detonatedIds,
  });

  return detonatedIds;
}
```

Use that function for both Space and death:

```js
socket.on("bombs:detonate-remote", () => {
  const player = getActivePlayerForSocket(socket);
  if (player) detonateRemoteBombs(player);
});

function kill(player, reason = "fire") {
  if (player.deathInProgress) return;
  player.deathInProgress = true;
  detonateRemoteBombs(player);
  // Continue death handling.
}
```

On the client, derive whether remote bombs exist from a map keyed by bomb ID rather than a separate boolean:

```js
const activeBombs = new Map();

function hasOwnedRemoteBombs(ownerId) {
  return [...activeBombs.values()].some(
    (bomb) => bomb.ownerId === ownerId && bomb.mode === "remote"
  );
}
```

The detonation event removes every listed ID and stops its audio. Reconnect snapshots replace the map completely.

## Verification

Plant multiple remote bombs where powerups permit it, then detonate manually, via chain reaction, and by player death. After each path, `remoteControlBombs.length` and `plantedBombs` should be zero, each bomb ID should detonate at most once, and all clients should remove the same IDs.
