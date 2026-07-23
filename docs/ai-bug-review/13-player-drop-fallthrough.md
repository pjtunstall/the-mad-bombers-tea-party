# Dropping a skate also disables remote-control

## Finding

The `"skate"` case in `Player.drop()` has no `break`, so it falls through into `"remote-control"`. Losing a skate consequently disables remote-control as an unrelated side effect.

Relevant code:

- `backend/classes/player.js:70-99`
- `backend/server.js:757-817`

## Offending code

```js
case "skate":
  if (this.skates > 0) {
    this.skates--;
  }
// No break.
case "remote-control":
  this.remoteControl = false;
  break;
```

This can be difficult to diagnose in multiplayer because the server changes bomb behavior, while the client has no complete authoritative powerup snapshot after the drop.

## Proposed change

The minimal fix is:

```js
case "skate":
  if (this.skates > 0) {
    this.skates--;
  }
  break;
case "remote-control":
  this.remoteControl = false;
  break;
```

A safer implementation avoids switch fallthrough entirely and reports the resulting state:

```js
drop(powerupName) {
  const dropHandlers = {
    "bomb-up": () => {
      this.maxBombs = Math.max(1, this.maxBombs - 1);
    },
    "fire-up": () => {
      this.fireRange = Math.max(1, this.fireRange - 1);
    },
    skate: () => {
      this.skates = Math.max(0, this.skates - 1);
    },
    "remote-control": () => {
      this.remoteControl = false;
    },
    "soft-block-pass": () => {
      this.softBlockPass = false;
    },
    "bomb-pass": () => {
      this.bombPass = false;
    },
    "full-fire": () => {
      this.fullFire = false;
    },
  };

  dropHandlers[powerupName]?.();
  return this.getPowerupState();
}
```

After a death/drop, emit the resulting authoritative state:

```js
io.emit("player:powerups-updated", {
  playerId: player.id,
  powerups: player.getPowerupState(),
});
```

This makes a server-side mode change visible and recoverable on every client.

## Verification

Unit-test every `drop()` case independently. Start with all flags enabled, drop one powerup, and assert that only the corresponding property changes. In a game, lose a skate while retaining remote-control and verify the next bomb remains remote-controlled on server and clients.
