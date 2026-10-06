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

The colony is counts, not characters: idle workers, nurses, diggers, brood, food. Workers
leave the nest only on a job (hauling or defending) and return to the idle pool
afterwards. Every worker and every brood eats each day. The queen lays while the stores are
healthy, up to what the brood chambers hold, and brood becomes workers after seven days.
Income is capped by what the garden offers, so population converges to what the stores can
carry. When the stores are empty the queen stops laying, brood is lost, and about one worker
in ten starves each day. A colony with no workers and no brood is dead.

**Assumed** — the queen lays up to 5 eggs a day while the stores hold two days of food, and
fewer when they hold less. Brood and workers each eat half a food a day. Starvation kills
about 10% of the workers in the nest per day. Cheap to change: numbers in `Assets/Settings/`.

**Colony tier**: Young below 25 workers. Established at 25 workers with a processing chamber.
Mature at 75 with a processing chamber and two each of brood and store chambers. A tier is
lost only when the colony falls 5 below its threshold. Bigger finds need a higher tier: a
dead insect needs an Established colony: a processing chamber and 25 workers, not just hands. **Assumed** — the chamber requirements
and the margin of 5. Cheap to change.

**Nursing**: the brood needs nurses, one for the queen plus one per five brood, and idle
workers fill that need before anything else. Trails never take nurses. A tap at the nest when
no idle worker is left can take nurses, down to half of what the brood needs. Brood left
short-handed dies off until the nurses come back. **Assumed** — this rule, and how fast
under-nursed brood dies. Cheap in code; it decides how many idle workers the early game has.

## The nest

The nest is managed from a **cutaway panel**, not walked through. Stand at the entrance and
the colony opens as a side-on view of its chambers drawn on the mound: brood, stores,
processing, and the slots where the next chamber can be dug. Building is a choice there,
paid in food and worker-time; the digging happens off screen as counts. You never lose the
third-person view or pick up a cursor. Walkable tunnels, if they ever come, are presentation
only — a space for story, not a second way to manage the colony.

There are three chambers:

- **Brood**, 30 food: room for 12 more brood.
- **Store**, 20 food: room for 150 more food. Anything delivered beyond the room is lost.
- **Processing**, 60 food: needed to become Established.

The panel is worked with the same inputs as walking: Move left and right picks a slot, Interact
digs or confirms, Mark or Pause closes it. Nothing in the game needs a pointer. **Assumed** —
direction-and-confirm navigation. Cheap to change; it is UI only.

The colony starts with one brood chamber, one store chamber and 20 food, too little to dig
anything: the first hauls pay for the nest. The cutaway has 4 slots to dig
in, 7 once the colony is Established and 10 once it is Mature. One chamber is dug at a time,
by four workers, in a day (a day and a half for processing). A dig can be cancelled for its food
back, as far as the stores have room for it. **Assumed** — the chamber kinds, costs and slot counts, and one dig at a time.
Cheap to change, except the slot layout, which the panel art is drawn around. **Assumed** —
the panel orders digging only; recruiting stays with trails and taps.

## The session

A game is **one year**: four seasons of ten days, about four hours at the six-minute day.
The year ends with an outcome — the colony's size and stores as winter breaks, or its
death — and then keeps going as an open sandbox for anyone who wants to stay. Pacing,
finds and threats are tuned to that clock; the save holds one colony.

The seasons change the garden and the colony. Finds appear through spring, summer and autumn,
and none appear in winter. Seeds and leaves are most common in autumn, dead insects in summer,
and pinecones come through summer and autumn. The queen lays half as much in autumn and not at
all in winter. In winter each worker eats a quarter more, unless the nest has been thatched
with pinecones. The outcome is the number of workers alive as the new spring begins, shown
with the stores, the peak and the year's totals. If every worker and every brood is lost
first, the outcome is the colony's death and the day it came, and the screen offers a new
colony in the same garden; the old save is replaced. **Assumed** — starting over after the
colony dies is always offered. It is separate from the player ant's own death (see Time,
weather, threats). Cheap to change. **Assumed** — these season
effects and the headline measure. Cheap to change.

## Finds

| Find | Ants to haul | Tier | Food | Notes |
|---|---|---|---|---|
| Sugar cube | 6 | 1 | 60 | The first slice's find. |
| Seed | 2 | 1 | 15 | Common. |
| Leaf | 4 | 1 | 10 | Low value, light; nest material later. |
| Dead insect | 10 | 2 | 120 | Needs a processing chamber; spoils over three days. |
| Pinecone | 16 | 3 | — | Shelter, not food: each one hauled home thatches the nest against winter (up to four). |

The player counts as one ant toward starting the haul, while standing within reach of the
find; the party then carries it home whether or not you stay. Numbers are the first cut and live in
`Assets/Settings/`, not in code.

Finds appear over time at random places around the nest. Each lies there a while before it is
gone: seeds for two days, leaves for a day and a half, a sugar cube for three, a pinecone for
four. A find you have marked stays until it is hauled. There are no scout ants: every find is
found and marked by you. **Assumed** — the spawn rates and lifetimes, and no scouts. Cheap to
change, though scouts would need the economy retuned so that they do not replace the player.

## Pheromone trails

A trail is a path with one strength. It decays on its own and is reinforced by every ant
that walks it, so an active haul keeps its own trail alive and a finished one fades. Rain
washes trails out quickly. Walking a marked find's trail home again, while workers are still
being called to it, renews it at full strength. One player-laid trail at a time: pressing Mark again while
walking home abandons the one being laid, and completing a new one replaces the old. The
point is that the player's movement *is* the order, not a cursor.

## Time, weather, threats

A day is **Assumed** six real minutes, a season ten days. Night slows everyone outside the nest.
No threat comes on the first day.

**Weather** is clear, overcast or rain, and changes only on the quarter day. Rain never follows
clear directly: overcast comes first, and the HUD shows rain one quarter day before it starts.
Overcast and rain bring slightly fewer finds. Rain calls home every worker on the way out or
waiting at a find, stops trails calling new workers, and washes trails down to almost nothing in
seconds. Parties already carrying keep going. When it stops, walk your trail again to renew it.
Spring and autumn are wettest, summer driest.

**Spiders** arrive at night and stay two days. Each settles on the busiest trail and walks a small
loop across it; when that trail goes quiet it moves to whichever trail the colony is using next. A
worker within reach is taken now and then, about one for every ten seconds workers spend near it,
and each loss weakens the trail. Mark a spider the way you mark a find and walk home: workers
gather just short of it, and six of them drive it off without loss. Fewer still win but lose some;
two are lost. If rain washes the defenders' trail, mark the spider again to renew it. Spiders hide
in rain.

**Birds** hunt by day, never in rain. A shadow follows one worker on the trails for five seconds,
then takes up to three workers near it and scatters the trail. Stand near the shadow and press
Interact to raise the alarm: the workers scatter and none are taken, though the strike still lands: it scatters the trail, and it kills you if you are standing under it.

**Raids** come from a rival colony you never see. It grows through the year and sends raiders, more
of them the bigger your colony's tier, across the garden from its edge. When raiders are spotted,
every worker on the way out turns home. The fight is at the nest, against every worker inside: a
fuller nest beats a raid cheaply, and the defenders fight harder while you are there with them.
Raiders loot the stores while they fight, and for a while longer if they break through. Every
raider killed weakens the rival. No raids in rain or winter.

Each threat is arithmetic against the colony's counts, not a boss fight. The queen and the brood
are never targets. A reasonable player loses about one worker in eight that they raise to threats
and still ends the year alive; ignoring rain or the defence costs far more. The rules and numbers
are in [SIM_M3.md](SIM_M3.md).

**Assumed** — the player ant dies to threats and comes back: within a spider's reach or a raid's
path for about three seconds, or under a bird's strike, you are lost, and ten seconds later you
leave the nest as a fresh worker, the colony one worker smaller. Nothing else is lost, and the queen
cannot die. Cost to change: making the player's death end the game means a new
outcome, a save that ends at death and a retune of every threat (about a day plus a balance pass);
letting the queen die means new colony-death rules and redoing the raid arithmetic (a few days).

**Assumed** — the weather and threat numbers, that spiders follow the colony's traffic, that a spider
is defended by marking it (not by a tap near it or an order from the nest) and a full party loses
nobody, that the alarm uses Interact, and that the nest's defence
against a raid is automatic. Cheap to change; none of it decides what a tap away from the nest does.

## Simulation

The full M1 rules — items, colony upkeep, trails, recruitment, hauling, tick order and
tests — are specified in [SIM.md](SIM.md). Colony building — chambers, queen and brood, tier,
finds that come and go, the year and its outcome, saves — is in [SIM_M2.md](SIM_M2.md). Weather,
spiders, birds, raids and the player ant's death are in [SIM_M3.md](SIM_M3.md).
Everything the player reads is in [UI_COPY.md](UI_COPY.md).

The simulation is headless: a fixed 10 Hz tick, one seed (driving one random stream for the
colony and finds and a second for weather and threats), systems run in a fixed order, no scene or renderer involved. The view reads the simulation; the simulation
does not know the view exists. **Assumed** — reproducible on a single build target only
(no fixed-point maths); cross-platform replays or lockstep multiplayer would be a rewrite.

**Assumed** — NPC ants have no identity, health or age: they are counts in the nest and
lightweight records on a trail when outside. Expensive to change if named ants are ever
wanted; cheap otherwise, and it is what makes hundreds of ants free on WebGL.

**Assumed** — saves are JSON in browser storage with a version field from day one, one slot,
written each game day and whenever the game loses focus, pauses or closes. A save too old to convert starts a new colony; a save from a newer
build is never overwritten.

## Controls

Keyboard + mouse: WASD move, mouse look, E interact, F mark trail, Q tap an ant, Shift
sprint, Esc pause. Gamepad: left stick move, right stick look, A interact, X mark, Y tap,
left trigger sprint, Start pause. Desktop
browsers only for now (**Assumed** — mobile needs a texture and UI pass).

---

## Undecided

**Undecided** — what tapping an ant away from the nest does (redirect it, recruit it, or
nothing). Blocks multi-trail play, which M2 does not include.

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
browsers only; versioned JSON saves; commercial-safe asset licences only. From
[SIM_M2.md](SIM_M2.md): nursing need and taps that take nurses; queen laying, brood and
starvation rates; chamber kinds, costs and slots; tier chamber requirements; find spawning and
lifetimes; pinecones as shelter; winter costs more; outcome = workers alive as spring begins;
no scout ants; the panel orders digging only. From [SIM_M3.md](SIM_M3.md): the player ant
respawns and the queen cannot die; the weather and threat numbers; spiders defended by marking;
the alarm on Interact; automatic nest defence; re-walking a trail renews it; a separate random
stream for weather and threats; colonies saved before M3 continue with fresh weather and threats.
