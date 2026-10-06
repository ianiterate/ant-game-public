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
- **Build:** 2026-10-06 (c2bd9be)

## What you can do right now

You have one year. Walk out from the nest mound and find food: sugar cubes, seeds, leaves,
dead insects. Press **Mark** next to a find and walk home; the path you walk becomes a
pheromone trail, idle workers follow it out, and when enough of them are there the party
carries the find back. Standing at the nest, **Tap** sends a few workers down your trail,
and **Interact** opens the nest: a cutaway where you dig brood, store and processing
chambers with the food you bring in. The queen lays while there is food, brood becomes
workers in a week, and the colony's tier rises with its size and chambers.

The garden fights back. Rain turns outbound workers home and washes trails out; walk a
trail again to renew it. Spiders settle on busy trails and take workers: **Mark** the
spider and walk home to call a party of defenders. A bird's shadow follows a worker for a
few seconds before it strikes; **Interact** near it to raise the alarm. A rival colony
raids the nest and loots the stores; be home when they come. If you are taken, you return
as a fresh worker after a few seconds. Autumn slows the queen and winter brings nothing new
and costs more, unless you have thatched the nest with pinecones. The year ends when spring
returns; the game saves itself in your browser each day. Click once in the page to lock the
mouse; Esc releases it.

## Controls

| | Move | Look | Mark trail / call defenders | Tap (at nest) | Nest / alarm / confirm | Sprint | Pause / release mouse |
|---|---|---|---|---|---|---|---|
| Keyboard + mouse | W A S D | mouse | F | Q | E | Shift | Esc |
| Gamepad | left stick | right stick | X | Y | A | left trigger | Start |

Browsers only expose gamepads after a first click or key press, so press something once
and the pad will be picked up.
