# Lobby disconnect and ready-button state can diverge

## Finding

The client treats any transport interruption as a new page/session, while lobby membership and ready-button visibility are reconstructed partly from one-time events and mutable counters. This makes a transient hotspot interruption capable of producing different UI on different clients.

Relevant code:

- `backend/server.js:125-176`, `279-316`
- `frontend/index.js:111-118`, `471-586`, `588-637`

## Offending design

```js
const socket = io(":3000", {
  reconnection: false,
});

socket.on("disconnect", () => {
  location.reload(true);
});
```

Server membership is represented by an array plus separate counters:

```js
let players;
let playersInChat;
let playersInCountdown;
let playersInGame;
```

Client ready-button visibility depends on its local event history:

```js
socket.on("player left", ({ player, playersInChat }) => {
  playersInLobby = playersInChat;

  if (playersInChat < 2) {
    readyButton.classList.add("hide");
    readyButton.removeEventListener("click", readyHandler);
  }
});
```

There is no stable player/session identity across a reconnect, no lobby revision, and no acknowledgment that a replacement socket supersedes an old one.

## Proposed change

Allow Socket.IO reconnection and identify a browser session independently of `socket.id`:

```js
const sessionId =
  localStorage.getItem("sessionId") ?? crypto.randomUUID();
localStorage.setItem("sessionId", sessionId);

const socket = io(":3000", {
  auth: { sessionId },
  reconnection: true,
});
```

On the server, maintain one canonical lobby member record per session:

```js
const lobbyMembers = new Map();
let lobbyRevision = 0;

io.on("connection", (socket) => {
  const sessionId = validateSessionId(socket.handshake.auth.sessionId);
  attachSocketToSession(sessionId, socket.id);
  emitLobbySnapshot(socket);

  socket.on("disconnect", () => {
    markSessionDisconnected(sessionId);
    scheduleRemovalUnlessReconnected(sessionId);
  });
});
```

Broadcast complete, revisioned snapshots after membership changes:

```js
function broadcastLobby() {
  lobbyRevision++;
  io.emit("lobby state", {
    revision: lobbyRevision,
    players: [...lobbyMembers.values()].map(toPublicPlayer),
    canStart: connectedChatPlayers().length >= 2,
  });
}
```

Render the ready button from that state:

```js
socket.on("lobby state", (state) => {
  if (state.revision <= latestLobbyRevision) return;
  latestLobbyRevision = state.revision;
  players = state.players;
  readyButton.hidden = !state.canStart;
  renderLobby(state);
});
```

Register the button listener once during startup:

```js
readyButton.addEventListener("click", () => {
  socket.emit(counting ? "countdown:pause-request" : "countdown:start-request");
});
```

The server, not each client, should decide whether the request is valid.

## Verification

In a two-player lobby, interrupt one client's network for shorter and longer than the reconnection grace period. Both screens should converge to the same revision, player list, and ready-button state. Reconnecting must not create a duplicate player or increment `playersInChat` twice.
