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
   Out in the field, tapping redirects: a few nearby workers leave what they were doing and
   join your current trail.
5. **Haul.** Once enough ants are at the find, the party carries it home. Food goes into
   the colony's stores; the colony grows.

The first playable slice is exactly this loop with one sugar cube, done in under three
minutes.

## World scale and movement

**Assumed** — 1 Unity unit = 1 cm. A worker is about 1 unit long (slightly large, for
readability). The first patch is 200 × 200 units — a 2 m square of garden. Expensive to
change once assets exist: every mesh re-export and every speed retuned.

Walking is on the **ground and gentle slopes** — leaves and stones lying on the ground are
walkable; vertical surfaces are not, and they stay that way: the garden is read from the
ground, and a wall is a wall.

An ant that walks into a low ledge climbs it: you get out of holes (the nest shaft) and
over anything up to about an ant-length tall. Taller than that is a wall. Grass stems are
never climbed. **Assumed** — the limit is 1.2 units above the ant's feet. Cheap to change
(one tuning value), but it decides which pebbles and stones are routes and which are
walls, so the level layout leans on it.

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
paid in food and worker-time; the digging is simulated as counts, and the cutaway shows the crew at work. You never lose the
third-person view or pick up a cursor. Walkable tunnels, if they ever come, are presentation
only — a space for story, not a second way to manage the colony.

The cutaway is being made into a living view of the nest. The queen lays, brood grows from egg
to larva to pupa and hatches, nurses tend it, store heaps rise and fall with the food, and diggers
open new chambers. Raids, winter and hunger show too, so the colony's state reads at a glance.
It only shows the colony; it changes no rule. **Assumed** — a small 3D diorama inside the panel,
built in stages starting with the queen, brood, stores and digging. The design is in
[NEST_VIEW.md](NEST_VIEW.md).

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

## Purpose

The year asks one thing: have as many workers alive as you can when spring returns. The game
says so at the start and keeps it in view.

**The opening card.** A new colony opens on a card before the first step. It says that the year
is forty days in four seasons, that winter brings no finds and costs more to live through, and
that what counts is the number of workers alive as spring returns. Interact, Mark or Pause
closes it. It appears for a new colony only — not when a save is resumed, and never over a
message about a save that cannot be loaded.

**The winter gauge.** The colony panel in the HUD carries one bar: the food in the stores
against what winter will need for the colony as it stands now (a full winter's upkeep for
today's workers, less what the thatch saves). The need grows as the colony grows, so the bar can
fall while the stores rise. A tick on the bar marks what the store chambers can hold; while the
tick sits short of the need, the stores cannot hold enough and the bar says so — another store
chamber is the answer. In winter the bar shows instead how many days the stores will last
against the days of winter left. The warning three days before winter points at the gauge.

**Season goals.** Each season has goals, shown one at a time in a line under the hints: the
season, how many of its goals are done, the first one not yet done, and one line on why it
matters. They are guidance, not gates. Nothing is locked behind them and nothing is awarded for
them; a goal already met when a colony is loaded counts as done without a fanfare. Completing a
goal, and completing a season's goals, each raises a short toast. While the first-run hints are
still showing, they come first.

**Assumed:** a season's goals only count from that season's first day — one already met
earlier is marked done quietly when its season begins, and while the current season's goals are
all done the line shows the next season's first goal dimmed as upcoming (cheap to change: one rule
in the goals tracker).

| Season | Goal | Why |
|---|---|---|
| Spring | Haul the sugar cube | Sugar is the first store. |
| Spring | Dig a second brood chamber | The queen lays only with room. |
| Spring | Reach 25 workers | More workers lift bigger finds. |
| Summer | Haul a dead insect | The biggest food in the garden. |
| Summer | Dig a second store chamber | Stores decide winter. |
| Summer | Store half of winter's need | The gauge, halfway. |
| Autumn | Store all of winter's need | What you hold is what survives. |
| Autumn | Thatch the nest with 4 pinecones | Each one cuts winter's cost. |
| Autumn | Drive off a spider | Raids scale with the rival and your tier, not your size. |
| Winter | Keep everyone alive: the stores never run dry | — |

Until the colony is established, the dead-insect goal reads "become established" instead; the same
holds for pinecones until it is mature.

**Assumed** — this goal list, its order, and "half of winter's need" as the summer mark. Cheap to
change: the goals are read from the colony's counts, not stored, so editing them breaks no save.

**What the nest can do.** The nest panel says, under the colony's tier, what that tier opens:
a Young colony takes seeds, leaves and sugar and has 4 slots to dig; an Established one adds dead
insects and 7 slots; a Mature one adds pinecones and 10 slots.

The economy stays gentle: a colony's hauls comfortably cover its upkeep. The pressure comes from
winter and from what the nest can hold, not from scarcity.

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

Finds appear over time at random places around the nest, more often near it than far: half of
the seeds and leaves lie within about 33 u of the nest, about 37 u on average, and none farther
than 90 u. Up to ten seeds and eight leaves lie in the garden at once, with at most one sugar
cube, two dead insects and two pinecones. Each lies there a while before it is gone: seeds for
two days, leaves for a day and a half, a sugar cube for three, a pinecone for four. A find you
have marked stays until it is hauled. There are no scout ants: every find is found and marked
by you. **Assumed** — the spawn rates and lifetimes, and no scouts. Cheap to change, though
scouts would need the economy retuned so that they do not replace the player. **Assumed** — the
near-biased spread and the limits of ten seeds and eight leaves. Cheap to change: one rule and
data.

One dead insect lies in the garden from the start, always within 40 u of the nest (25 to 40 u),
inside the range you sense finds at, so a Young colony can walk up to the biggest food there is
and read what it needs: an Established colony. It rots in three days, long before a colony can
take it. **Assumed** — the starting insect and its 40 u limit. Cheap to change: data.

**Antennae.** You sense unclaimed finds within about 45 u, the six nearest at most. One in view
carries a small label with its name and distance in body lengths; one off screen shows as an
amber chevron at the edge of the screen, pointing towards it. Finds the colony cannot take yet
are shown dimmed. The chevrons hide while you are laying a trail. When a find appears within
about 40 u, a toast says you have caught its scent, at most once a minute. The grass parts
around a find so that it can be seen from a few steps away. **Assumed** — the 45 u range, six
markers, the 40 u scent range and once-a-minute limit. Cheap to change: presentation constants,
nothing in the simulation or the save.

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
loop across it, never closer than a short walk from the nest; when that trail goes quiet it
moves to whichever trail the colony is using next. Finds right by the nest are safe from them. A
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

The player ant dies to threats and comes back: within a spider's reach or a raid's path for
about three seconds, or under a bird's strike, you are lost, and ten seconds later you leave the
nest as a fresh worker, the colony one worker smaller. Nothing else is lost, and the queen cannot
die. The colony is the thing that can be lost; you are one of its ants.

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

## Sound

The garden is heard as much as seen: a meadow bed by day, crickets by night, rain when it
rains, with a faint bird layer in clear daylight. Ants near you skitter; a spider's strike
lands with a thud; the bird's shadow arrives on a wing-beat; raiders chitter as they march.
Short stings mark a delivery, the year's end and the colony's death. The nest panel and the
cards duck everything underneath them. All sound is CC0 from freesound, Kenney and
OpenGameArt and is credited in the attribution file that ships with the build. **Assumed** —
the mix levels and which events get a cue; cheap to change in the Audio Bank asset.

## Controls

Keyboard + mouse: WASD move, mouse look, E interact, F mark trail, Q tap an ant, Shift
sprint, Esc pause (a second Esc opens credits and sound). Gamepad: left stick move, right
stick look, A interact, X mark, Y tap, left trigger sprint, Start pause. Touch (phones and
tablets, landscape only): a joystick under the left thumb moves, dragging on the right half of
the screen looks, round buttons for Mark, Tap, Interact and Run sit in the bottom-right corner,
and Menu in the top-right corner opens credits and sound.

**Assumed** — touch is landscape only; held upright, the page asks you to turn the device
sideways and the game runs on underneath. Cost to change: a portrait HUD and button layout,
which the phone-sized HUD does not have room for today. **Assumed** — Run is a latch on touch:
tap it to run, tap again to walk, and it switches itself off shortly after you let go of the
joystick. Cheap to change.

---

## Settled

These were open questions; they were decided on 2026-10-06 and are now design.

- **Tapping in the field redirects.** Away from the nest, Tap takes up to three workers within
  a few body lengths off whatever they were doing and sends them down your current trail. It is
  the only way to steal from one job for another, and it costs the job they left.
- **Embodied only.** Recruiting is trails and taps. The nest panel orders digging and nothing
  else. There is no map, no cursor and no order queue.
- **A fictional colony.** No real species is named. Behaviours are plausible, not documentary,
  and threats are "a spider", "a bird", "the rival colony".
- **No climbing.** Ground and gentle slopes, for good. Stems and walls are scenery.
- **Licence.** The play repository's design documents are CC BY 4.0. The game build is free to
  play where it is published and may not be redistributed. The source stays private.

## Undecided

Nothing at the moment. Run `/open-questions` to confirm.

## Assumed (summary)

Listed inline above or in [SIM.md](SIM.md): the game starts at dawn; 1 u = 1 cm; 10 Hz tick, 2D on the ground plane, sim pauses when the
tab is hidden; float determinism on one target; NPC ants are counts; trail-graph pheromones
rather than a diffusion grid; player counts as one ant; six-minute day, ten-day season;
Input System, Cinemachine 3, UI Toolkit; WebGL2 with gzip + decompression fallback; touch is
landscape only, with a latching Run button; versioned JSON saves; commercial-safe asset licences only. From
[SIM_M2.md](SIM_M2.md): nursing need and taps that take nurses; queen laying, brood and
starvation rates; chamber kinds, costs and slots; tier chamber requirements; find spawning and
lifetimes; pinecones as shelter; winter costs more; outcome = workers alive as spring begins;
no scout ants; the panel orders digging only; near-biased find spawning, ten seeds and eight
leaves at once, and one dead insect at the start. From the Purpose section: the season goals and
the summer half-way mark; the antennae range, markers and scent toast. From [SIM_M3.md](SIM_M3.md): the player ant
respawns and the queen cannot die; the weather and threat numbers; spiders defended by marking;
the alarm on Interact; automatic nest defence; re-walking a trail renews it; a separate random
stream for weather and threats; colonies saved before M3 continue with fresh weather and threats. From
[NEST_VIEW.md](NEST_VIEW.md): the living cutaway as a 3D diorama in the panel; capped ant bodies;
processing shown as presentation only (food is stored on delivery); no nest-view state in the save.
