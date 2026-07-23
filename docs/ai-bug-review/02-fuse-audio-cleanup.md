# Bomb fuse audio is not reliably cleaned up

## Finding

Both remote-control and normal bomb fuses can continue looping after the game ends. Three confirmed defects contribute:

1. The server emits `"detonate remote control bomb"` while the client listens for `"detonate remote control bombs"`.
2. Remote fuse cleanup writes to the wrapper object instead of its nested `Audio` element.
3. Normal fuses are not included in game-over cleanup, and late explosion events are ignored once `isGameOver` is true.

Relevant code:

- `backend/server.js:268-276`, `841-854`
- `frontend/index.js:100-109`, `1032-1077`, `1079-1099`, `1175-1190`

## Offending code

Event mismatch:

```js
// Server
io.emit("detonate remote control bomb", index);

// Client
socket.on("detonate remote control bombs", (index) => {
  // cleanup
});
```

Wrong cleanup target:

```js
for (const remoteControlFusesArray of remoteControlFuses) {
  for (const fuse of remoteControlFusesArray) {
    if (fuse) {
      fuse.src = "";
    }
  }
  remoteControlFusesArray.length = 0;
}
```

Each entry is actually `{ fuse: Audio, y, x }`, so `fuse.src` creates or changes a property on the wrapper object.

Normal-fuse leak:

```js
socket.on("add fire", (arr) => {
  if (isGameOver) {
    return;
  }

  triggerBombSound(
    gridData[arr[0].y][arr[0].x].bomb.fuse,
    gridData[arr[0].y][arr[0].x].bomb.full
  );
});
```

If a bomb's server timer fires after `"game over"`, the early return skips the only normal path that stops its fuse.

## Proposed change

Define event names once and use the same value on both sides:

```js
const EVENTS = Object.freeze({
  DETONATE_REMOTE_BOMBS: "detonate remote control bombs",
});
```

Preferably, use a shorter stable protocol name such as `"bombs:remote-detonated"` in both server and client.

Centralize all active audio:

```js
const activeFuseSounds = new Map();

function startFuseSound(bombId) {
  const audio = fuseSound.cloneNode(true);
  audio.loop = true;
  audio.play().catch(() => {});
  activeFuseSounds.set(bombId, audio);
}

function stopFuseSound(bombId) {
  const audio = activeFuseSounds.get(bombId);
  if (!audio) return;

  audio.pause();
  audio.currentTime = 0;
  audio.removeAttribute("src");
  audio.load();
  activeFuseSounds.delete(bombId);
}

function stopAllFuseSounds() {
  for (const bombId of activeFuseSounds.keys()) {
    stopFuseSound(bombId);
  }
}
```

Every bomb event should carry a server-generated `bombId`. Cleanup then does not depend on mutable grid cells or separate normal/remote collections:

```js
socket.on("bomb:planted", ({ bombId }) => {
  startFuseSound(bombId);
});

socket.on("bomb:detonated", ({ bombId, full }) => {
  stopFuseSound(bombId);
  playExplosionSound(full);
});

socket.on("game over", (result) => {
  stopAllFuseSounds();
  showGameOver(result);
});

socket.on("disconnect", stopAllFuseSounds);
window.addEventListener("pagehide", stopAllFuseSounds);
```

If a smaller patch is preferred, at minimum:

```js
for (const entry of remoteControlFuses.flat()) {
  entry.fuse.pause();
  entry.fuse.src = "";
}
```

and stop the bomb audio before the `isGameOver` visual guard:

```js
socket.on("add fire", (arr) => {
  const origin = gridData[arr[0].y]?.[arr[0].x];
  if (origin?.bomb?.fuse) {
    stopAudio(origin.bomb.fuse);
  }

  if (isGameOver) return;
  renderFire(arr);
});
```

## Verification

Test game over with active normal bombs, active remote bombs, chain-detonated remote bombs, and a disconnect during each case. No fuse should remain audible after detonation, game over, disconnect, or navigation.
