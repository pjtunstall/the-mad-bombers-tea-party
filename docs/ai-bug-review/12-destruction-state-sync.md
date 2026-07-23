# Destruction events leave client state stale

## Finding

When fire destroys a powerup, the server clears the powerup before reading its name for the event. The client therefore receives `null` and cannot remove the specific powerup CSS class. The client also emits a `"destroy"` request that the server never handles, indicating unclear ownership of destruction state.

Relevant code:

- `backend/server.js:729-755`
- `frontend/index.js:1032-1052`, `1101-1118`

## Offending code

```js
} else if (grid[y][x].powerup) {
  grid[y][x].powerup = null;
  io.emit("destroy powerup", {
    y,
    x,
    powerupName: grid[y][x].powerup,
  });
}
```

`powerupName` is always `null`.

The client separately sends:

```js
if (gridData[cellData.y][cellData.x].type === "breakable") {
  socket.emit("destroy", { y: cellData.y, x: cellData.x });
}
```

There is no `socket.on("destroy")` on the server. The server is already authoritative and emits `"destroy block"` itself.

## Proposed change

Capture data before mutation and send one authoritative cell update:

```js
function destroyPowerup(y, x) {
  const cell = grid[y][x];
  const powerup = cell.powerup;
  if (!powerup) return;

  cell.powerup = null;
  cell.revision++;

  io.emit("cell:updated", {
    y,
    x,
    revision: cell.revision,
    type: cell.type,
    powerup: null,
    removedPowerupName: powerup.name,
  });
}
```

The client applies the resulting state:

```js
socket.on("cell:updated", (update) => {
  const current = gridData[update.y][update.x];
  if (update.revision <= current.revision) return;

  gridData[update.y][update.x] = {
    ...current,
    ...update,
  };
  renderCell(update.y, update.x);
});
```

`renderCell` should derive all relevant classes from current data instead of trying to remember exactly which old class to remove:

```js
function renderCell(y, x) {
  const element = cellsArr[y][x];
  const cell = gridData[y][x];

  element.classList.remove(...POWERUP_CLASS_NAMES);
  element.classList.toggle("power-up", Boolean(cell.powerup));

  if (cell.powerup) {
    element.classList.add(cell.powerup.name);
  }
}
```

Delete the unmatched client `socket.emit("destroy", ...)`. If client animation feedback is desired, it should be triggered by the authoritative update, not a second mutation request.

## Verification

Destroy each powerup type with fire and inspect both `gridData` and DOM classes on every client. Reorder or repeat `cell:updated` messages in a test and verify that revisions prevent stale state from replacing newer state.
