# Countdown races leave stale or misidentified players

## Finding

The countdown runs independently in every browser. The server starts only after receiving a signal from the expected number of browsers, while disconnects mutate that expected number and preserve numeric indexes for later processing. Filtering/reindexing can invalidate those recorded indexes.

The current `isDead()` guards also do not make the game loop safe: after returning `false` for an invalid player, the loop still dereferences that player.

Relevant code:

- `backend/server.js:192-219`, `279-316`, `398-430`, `499-520`, `588-613`, `824-858`
- `frontend/index.js:691-750`

## Offending code

Client-owned countdown:

```js
for await (let count of counter(resumeFrom - 1)) {
  resumeFrom = count;
  imageElement.src = `frontend/assets/images/numbers/${count}.jpg`;
}

socket.emit("start game");
```

Server quorum:

```js
socket.on("start game", () => {
  numberOfStartSignalsReceived++;
  if (numberOfStartSignalsReceived < playersInCountdown) {
    return;
  }
  setTimeout(() => {
    initializeGame();
  }, 500);
});
```

Disconnect index retained across filtering:

```js
disconnectees.push(player.index);

players = players.filter((p) => p && p.color);
for (let i = 0; i < players.length; i++) {
  players[i].index = i;
  if (disconnectees.includes(i)) {
    // This compares new indexes with old indexes.
  }
}
```

Incomplete invalid-player guard:

```js
for (const player of players) {
  if (isDead(player)) {
    kill(player);
  }
  if (beat === true || player.skates > 0) {
    move(player);
  }
}
```

## Proposed change

Make countdown a server deadline, not a client quorum:

```js
let countdownTimer = null;

function startCountdown(requestingPlayer) {
  if (gameState.phase !== "lobby") return;
  if (!canStartGame()) return;

  const endsAt = Date.now() + 10_000;
  transitionTo("countdown", { endsAt });

  countdownTimer = setTimeout(() => {
    const participants = connectedCountdownPlayers();
    if (participants.length < 2) {
      return resetToLobby();
    }
    initializeGame(participants);
  }, 10_000);
}
```

Clients display remaining time from `endsAt`; they never signal that time has elapsed:

```js
function renderCountdown(endsAt) {
  cancelAnimationFrame(countdownFrame);

  function update() {
    const seconds = Math.max(0, Math.ceil((endsAt - Date.now()) / 1000));
    showCountdownNumber(seconds);
    if (seconds > 0) countdownFrame = requestAnimationFrame(update);
  }

  update();
}
```

Use stable IDs rather than array positions:

```js
const disconnectedPlayerIds = new Set();

socket.on("disconnect", () => {
  disconnectedPlayerIds.add(player.id);
});

for (const player of participants) {
  if (disconnectedPlayerIds.has(player.id)) {
    eliminateDisconnectedPlayer(player);
  }
}
```

Harden the game loop even if state becomes malformed:

```js
for (const player of players) {
  if (!isValidActivePlayer(player)) continue;

  if (isDead(player)) {
    kill(player);
    continue;
  }

  if (beat || player.skates > 0) {
    move(player);
  }
}
```

and defensive collision checks:

```js
for (const otherPlayer of players) {
  if (!isValidActivePlayer(otherPlayer)) continue;
  if (otherPlayer.id !== player.id &&
      otherPlayer.position.y === y &&
      otherPlayer.position.x === x) {
    return false;
  }
}
```

Every scheduled callback should capture a `gameId` and no-op if that game is no longer current. Starting/restarting must clear both the countdown timer and any pending initialize timer.

## Verification

Start countdowns with two to four players and disconnect each player position at multiple points, including the final second. Background one mobile tab so its timers are throttled. The server should start or cancel at its deadline without waiting for client signals, and no stale player should enter the game loop.
