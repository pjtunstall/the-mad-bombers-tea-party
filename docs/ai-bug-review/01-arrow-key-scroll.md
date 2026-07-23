# Arrow keys scroll the game on large screens

## Finding

This is a confirmed client-side input/layout bug. During a game, arrow keys are handled as controls but their normal browser scrolling action is not cancelled. The effect is unusually dramatic on large screens because the entire document can be transformed to a scale greater than 100%.

Relevant code:

- `frontend/index.js:758-760`
- `frontend/index.js:935-943`
- `frontend/styles.css:685-693`

## Offending code

```js
const factor = Math.min(screen.height / 768, screen.width / 1366);
document.body.style.transform = `scale(${0.5 * factor})`;
```

On a 3840×2160 screen, for example, the computed scale is approximately `1.4`. A transform changes visual rendering without making the surrounding layout responsive in the usual way, so it can produce a large overflow area.

The keyboard handler then leaves the browser default enabled:

```js
let onKeyDown = (e) => {
  switch (e.key) {
    case "ArrowUp":
    case "ArrowDown":
    case "ArrowLeft":
    case "ArrowRight":
      socket.emit("move", { index: ownIndex, key: e.key });
      anticipateMove(e.key);
      break;
  }
};
```

## Proposed change

Cancel default behavior only for keys that the active game consumes:

```js
const MOVEMENT_KEYS = new Set([
  "ArrowUp",
  "ArrowDown",
  "ArrowLeft",
  "ArrowRight",
]);

function onKeyDown(event) {
  if (MOVEMENT_KEYS.has(event.key)) {
    event.preventDefault();
    socket.emit("move", { key: event.key });
    anticipateMove(event.key);
    return;
  }

  if (event.key.toLowerCase() === "x") {
    event.preventDefault();
    requestBombPlacement();
    return;
  }

  if (event.code === "Space") {
    event.preventDefault();
    requestRemoteDetonation();
  }
}
```

The same cancellation should be applied to arrow-key `keyup`, because some browsers can scroll on either part of the key interaction:

```js
function onKeyUp(event) {
  if (!MOVEMENT_KEYS.has(event.key)) return;

  event.preventDefault();
  direction[ownIndex] = { y: 0, x: 0, key: "" };
  socket.emit("stop");
}
```

The larger layout fix is to scale a dedicated game element and cap the scale, rather than transforming `body`:

```js
const viewportScale = Math.min(
  window.innerHeight / 768,
  window.innerWidth / 1366,
  1
);

game.style.setProperty("--game-scale", viewportScale);
```

```css
#game {
  transform: scale(var(--game-scale, 1));
  transform-origin: top center;
}

body.game-active {
  overflow: hidden;
}
```

Using `window.innerWidth`/`innerHeight` measures the available browser viewport rather than the physical screen.

## Verification

Test Chrome at normal size, zoomed out, and on a 4K display. Arrow keys should move the player without changing `window.scrollX` or `window.scrollY`, and the game should never render larger than the available viewport.
