# The Mad Bomber's Tea Party

![red](red.jpg)

1. [Background](#background)
2. [Setup](#setup)
3. [Progress](#progress)
4. [Framework](#framework)

## 1. Background

This is our take on the final project of the JavaScript section of the [01Founders course](https://01edu.notion.site/Global-01-Curriculum-50b7d94ac56a429fb3aee19a32248732), a branch of the [01Edu](https://01-edu.org/pedagogy) coding bootcamp system. The [brief](https://github.com/01-edu/public/tree/master/subjects/bomberman-dom) asks us to make a browser game for two to four players, based on the classic 1983 [Bomberman](https://en.wikipedia.org/wiki/Bomberman), using web sockets, but no WebGL or any 3rd-party framework.

This game was adapted from a single-player original by some of our fellow students: Peter Coles, Michael Adeleke, and Mohammed Rashidur Rahman.

[Other features](docs/further.md) may be added eventually.

## 2. Setup

Clone the repo and navigate into the backend folder:

```zsh
git clone https://github.com/pjtunstall/mad
cd mad/backend
```

Install Node dependencies with `npm install`. Then run `node server.js` to start the server on port 3000.

The server will log its IP address in the terminal. To connect over a mobile hotspot, players can enter that address into their browser. As yet it only supports a single instance of the game.

Note: the instructions ask us to have a lobby with a 20s countdown, followed by a 10s countdown for whoever has joined during the first countdown. Instead, we chose to implement a 10s countdown only. The two countdowns didn't make dramatic sense, especially in the context of our more elaborate intro. Also, 20s is not long to chat! (The instructions don't make it clear whether they expect the chat to continue during the game. We assumed not.)

## 3. Progress

This multiplayer version can be played in Chrome, Brave, Firefox, Edge, and Safari. As in its single-player precursor, all bonus powerups are implemented except bomb-push (throw). Intermittent [bugs](docs/bugs.md) remain. To tackle these, more play testing would be needed under various conditions and on different devices. It's still too sensitive to screen dimensions. Size, proportions, and alignment could be fixed better, leaving less to chance, especially on the intro. See [Further](docs/further.md) (in particular, DESIGN) for more detail.

UPDATE (2026): An [AI code review](docs/ai-bug-review/ai-bug-review.md) identified many bugs and possible causes. At some point, it will be interesting to check through these, apply the obvious fixes (such as typos), and test the less obvious ones individually.

## 4. Framework

In the version submitted for 01Founders, we used my homemade frontend framework, [overReact](https://github.com/pjtunstall/overreact), in a minimal way, just to construct the grid, so as to meet the requirements for the exercise. For more detail, see this [discussion](docs/framework.md), which includes the original plan and thoughts on how it might be developed it further. I've removed this dependency now as it was superfluous to the needs of the game.
