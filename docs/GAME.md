# ant-game — design

**What the game is right now.** Present tense, no history. Run `/open-questions` to
see everything currently waiting on a human call.

> This file is published verbatim to the public play repository on every web deploy.
> Write it knowing players may read it.

---

## Premise

You are one ant. The garden is enormous: a grass blade is a tree, a sugar cube is a
boulder, a dead beetle is a week of food. Nothing worth having can be moved by one ant,
so the game is about getting the colony to help — and about keeping the colony alive
through everything a real colony faces.

Art direction: **stylised realism** — macro-photography lighting and materials with
simplified shapes, built for URP on WebGL.

## Core loop

1. **Explore** the garden patch at ant scale in a third-person follow camera.
2. **Find** something: a sugar cube, a leaf, a seed, a dead insect. Each needs a minimum
   number of ants to haul and a minimum colony tier to process.
3. **Mark a trail**: press Mark at the find and walk home. The trail you walk is laid as
   pheromone from nest to find.
4. **Ants come.** Idle workers follow the trail at a rate set by its strength, until the
   party is big enough. You can also stand at the nest and *tap* to send a few directly.
   Tapping an ant out in the field is a later question (see Undecided).
5. **Haul.** Once enough ants are at the find, the party carries it home. Food goes into
   the colony's stores; the colony grows.

The first playable slice is exactly this loop with one sugar cube, done in under three
minutes.

## World scale and movement

**Assumed** — 1 Unity unit = 1 cm. A worker is about 1 unit long (slightly large, for
readability). The first patch is 200 × 200 units — a 2 m square of garden. Expensive to
change once assets exist: every mesh re-export and every speed retuned.

Walking is on the **ground and gentle slopes** — leaves and stones lying on the ground are
walkable; vertical surfaces are not. Climbing stems, walls and leaf undersides is a later
question (see Undecided).

## The colony

The colony is counts, not characters: idle workers, nursing workers, brood, food. Workers
leave the nest only on a job (hauling, later building and defending) and return to the
idle pool afterwards. The queen and brood produce new workers while there is food; every
worker costs upkeep per day. Income is capped by what the garden offers, so population
converges to what the stores can carry.

**Colony tier** is derived from population (roughly <25 / 25–75 / 75+) and which chambers
exist. Bigger finds need a higher tier: a dead insect needs a processing chamber, not just
hands.

Nursing workers are reserved: recruitment draws only from the idle pool, and the HUD shows
the nurses as reserved. Whether a big haul can ever pull nurses, and the brood-starvation
tension that would come with it, is a colony-building question (see Undecided).

## Finds

| Find | Ants to haul | Tier | Food | Notes |
|---|---|---|---|---|
| Sugar cube | 6 | 1 | 60 | The first slice's find. |
| Seed | 2 | 1 | 15 | Common. |
| Leaf | 4 | 1 | 10 | Low value, light; nest material later. |
| Dead insect | 10 | 2 | 120 | Needs a processing chamber; spoils. |
| Pinecone | 16 | 3 | — | Shelter material, not food. |

The player counts as one ant toward starting the haul, while standing within reach of the
find; the party then carries it home whether or not you stay. Numbers are the first cut and live in
`Assets/Settings/`, not in code.

## Pheromone trails

A trail is a path with one strength. It decays on its own and is reinforced by every ant
that walks it, so an active haul keeps its own trail alive and a finished one fades. Rain
washes trails out quickly. One player-laid trail at a time: pressing Mark again while
walking home abandons the one being laid, and completing a new one replaces the old. The
point is that the player's movement *is* the order, not a cursor.

## Time, weather, threats (later milestones)

A day is **Assumed** six real minutes, a season ten days. Night slows foraging and brings
predators. Weather is clear / overcast / rain; rain reverses outbound ants and washes
trails. Threats: spiders on patrol, a bird taking ants off a trail segment, raids from a
rival colony, winter with no new finds. Each is arithmetic against the colony's counts,
not a boss fight.

## Simulation

The full M1 rules — items, colony upkeep, trails, recruitment, hauling, tick order and
tests — are specified in [SIM.md](SIM.md). Everything the player reads is in
[UI_COPY.md](UI_COPY.md).

The simulation is headless: a fixed 10 Hz tick, one seeded random source, systems run in
a fixed order, no scene or renderer involved. The view reads the simulation; the simulation
does not know the view exists. **Assumed** — reproducible on a single build target only
(no fixed-point maths); cross-platform replays or lockstep multiplayer would be a rewrite.

**Assumed** — NPC ants have no identity, health or age: they are counts in the nest and
lightweight records on a trail when outside. Expensive to change if named ants are ever
wanted; cheap otherwise, and it is what makes hundreds of ants free on WebGL.

**Assumed** — saves are JSON in browser storage with a version field from day one; a save
from an older major version starts a new game.

## Controls

Keyboard + mouse: WASD move, mouse look, E interact, F mark trail, Q tap an ant, Shift
sprint, Esc pause. Gamepad: left stick move, right stick look, A interact, X mark, Y tap,
left trigger sprint, Start pause. Desktop
browsers only for now (**Assumed** — mobile needs a texture and UI pass).

---

## Undecided

**Undecided** — nest interior: walkable 3D tunnels you enter, or a 2D cutaway panel.
Blocks colony building (M2). Walkable roughly doubles that milestone.

**Undecided** — session shape: open-ended sandbox with a persistent colony, or a one-year
run that ends with an outcome. Blocks pacing, spawn curves and save semantics (M2).

**Undecided** — death: when the player ant dies, respawn as a fresh worker, or game over;
and whether the queen's death ends the colony. Affects tone, saves and threat tuning (M3).

**Undecided** — what tapping an ant away from the nest does (redirect it, recruit it, or
nothing). Blocks multi-trail play in M2.

**Undecided** — how workers move between idle and nursing duty, and whether a large
recruitment can pull nurses off the brood. Blocks the queen and brood (M2) and the HUD's
nursing warning.

**Undecided** — how much strategy-game is in it: embodied-only recruiting (trail and tap),
or also orders issued from the colony panel. Affects UI from M2.

**Undecided** — species fidelity: a real species (black garden ant, *Lasius niger*) with
real behaviours and threats named in-game, or a fictional colony. Affects art and all text.

**Undecided** — climbing: full surface walking (stems, walls, leaf undersides) later, or
never. A one-to-two-week feature that changes the controller, camera and simulation.

**Undecided** — licence of the public play repository (code and docs are visible; which
terms apply).

## Assumed (summary)

Listed inline above or in [SIM.md](SIM.md): the game starts at dawn; 1 u = 1 cm; 10 Hz tick, 2D on the ground plane, sim pauses when the
tab is hidden; float determinism on one target; NPC ants are counts; trail-graph pheromones
rather than a diffusion grid; player counts as one ant; six-minute day, ten-day season;
Input System, Cinemachine 3, UI Toolkit; WebGL2 with gzip + decompression fallback; desktop
browsers only; versioned JSON saves; commercial-safe asset licences only.
