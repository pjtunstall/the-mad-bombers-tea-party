# Grid bounds are checked incorrectly and too late

## Finding

The bounds expression in `walkable()` uses OR where all four conditions must be true. It also indexes the grid before proving that the row exists. The hotspot did not make JavaScript arrays unreliable; it made stale or malformed state more likely to reach an unsafe code path.

Relevant code:

- `backend/server.js:522-613`
- `backend/server.js:824-839`

## Offending code

```js
if (!grid[y][x]) {
  console.log(
    `${player.role} tried to move to grid[${y}][${x}], which is ${grid[y][x]}`
  );
  return false;
}

const isInBounds =
  y > 0 ||
  y < numberOfRowsInGrid - 1 ||
  x > 0 ||
  x < numberOfColumnsInGrid - 1;
```

If `y` is outside the array, `grid[y]` is `undefined`, so evaluating `grid[y][x]` throws before the guard can run. For normal numeric values the OR expression is effectively always true; for example, `y = -100` still satisfies `y < numberOfRowsInGrid - 1`.

`isDead()` has a related partial guard:

```js
return grid[player?.position?.y][player?.position?.x]?.type === "fire";
```

Optional chaining on the final cell does not protect `grid[undefined][...]`.

## Proposed change

Create one bounds helper and call it before every grid lookup:

```js
function isInsideGrid(y, x) {
  return (
    Number.isInteger(y) &&
    Number.isInteger(x) &&
    y >= 0 &&
    y < numberOfRowsInGrid &&
    x >= 0 &&
    x < numberOfColumnsInGrid
  );
}

function isInsidePlayableArea(y, x) {
  return (
    isInsideGrid(y, x) &&
    y > 0 &&
    y < numberOfRowsInGrid - 1 &&
    x > 0 &&
    x < numberOfColumnsInGrid - 1
  );
}
```

Then:

```js
function walkable(y, x, player) {
  if (!isValidActivePlayer(player)) return false;
  if (!isInsidePlayableArea(y, x)) return false;

  const destination = grid[y][x];
  if (!destination) return false;

  for (const otherPlayer of players) {
    if (!isValidActivePlayer(otherPlayer)) continue;
    if (
      otherPlayer.id !== player.id &&
      otherPlayer.position.y === y &&
      otherPlayer.position.x === x
    ) {
      return false;
    }
  }

  return (
    destination.type === "walkable" ||
    destination.type === "fire" ||
    (player.softBlockPass && destination.type === "breakable") ||
    (player.bombPass && destination.type === "bomb")
  );
}
```

And:

```js
function isDead(player) {
  const y = player?.position?.y;
  const x = player?.position?.x;
  if (!isInsideGrid(y, x)) return false;
  return grid[y][x]?.type === "fire";
}
```

Bomb placement, detonation, fire propagation, spawning, and powerup drops must use the same helper. Input validation at only one call site leaves other grid accesses vulnerable.

## Verification

Unit-test boundary values `-1`, `0`, last valid index, and one past the end for each axis, plus `undefined`, `NaN`, floats, strings, and very large values. Fuzz socket payloads and assert that no handler throws or mutates a cell outside the playable area.
