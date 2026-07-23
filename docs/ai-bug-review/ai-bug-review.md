# AI bug review

This was a static review of the current code, not a live multi-device play test. “Confirmed” below means that the defect is visible in the control flow; it does not mean that every historical symptom was reproduced.

## Summary

Several of the intermittent bugs have fairly direct explanations:

- The large-screen scrolling is caused by allowing the arrow keys' browser default action, made much more visible because the whole `body` can be scaled above 100%.
- The remote-control fuse that survives game over has a definite cleanup bug.
- The remote-control detonation event has different names on the server and client, leaving client state and audio bookkeeping stale.
- The “game in progress” page relies on a one-shot event and has no state resynchronization. This is inherently unreliable on the mobile-hotspot conditions where it was observed.
- The old out-of-range grid crash was not caused by the hotspot creating an invalid array. The bounds check is incorrect and occurs after an unsafe grid lookup; latency or stale input merely made the latent problem easier to reach.

The most important general issue is that the server accepts player indexes and bomb coordinates from the client instead of deriving them from the socket's `player`. That makes stale messages capable of affecting the wrong player and turns otherwise recoverable protocol desynchronization into server exceptions.

## Detailed reports and proposed changes

Each report includes the current offending code, a more detailed failure sequence, proposed replacement code, and verification ideas:

1. [Arrow-key scrolling and large-screen scaling](01-arrow-key-scroll.md)
2. [Bomb fuse audio cleanup](02-fuse-audio-cleanup.md)
3. [“Game in progress” lifecycle](03-game-in-progress-lifecycle.md)
4. [Full-fire and remote-control interaction](04-full-fire-remote-control.md)
5. [Lobby disconnect and ready-button state](05-lobby-disconnect-state.md)
6. [Countdown races and stale players](06-countdown-and-stale-players.md)
7. [Grid bounds validation](07-grid-bounds.md)
8. [Client-controlled identity and coordinates](08-client-controlled-player-state.md)
9. [Remote-bomb server/client state](09-remote-bomb-state.md)
10. [Explosion timer races](10-explosion-timer-races.md)
11. [Protocol phase validation and idempotency](11-protocol-phase-validation.md)
12. [Destruction and powerup state synchronization](12-destruction-state-sync.md)
13. [Player powerup-drop fallthrough](13-player-drop-fallthrough.md)

## Findings corresponding to `bugs.md`

### 1. Arrow keys scroll a huge game area — confirmed

`frontend/index.js:935-943` handles arrow-key `keydown` events but never calls `e.preventDefault()`. Arrow keys therefore both control the player and perform their normal browser scrolling.

The unusual size on a large display is explained by `frontend/index.js:758-760`:

```js
const factor = Math.min(screen.height / 768, screen.width / 1366);
document.body.style.transform = `scale(${0.5 * factor})`;
```

On a sufficiently large screen, `0.5 * factor` is greater than `1`. Transforming the whole body above 100% creates visual overflow, while `#grid-wrapper` is already at least the fixed grid height plus 240px and is `100vw` wide (`frontend/styles.css:685-693`). The browser then has a large scrollable overflow area for the un-cancelled arrow keys to move around.

### 2. Bomb hiss continues indefinitely — confirmed for remote-control bombs

There are two independent defects.

First, the server emits the singular event name `"detonate remote control bomb"` at `backend/server.js:276` and `backend/server.js:853`, but the client listens for the plural `"detonate remote control bombs"` at `frontend/index.js:1090`. Consequently, the client listener that stops all recorded fuses, empties `remoteControlFuses[index]`, and resets `isRemoteControlBombPlanted` does not run.

Second, game-over cleanup iterates entries shaped like `{ fuse, y, x }`, but assigns to the wrapper object's `src`:

```js
for (const fuse of remoteControlFusesArray) {
  if (fuse) {
    fuse.src = "";
  }
}
```

This is at `frontend/index.js:1183-1189`. It needs to stop/reset the nested audio element (`fuse.fuse`), so the looping `<audio>` clone can continue after the game ends. This directly explains the separately reported remote-control fuse continuing until reload.

Normal bomb fuses have a second confirmed end-game leak. They are stored only on `gridData[y][x].bomb`, not in the collection traversed by game-over cleanup (`frontend/index.js:1072-1077`, `1183-1190`). The server's bomb timeouts continue after game over, but the client returns immediately from late `"add fire"` events when `isGameOver` is true (`frontend/index.js:1032-1035`). That skips `triggerBombSound`, which is the normal path that stops the looping fuse. This directly explains the more general “bomb hiss continued indefinitely” report when a normal bomb is active as the game ends.

### 3. Player stuck on “game in progress” — strong architectural cause

When a connection arrives while `isGameInProgress` is true, the server sends `"game in progress"` once and immediately returns (`backend/server.js:125-129`). That socket gets no normal per-connection handlers and no snapshot/version of server state.

The only route off the waiting screen is the one-shot `"play again"` broadcast in `restart()` (`backend/server.js:382-395`), or the client's 40-second reload workaround (`frontend/index.js:190-209`). Socket.IO preserves event order on a live connection, but it does not make a transiently disconnected client receive broadcasts that happened while it was absent. Reconnection is explicitly disabled at `frontend/index.js:111-113`.

This means the symptom is expected even without packet loss: game over sets `isAfterGame` but does not clear `isGameInProgress` (`backend/server.js:787-804`), so a reload still returns to the waiting screen until every original in-game socket has disconnected. A flaky hotspot adds another failure mode: if the waiting socket drops around the eventual `"play again"` broadcast, there is no state query or replay to correct it. The robust design would make the current phase queryable/sent on every connection and make phase transitions idempotent, rather than treating `"play again"` as a command that must be received exactly once.

There is also fragile accounting behind the transition: `playersInGame` is decremented on disconnects (`backend/server.js:282-295`) but not when players are eliminated normally; `restart()` waits for disconnect behavior rather than scheduling a server-owned end-of-game transition. That couples recovery to every original browser completing its outro/reload.

### 4. Full-fire after remote-control — partial explanation; invisibility not proven

The server always gives remote-control mode precedence:

```js
if (players[index].remoteControl) {
  plantRemoteControlBomb(y, x, index);
  return;
}
```

This is `backend/server.js:260-266`. Collecting full-fire does not replace remote-control mode. The next `X` therefore plants a remote-control bomb carrying the full-fire flag (`backend/server.js:645-674`), and Space detonates it. Only after that has consumed `player.fullFire` can a later bomb behave normally. That sequence matches most of the report.

The reviewed code does emit and render a remote-control bomb (`backend/server.js:662`, `frontend/index.js:1079-1087`), so the “invisible” part cannot be proved from the current version. A dropped render event is unlikely while the same Socket.IO connection remains live because messages are ordered. More likely possibilities are that this observation came from an older revision, or that another explosion/rendering race removed the cell's bomb classes. Instrumenting bomb IDs and state transitions would be needed to settle that part.

### 5. False disconnection / one ready button — likely state desynchronization

The client disables reconnection and reloads immediately on any socket disconnect (`frontend/index.js:111-118`). A brief transport loss can therefore create a genuine old-socket disconnect even while the replacement page/socket makes the person appear present again.

Server state is held in several separate mutable counters and arrays (`players`, `playersInChat`, `playersInCountdown`, `playersInGame`) rather than derived from one authoritative phase/member set. Events are broadcasts without a state revision. This permits two clients to display different UI after a transient disconnect/reload even if chat appears to work again.

The ready button is especially dependent on local event history: it is shown/hidden from `"player joined chat"` and `"player left"` events and a local `playersInLobby` count (`frontend/index.js:487-535`, `537-586`, and `588-637`). There is no periodic or post-reload lobby snapshot after the initial `"players"` message. This is a plausible cause, but the exact historical sequence cannot be proven statically.

There is also a definite listener-duplication bug: `"player joined chat"` adds `readyHandler` each time at `frontend/index.js:518`, while `transitionToChat()` adds it again at line 635. `addEventListener` deduplicates the exact same function/type/capture tuple while it remains registered, but hide/show transitions remove it only in one branch. The broader ready protocol is still non-idempotent: every accepted `"ready"` resets the server counters and broadcasts another countdown.

### 6. Countdown reload and `isDead` crash — confirmed unsafe races, exact old path uncertain

The protections now in `isDead` avoid several old failures, but the disconnection/countdown protocol remains racy:

- A disconnect during countdown decrements the global `playersInCountdown` regardless of whether that socket was one of the counted chat players (`backend/server.js:301-310`).
- Such a disconnected player is deliberately left in `players` and recorded by numeric index.
- `initializeGame()` later filters and reindexes `players` (`backend/server.js:407-412`) but tests `disconnectees.includes(i)` using the old indexes (`backend/server.js:420-426`). If filtering removed an earlier entry, the wrong player can be killed.
- Game start depends on every counted browser independently finishing a client-side countdown and sending `"start game"` (`backend/server.js:210-219`, `frontend/index.js:724-750`). Disconnects, duplicate ready events, pauses, or delayed packets can alter the count while those signals are in flight.

There is another restart/start race: `restart()` changes global state after 100ms, while an already accepted `"start game"` schedules `initializeGame()` after 500ms. Neither scheduled callback is cancelled by the other.

These races can produce stale, missing, or reindexed players, which is consistent with the old `isDead(player)` exception. The exact Safari sequence belongs to an older revision, so the original failing path is no longer recoverable with certainty from the present code.

The current defensive checks also stop too early. `isDead()` returns `false` for an invalid player, after which `gameLoop()` still reads `player.skates` and calls `move(player)` (`backend/server.js:507-518`). `walkable()` likewise reads `otherPlayer.position` without checking `otherPlayer` (`backend/server.js:588-594`). A stale/null player can therefore still crash the next operation even though the former “lightning conductor” now has guards.

### 7. Old `walkable` grid crash — the hotspot exposed an ordinary bounds bug

The current guard at `backend/server.js:595` is still unsafe if `y` itself is outside the array, because evaluating `grid[y][x]` first tries to index `undefined`.

The subsequent bounds expression at `backend/server.js:602-606` uses `||`:

```js
y > 0 ||
y < numberOfRowsInGrid - 1 ||
x > 0 ||
x < numberOfColumnsInGrid - 1
```

For any ordinary number this is effectively always true. The comparisons need to be conjunctive, and bounds must be validated before reading `grid[y][x]`. The hotspot did not make the grid sparse; lag, stale state, or a malformed client message only exposed this validation error.

## Additional network-related defects

### Client-controlled identity and coordinates can crash or corrupt the game — high severity

The `"move"`, `"bomb"`, and `"detonate remote control bombs"` handlers trust indexes supplied in messages (`backend/server.js:221-276`). Bomb placement also trusts client-supplied `y` and `x` (`backend/server.js:260-266`, `615-675`). None verifies that the index belongs to `socket.id`, that the player exists and is alive/in-game, or that the coordinates equal the authoritative player position and are in bounds.

Consequences include:

- a stale reindexed client controlling or bombing as a different player;
- `players[index]` exceptions after disconnect/reindex;
- arbitrary out-of-range `grid[y][x]` exceptions;
- one client detonating another player's remote bombs.

Even for a trusted local game, deriving identity from the closure's `player` object and validating phase/coordinates would remove a large class of intermittent network failures.

### Remote bombs remain armed in server state after forced detonation

`kill()` detonates every remote bomb but never empties `player.remoteControlBombs` (`backend/server.js:841-857`). It also contains the typo `players.plantedBombs--` at line 851 instead of decrementing the individual player. After a respawn, a later Space message can therefore try to detonate stale coordinates again, while the planted-bomb count can remain wrong.

Combined with the singular/plural event mismatch, the client also continues to believe its remote bombs are planted.

### Explosion timers can overwrite newer cell state

Every `addFire()` call schedules an unconditional `grid[y][x].type = "walkable"` one second later (`backend/server.js:729-754`). If a second explosion reaches the same cell before the first timer fires, the first timer can clear the second fire early. Old timers are not associated with a bomb/game generation, so delayed callbacks can also mutate a newer state after a restart.

This is the kind of timing defect that becomes more visible when clients render delayed event streams, although the authoritative bug is server-side timer ownership rather than the network itself.

### Protocol events are not phase-checked or idempotent

Most socket handlers remain active in every phase. Repeated `"color"` increments `playersInChat`; repeated `"ready"` resets countdown bookkeeping and launches another client countdown; any socket can send `"pause"`; and repeated `"start game"` increments a global signal count (`backend/server.js:171-219`). Delayed or duplicated application actions can therefore mutate the wrong phase.

Socket.IO normally delivers each emitted event once per live connection, but double clicks, old listeners, stale clients, or deliberate input are enough to trigger these paths. A server-owned state machine should reject events that are invalid for the socket's current phase.

One concrete manifestation is that `"ready"` is broadcast with `io.emit` to every socket. A client still entering its code/name or choosing a role handles that event while `phase !== "chat"` by calling `gameInProgress()` (`backend/server.js:192-203`, `frontend/index.js:691-695`). It can therefore be sent to the waiting screen even though it was connected before the game and never joined the countdown.

### Countdown can wait forever for clients

The authoritative server has no countdown deadline. It starts only after `numberOfStartSignalsReceived >= playersInCountdown` (`backend/server.js:210-218`), while each browser runs its own timer. A connected but throttled/suspended tab can indefinitely prevent initialization. Mobile browsers commonly throttle background timers, making this another likely mobile-network/device symptom.

### Normal bomb sound has no end-of-game cleanup

Normal fuse clones are stored only on `gridData[y][x].bomb`, not in a collection that game-over cleanup traverses (`frontend/index.js:1072-1077`). As detailed above, the client then deliberately ignores the late explosion event that would stop them.

### Destruction events can leave client powerup state stale

When fire destroys a powerup, the server sets `grid[y][x].powerup = null` before reading its name for the `"destroy powerup"` event (`backend/server.js:732-734`). The client consequently receives `powerupName: null` and cannot remove the specific CSS class (`frontend/index.js:1114-1118`). The client also emits a `"destroy"` event for every breakable fire tile at line 1049, but the server has no matching handler; authority is split/confused even though that extra event is currently harmless.

### Dropping a skate also disables remote-control

`Player.drop()` has no `break` after the `"skate"` case (`backend/classes/player.js:82-88`). Losing a skate therefore falls through and sets `remoteControl = false`. The server and client can then disagree about the expected bomb mode after a death/powerup drop, particularly because the client has no authoritative powerup snapshot.

## Suggested fix order

1. Stop trusting client indexes/coordinates; derive the player from the socket and validate phase and bounds before every grid access.
2. Move countdown and game-phase transitions to a server-owned state machine with a deadline and a state snapshot on every connection/reconnection.
3. Use one remote-detonation event name and centralize cleanup of every active audio instance on detonation, game over, disconnect, and page transition.
4. Fix `walkable` bounds ordering/logic and make delayed timers belong to a specific game generation.
5. Cancel arrow-key defaults during gameplay and avoid scaling the entire body above 100%.

