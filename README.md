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
- **Build:** 2026-10-10 (beabf95)

## What you can do right now

Each year has one measure: how many workers are alive when spring comes back. Survive it and you can carry the colony into a harder year, for as many years as it lasts. The
game says so when a colony begins, a winter bar in the corner shows stores against what winter
will cost, and each season sets three goals that explain themselves. Your antennae sense finds
within a hand's width: chevrons at the screen edge point at them, and the grass parts around
them.

You have one year. Walk out from the nest mound and find food: sugar cubes, seeds, leaves,
dead insects. Press **Mark** next to a find and walk home; the path you walk becomes a
pheromone trail, idle workers follow it out, and when enough of them are there the party
carries the find back. **Tap** at the nest sends a few workers down your trail, or out in the
field calls nearby workers off their job onto yours; **Interact** opens the nest: a living cutaway where you watch the queen lay, nurses tend the brood, stores pile up and diggers work, and where you dig brood, store and processing
chambers with the food you bring in. The queen lays while there is food, brood becomes
workers in a week, and the colony's tier rises with its size and chambers. Dead insects are cut up in the processing chamber before their food reaches the stores; winter, rain, hunger and raids all show in the cutaway.

The garden fights back. Rain turns outbound workers home and washes trails out; walk a
trail again to renew it. Spiders settle on busy trails and take workers: **Mark** the
spider and walk home to call a party of defenders. A bird's shadow follows a worker for a
few seconds before it strikes; **Interact** near it to raise the alarm. A rival colony
raids the nest and loots the stores; be home when they come. If you are taken, you return
as a fresh worker after a few seconds. Autumn slows the queen and winter brings nothing new
and costs more, unless you have thatched the nest with pinecones. When spring returns the year ends: carry on into the next, harder year, or
stay in the garden as a sandbox. The game saves itself in your browser each day. Click once in the page to lock the
mouse; Esc releases it.

## Controls

| | Move | Look | Mark trail / call defenders | Tap (call workers) | Nest / alarm / confirm | Sprint | Pause / release mouse |
|---|---|---|---|---|---|---|---|
| Keyboard + mouse | W A S D | mouse | F | Q | E | Shift | Esc |
| Gamepad | left stick | right stick | X | Y | A | left trigger | Start |
| Touch | joystick under the left thumb | drag on the right half | Mark | Tap | Interact (shows Nest or Alarm) | Run (tap on, tap off) | Menu |

Phones and tablets work in landscape, with the on-screen controls above; held upright, the page asks you to turn the device.

Browsers only expose gamepads after a first click or key press, so press something once
and the pad will be picked up.

## Licence

The design documents in `docs/` are licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
The game build is free to play here and on itch.io and may not be redistributed. Third-party
sound is CC0 and credited in the game's credits panel and in `docs/`'s attribution. The
source code is not published.
