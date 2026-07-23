# Socket events are not phase-checked or idempotent

## Finding

Most event handlers accept messages in any phase. Repeated or out-of-phase messages can increment counters, reset countdown state, pause other players, or start multiple local countdowns. A particularly direct bug sends clients still choosing a role to the “game in progress” screen.

Relevant code:

- `backend/server.js:154-219`
- `frontend/index.js:487-535`, `683-750`

## Offending code

```js
socket.on("color", (color) => {
  player._color = color;
  player.phase = "chat";
  playersInChat++;
  io.emit("player joined chat", { player, playersInChat });
});
```

The same socket can send `"color"` repeatedly and increment the counter repeatedly.

```js
socket.on("ready", () => {
  isGameInProgress = true;
  playersInCountdown = 0;
  // Recounts and broadcasts on every request.
  io.emit("ready");
});

socket.on("pause", () => {
  io.emit("pause");
});
```

Any connected lobby socket can repeatedly reset or pause the global countdown.

The broadcast includes clients outside chat:

```js
socket.on("ready", () => {
  if (phase !== "chat") {
    gameInProgress();
    return;
  }
  startCountdown(resumeFrom);
});
```

A client in code/name/roles therefore receives the wrong UI transition.

## Proposed change

Define allowed transitions and enforce them server-side:

```js
const allowedPlayerTransitions = {
  start: new Set(["name"]),
  name: new Set(["roles"]),
  roles: new Set(["chat"]),
  chat: new Set(["countdown"]),
  countdown: new Set(["game", "chat"]),
  game: new Set(["finished"]),
};

function transitionPlayer(player, nextPhase) {
  if (!allowedPlayerTransitions[player.phase]?.has(nextPhase)) {
    return false;
  }
  player.phase = nextPhase;
  return true;
}
```

Make handlers reject invalid/current duplicate state:

```js
socket.on("color", (color, acknowledge) => {
  if (player.phase !== "roles") {
    return acknowledge?.({ ok: false, reason: "wrong-phase" });
  }
  if (!isAvailableColor(color)) {
    return acknowledge?.({ ok: false, reason: "unavailable-color" });
  }

  player._color = color;
  transitionPlayer(player, "chat");
  broadcastLobby();
  acknowledge?.({ ok: true });
});
```

Only the server transitions the global game phase:

```js
socket.on("countdown:start-request", (acknowledge) => {
  if (gameState.phase !== "lobby" || player.phase !== "chat") {
    return acknowledge?.({ ok: false, reason: "wrong-phase" });
  }
  if (!canStartGame()) {
    return acknowledge?.({ ok: false, reason: "not-enough-players" });
  }

  startServerCountdown();
  acknowledge?.({ ok: true });
});
```

Broadcast a state snapshot only to intended participants, or let all clients render the correct spectator/waiting state from the same snapshot. Avoid command names such as `"ready"` whose meaning depends on local phase.

For every request that can be retried, assign a request ID or make the operation naturally idempotent. For example, “set my selected color to X” is safer than “increment chat count.”

## Verification

For every socket event, test it in every game/player phase and send it twice. Invalid messages should return a controlled rejection and leave all counters/state unchanged. Start a game while another client is in each intro phase and verify that it receives a coherent waiting/spectator state rather than running chat-only countdown code.
