# “Game in progress” lifecycle can strand clients

## Finding

`isGameInProgress` is set when countdown begins but is not cleared when the game ends. It is cleared only by `restart()`, which depends on all original in-game sockets eventually disconnecting. Waiting clients receive one command-like `"play again"` event rather than an authoritative state snapshot.

Relevant code:

- `backend/server.js:18-27`, `125-129`, `192-204`, `279-316`, `382-395`, `787-804`
- `frontend/index.js:111-118`, `190-209`, `1175-1258`

## Offending code

```js
io.on("connection", (socket) => {
  if (isGameInProgress) {
    socket.emit("game in progress");
    return;
  }

  // Normal handlers are never installed for this socket.
});
```

At game over:

```js
if (numPlayers === 1) {
  isAfterGame = true;
  clearInterval(gameLoopId);
  io.emit("game over", { survivorIndex, type: isNotDisconnected });
}
```

`isGameInProgress` remains true. The eventual reset is coupled to disconnections:

```js
if (isAfterGame) {
  playersInGame--;
  if (playersInGame === 0) {
    restart();
  }
  return;
}
```

The waiting client works around this by repeatedly reloading:

```js
setInterval(() => {
  window.location.reload();
}, 40000);
```

Reloading cannot help while the server flag remains true, and a one-shot `"play again"` broadcast cannot be recovered if the client is disconnected when it is sent.

## Proposed change

Use one explicit server phase:

```js
const GamePhase = Object.freeze({
  LOBBY: "lobby",
  COUNTDOWN: "countdown",
  PLAYING: "playing",
  FINISHED: "finished",
});

let gameState = {
  revision: 0,
  phase: GamePhase.LOBBY,
  players: [],
  countdownEndsAt: null,
  result: null,
};
```

Every connection should get the current snapshot and should still receive a disconnect handler:

```js
io.on("connection", (socket) => {
  socket.emit("game state", publicGameState());

  socket.on("disconnect", () => {
    removeSocketFromCurrentState(socket.id);
  });

  if (gameState.phase !== GamePhase.LOBBY) {
    return;
  }

  registerLobbyHandlers(socket);
});
```

The client should render from state rather than interpret a one-time command:

```js
socket.on("game state", (state) => {
  if (state.revision < latestRevision) return;
  latestRevision = state.revision;

  switch (state.phase) {
    case "lobby":
      showLobby(state);
      break;
    case "countdown":
      showCountdown(state.countdownEndsAt);
      break;
    case "playing":
      showGameOrWaitingScreen(state);
      break;
    case "finished":
      showFinishedState(state.result);
      break;
  }
});
```

At game end, transition immediately and schedule a server-owned return to lobby:

```js
function finishGame(result) {
  clearGameTimers();
  gameState = {
    ...gameState,
    revision: gameState.revision + 1,
    phase: GamePhase.FINISHED,
    result,
  };
  io.emit("game state", publicGameState());

  setTimeout(resetToLobby, 10_000);
}
```

This removes the requirement that every browser complete its outro and disconnect before another game can exist. If reconnecting players should rejoin an active game, add a stable session token; otherwise the snapshot can explicitly mark them as spectators.

## Verification

Connect a waiting browser during play, finish the match while some original players remain on the outro, and verify the waiting browser receives `finished` and then `lobby` without reload. Repeat while briefly disabling its network across both transitions; its reconnect snapshot should still converge to the current phase.
