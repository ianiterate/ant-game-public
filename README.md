# ant-game

A 3D ant colony simulation, in development. You are one ant. Explore a garden at ant
scale, find food that is far too big for you — a sugar cube, a leaf, a dead beetle — lay a
pheromone trail home, and watch your colony come to help. Grow the colony, and keep it alive
through everything a real colony faces.

**Play it here: https://ianiterate.github.io/ant-game-public/**

This repository holds the playable web build and the current design notes; the source lives
elsewhere.

- **Design:** [docs/GAME.md](docs/GAME.md) — what the game is right now, including what is
  still undecided.
- **Build:** 2026-10-05 (49d43eb)

## What you can do right now (first slice)

Walk out from the nest mound, find a sugar cube, press **Mark** next to it, and walk home.
The path you walk becomes a pheromone trail. Idle workers follow it out, and when six ants
are at the cube the party carries it back and the colony's food goes up. Standing at the
nest, **Tap** sends a few workers down your trail directly. Click once in the page to lock
the mouse; Esc releases it.

## Controls

| | Move | Look | Mark trail | Tap (at nest) | Sprint | Pause / release mouse |
|---|---|---|---|---|---|---|
| Keyboard + mouse | W A S D | mouse | F | Q | Shift | Esc |
| Gamepad | left stick | right stick | X | Y | left trigger | Start |

Browsers only expose gamepads after a first click or key press, so press something once
and the pad will be picked up.
