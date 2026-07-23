# Bugs

## Current

- In Chrome for one player with a big screen, arrow keys caused the game grid to move around a very big, scrollable area, even when zoomed right out.
- On one occasion, when two people were playing abomb hiss continued indefinitely. I don't know the exact circumstances.

## Seen on previous versions

More play testing would help to replicate any of the following that are still happening and investigate the causes, as well as revealing any yet-to-be-discovered bugs.

- Server crashed twice on game of 4 players over mobile hotspot at `(player.bombPass && grid[y][x]?.type === "bomb")` in `walkable` before I added a check to make sure `grid[y][x]` is defined.
- Sometimes, after a game, one or more players would get stuck on the "game in progress" screen. Only seen over mobile hotspot. Since I saw that, I'm having the client reload the page every 40s while on the game-in-progress screen just in case it's no longer in progress but somehow they didn't receive the signal to that effect.
- On one occasion, collecting full-fire after remote-control, at first 'X' seemed to be doing nothing, but it did plant an invisible bomb, which exploded normally on SPACE, and only then could the full-fire be planted. I fixed another thing: making sure you can't plant a remore-control bomb on a soft block, in case you have soft-block-pass, and tested many times but, since then, I've yet to get this glitchy interaction between full-fire and remote-control to recur.
- On one occasion, one player heard the remote control fuse sound even after the game was over, until the page was reloaded; this on the same occasion as the next two bugs when 4 people played over a mobile hotspot.
- Possibly fixed now that I dealt with another disconnection-related error, but, just in case, here's a note of what happened: I once saw a bug where server and client logged that one of two players had disconnected, but they hadn't. Both players were still in the chat. Only one had the ready button visible. On pressing it, the countdown was triggered for both. Haven't manage to replicate it.
- A bug I saw once, but haven't managed to replicate after many attempts, possibly already fixed now that disconnections during countdown are handled better. But I'll leave the details here just in case. Server crashed when a player in Safari pressed CTR+SHIFT+R to view simplified page, without styles, during countdown. Apparently this led to them being undefined even though the normal disconnection logic had not gone ahead. It triggered that classic lightning-conductor-of-errors, `isDead(player)`: `return grid[player?.position?.y][player?.position?.x].type === "fire";` (accusing arrow points to 2nd instance of player in the line), "TypeError: Cannot read properties of undefined (reading 'undefined')". Since then I've added some protections and logging in case of future issues.
