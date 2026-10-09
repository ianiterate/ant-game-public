# Nest view — the living cutaway

Design for animating the nest panel so the colony's state reads at a glance: the queen laying,
brood growing, nurses tending, stores filling, diggers digging. Presentation only. It reads the
simulation and changes nothing in it: no Sim changes, no save changes. [GAME.md](GAME.md) "The
nest" is the game; [SIM_M2.md](SIM_M2.md) §1–§4, §12 and [SIM_M3.md](SIM_M3.md) §7 are the rules
this view draws. [NEST_VIEW_THEME.md](NEST_VIEW_THEME.md) is what it looks like (the cut face,
palette, motion verbs); where the two meet, the theme decides the look and this file decides
which count or event drives it. Run `/open-questions` for everything marked below.

Units: the diorama has its own scale (§1.3). Reference pixels are the 1600×900 reference of
`HudPanelSettings`, as in `NestPanel.uss`.

---

## 1. Rendering approach

**A 3D diorama rendered to a RenderTexture inside the existing panel** (option (a)): a shallow
side-on nest — soil, shaft, tunnels, the queen's chamber and ten chamber pockets — built from a
handful of meshes, populated with real `Ant_Worker` bodies and a queen, lit by its own simple
shader, seen by one orthographic camera that renders only while the panel is open. The RT is the
background of `#nest-cutaway`; the slot cards, labels and touch targets stay UI Toolkit elements
on top of it.

**Assumed** — option (a). Not reviewed by you. Cost to change: high once the queen, brood and
clip assets exist; nothing before that.

Why this one:

- **It reuses what the game already looks like.** The worker mesh, its Idle/Walk rig, the chitin
  material work and the find models (dead insect, seed, sugar) all carry over. The nest is then
  the same world, closer — which is what "thematic" asks for. Option (b) would need a second art
  pipeline (sprites or vector art) that the project does not have, and a second look to match.
- **The RT keeps its last frame.** The diorama can render at 30 Hz or 15 Hz on a phone and the UI
  still shows a steady picture. A camera drawing straight to the screen through a hole in the panel
  cannot skip frames on WebGL, because the browser does not keep the drawing buffer.
- **The panel stays a panel.** Opening at the entrance, direction-and-confirm navigation, tap
  support and the `touch-compact` layout all stay as they are. Option (c), cutting the real mound
  open, fights the third-person camera, needs an interior in the terrain, and keeps the full
  garden rendering under a view that covers 90% of the screen.

### 1.1 Setup

- **Where it lives.** A `NestDiorama` root far from the garden (y = −500), on its own layer,
  disabled while the panel is closed. The garden camera's culling mask excludes that layer. The
  diorama camera sees only that layer.
- **Camera.** Orthographic, looking along +z at a slab about 12 units deep. It uses a dedicated
  URP renderer asset with no renderer features: no shadows, no post-processing, no depth or opaque
  texture, depth priming off. MSAA is 2× on desktop and off on touch.
- **RenderTexture.** It has the cutaway element's aspect ratio and is at most 1024 × 512 (768 × 384
  on touch), with a 16-bit depth buffer. It is created once and recreated only when the element's
  size changes (`GeometryChangedEvent`). Shown with `Background.FromRenderTexture`. Check
  `IsCreated()` again on every open, because the browser can lose its graphics context.
- **Lighting.** A small `NestLit` shader. The light comes from the open cut and down the shaft,
  and falls off with depth (the theme's "no lamps, no glow"). It is half-Lambert from that key,
  with a depth falloff from the key colour to the theme's deep colour, plus the existing rim term.
  Four globals drive it: `_NestKeyColor` and `_NestExposure` (day and night, §2.3), and
  `_NestDesaturate` and `_NestStarve` (winter and hunger, §2.3). It ignores the sun and every scene
  light, so the garden's lighting cannot leak underground. No realtime shadows. Ants get the
  existing contact-shadow quad.
- **Layout follows the UI.** Chamber positions are not authored twice. On a geometry change, the
  diorama reads each slot card's and the queen card's `layout` rect inside `#nest-cutaway`, maps
  it through the ortho camera, and places that chamber's pocket there. A pocket may be scaled
  unevenly, but an ant never is. The tunnels are a fixed graph — slot → row gallery → trunk → shaft
  — whose points are worked out from those rects. They are built into one mesh on a layout change,
  never per frame, from the cut profile of the kit's tunnel segment: walls and the rim of the cut
  edge run only along the free edges of the network, so tunnels open cleanly into each other and
  into the pockets, and a vertical shaft has no floor up its side. So a slot moved in `NestPanel.uss`, in either the desktop or the
  `touch-compact` layout, moves its chamber with it.
- **A stable look per colony.** Decorative scatter (pebbles, rootlets, grain in the soil) comes
  from a `System.Random` seeded with `World.Seed`, so a colony's nest looks the same every time it
  is opened. Presentation randomness never touches the simulation's `Rng`.

**Assumed** — the layout is read from the UI's rects, not authored in a second asset. Cheap to
change. Its cost is that the tunnel graph has to be topological, not hand-drawn.

### 1.2 Budget

| Item | Desktop | Touch (landscape phone) |
|---|---|---|
| Ant bodies | ≤ 34 workers + ≤ 6 raiders (§2.2) + the queen | same |
| Extra cameras | 1, only while the panel is open | same |
| RT | ≤ 1024 × 512, 2× MSAA | ≤ 768 × 384, no MSAA, rendered every 2nd frame |
| Draw calls (SRP Batcher) | ≈ 60–75: soil 1, pockets ≤ 11, tunnels 1, ants ≤ 41, brood 3 instanced, piles ≤ 11, props ≤ 6 | same |
| CPU (estimate, to be measured) | ≤ 1.5 ms: 41 Animators ≈ 0.6, CPU skinning of ≈ 25k vertices ≈ 0.3, culling and submit ≈ 0.4, agents ≈ 0.2 | ≤ 2.5 ms, with resting ants' Animators stepped by hand at 15 Hz |
| GPU | ≈ 0.5 Mpx at about 2× overdraw: negligible beside the garden | ≈ 0.3 Mpx |
| Memory | RT 2 MB colour + 1 MB depth (+2 MB MSAA); meshes and clips < 1 MB | ≈ 1.2 MB |
| Per-frame allocation | none: pools, matrices and agent arrays are built when the scene loads, not on the first open | same |
| Closed panel | zero: the root is disabled, the camera off, the Animators off. Only the event recorder runs (§4.2). | same |

The garden camera keeps rendering under the panel; you never lose the third-person view. If a
phone misses its frame budget, there is a relief valve. While the panel is open, the garden
camera renders once into a low-resolution still that becomes the panel backdrop, and is then
switched off until the panel closes. **Assumed** — off by default, because a frozen strip of
garden at the top of the screen is a visible cost. Cheap to turn on: one flag.

**Assumed** — animated skinned bodies (one Animator each), not vertex-animation textures.
Measure first. If profiling shows more than the budget above, baking the clips to a vertex
animation texture and drawing all ants in one instanced call is a contained change: about two
days of shader and Blender work, nothing else touched.

### 1.3 Scale and readability

The cutaway is about 930 × 560 reference px on desktop. A slot is then about 158 × 146 px and the
queen's chamber about 205 × 465 px. On a landscape phone (`touch-compact`) the cutaway is about
730 × 230 px and a slot about 139 × 60 px.

The diorama's scale is set so that a worker is 32 px long on desktop and 0.8 of that (about
26 px) on compact: `workerPixels` in `NestViewSettings`. The queen is twice a worker's length (the
theme). Brood items are 15–23 px (brood drawn at 1.8×). A compact chamber keeps its caption in its top 16 px, so its
contents get about 44 px of height: room for one ant standing, a row of brood and a pile up to
30 px tall. That reads as motion and silhouette, not as detail. The numbers stay in the captions.

The camera looks slightly down into the cut, by `cameraTilt` (18°) on `NestDiorama`, so chamber
floors and the ants' backs show instead of a thin side profile. The layout still follows the
cards: each card's rect is projected along the tilted view onto the plane the ants stand in, so a
room is drawn under its card whatever the tilt (it is taller in the world by 1 / cos 18°, about 5%).
Seen from above, a cavity's roof is drawn as the cut face, not as a window into the cavity.

**Assumed** — a worker 32 px long on desktop, up from 20, because ants read thin in profile at
20 px and still too small at 26. A slot now holds about five worker lengths instead of eight, so
fewer brood rows fit and a full room fills sooner. Cheap: one value in `NestViewSettings`.

The gutters between the queen's column and the slot columns are narrow (about 14 px on desktop),
so the two trunks run down the middle of each gutter, as wide as it allows less a sliver of soil
either side (at most a full tunnel). At 32 px that is a crack about a third of a worker wide:
workers climbing it overlap the soil beside it.

**Assumed** — trunks narrowed to fit the gutters rather than drawn across the cards' edges. Cheap:
one rule in `NestGeometry`; widening the gutters in `NestPanel.uss` widens the trunks with them.

A strip of the surface shows above the cut: the ground line sits one unit (32 px) below the frame's
top (`groundDrop` on `NestDiorama`), so the shaft's mouth and the spoil heap of an active dig are in
the picture. The heap grows with the pellets carried up during a dig and starts again with the
next one.

**Assumed** — the visible surface strip and spoil heap. The theme had pellets dropped out of the top
of the frame. Cheap: `groundDrop` 0 puts the ground back on the frame's edge.

**Assumed** — the 18° downward tilt, for the same reason. Cheap: one value on `NestDiorama`, 0
returns to a straight side-on cut. Past about 30° the cut stops reading as a cross-section and the
rims of the pockets start to hide their floors.

**Assumed** — ants are drawn 1.25 times larger, relative to their chamber, on compact than on
desktop (compact is 0.8 of the desktop size, while its chambers are much smaller). Cheap: one
value.

### 1.4 Assets needed (for `blender-artist`)

| Asset | Notes |
|---|---|
| `Ant_Queen` | ≤ 1 800 triangles, ≤ 24 bones (a gaster bone that can swell). Clips: `Rest` (slow breathing), `Lay` (gaster contraction, about 1.2 s), `Still` (winter: legs tucked). |
| Worker clips on the existing `Ant_Worker` rig | `Dig` (forelegs scrape, head down, loop), `Carry` (Walk with mandibles raised; the carried prop parents to a head socket), `Tend` (head down, antennae tapping, loop), `Rest` (legs folded, slow antennae), `Huddle` (tucked, nearly still), `Spar` (raised front, lunge; raids). |
| Brood | `Egg` (≈ 40 tris), `Larva` (≈ 80 tris, curled), `Pupa` (≈ 80 tris, cocoon), and a split-cocoon variant for hatching. |
| Store pile | One heap mesh with a height-mask shader (fill 0–1), plus scattered crumbs for spills. |
| Soil and pockets | A soil backdrop with a cross-section texture, a chamber pocket mesh (rim + floor), a tunnel cross-section profile, and a spoil-heap mesh for the surface. |
| Props | Dirt pellet; the dead insect and seed, reusing `Assets/Art/Finds` at diorama scale; seed husks; a pinecone scale (thatch). |

Raiders reuse `Ant_Worker` with `Ant_Rival.mat`, as `RaiderViewPool` does. The player's ant, when it
is drawn among the defenders, uses the player's cool-rim material.

---

## 2. What is visible at a glance

### 2.1 Places

Every ant body stands at a **post**: a named point in a chamber, a gallery or the shaft. A body
walks between posts along the tunnel graph. It never teleports, except when the panel opens.
Posts:

- **Shaft**: the vertical entrance from its mouth at the ground line down to the landing; the
  trunks either side of the queen's column carry on down to her chamber.
- **Landing**: the wide gallery just below the entrance, in the upper part of the queen's column,
  where idle workers wait in a loose crowd, two rows deep. This is where "workers free" is read:
  an empty landing means nobody is free.
- **Queen's chamber**: the foot of the queen's column, a wide low room she lies along (at most 55%
  of the column, and on screen no taller than 0.8 of its width), with
  her attendants, the base brood (the first 16 places) and base larder (the first 200 food). They
  sit in the same order the panel already uses to split totals: base first, then chambers in slot
  order (SIM_M2.md §12.2).
- **Slot pockets**: dug rooms only (brood, store, processing) and the active dig, whose stub grows
  out from the trunk. Undug and locked slots are solid soil: nothing is drawn for them, and their
  cards alone mark them.
- **Midden**: a dead-end pocket off the shaft near the surface, for husks, shell plates and the
  dead (§2.3).

### 2.2 Who is drawn — the body allocation

The colony is counts; the diorama shows at most 34 workers. One body is one ant, up to the cap
for its role. Beyond the cap the role is shown as full, and the caption gives the number. Roles are
filled in this priority each snapshot:

| Role | Shown | Cap |
|---|---|---|
| Diggers | `Crew` of the build task | 4 |
| Queen's attendants | `min(Nursing, QueenNurses + 2)` | 3 |
| Nurses | `ceil(nurses in a chamber / 2)` per brood-holding chamber; nurses are spread by that chamber's share of the brood | 8 |
| Processing | 3 while a carcass is being cut (§2.3); otherwise 2 scraping the floor, from `Idle` | 3 |
| Shaft traffic | one body per `AntLeftNest` / `AntReturned` / delivery carrier, played over 3 s | 4 |
| Idle, waiting on the landing | `Idle` minus the processing bodies drawn from idle | 12 |
| Raiders (raids only) | `min(Raiders, 6)` | 6 |

To keep it stable: targets are recomputed at 10 Hz, but at most two bodies change role per second.
A body joining walks in from the shaft or the landing. A body leaving walks out. Deaths are the
exception (§2.3).

No body stands anywhere just to look busy. Each one is a count or an event (the theme's first
rule). That is why there are no store keepers: workers appear at a pile only when a delivery
comes in.

**Assumed** — this allocation and its caps. Cheap to change: data in a `NestViewSettings`
ScriptableObject.

### 2.3 State → visual

"Idle anim" is what plays when nothing is changing. "Active" is the moment a change happens.

| Subject | Trigger (snapshot fields, §4) | Visual | Idle anim | Active | On a phone |
|---|---|---|---|---|---|
| **Queen laying** | `EggsPerDay`, `EggAccum` | The queen lies in the centre chamber, with attendants at her head and flank. Her gaster swells with `EggAccum` (0 → 1), so the next egg is visibly coming. | `Rest`, breathing period `clamp(4 s · 5 / EggsPerDay, 3, 12)`; attendants groom | On an `EggsLaid` delta: `Lay` (the gaster heaves), an egg appears at her side, and an attendant carries it up the tunnel to the chamber whose egg count rose (≈ 3 s). At full rate this happens every 72 s. | The queen is 32–40 px long: the swell and the egg read; the breathing is a bonus. |
| **Queen blocked** | `QueenBlocked` | She and her attendants are still, the gaster swollen and held, and every brood chamber is packed to its doorway. The eye goes from her to the crowded doors. | Still | On `BroodCapacityReached`: the attendants settle and stop grooming. | Reads as "she's stuck"; the card says "full". |
| **Queen, not laying** | `EggsPerDay == 0` (winter, or food = 0) | Gaster slack, still. The brood doors are open and the stores bare (no food), or the colony is packed around her (winter). | Near-still | — | Told apart by frost (winter) or bare floors (hunger). |
| **Queen dead** | — | Not drawn. The queen cannot die (SIM_M3.md §8), and a dead colony opens the death screen, not the panel. | — | — | — |
| **Brood stages** | `BroodHatchIn[i]`, split per chamber | For `M = 7`: indices 6–5 are eggs (an ivory clump), 4–2 larvae (curled cream grubs), 1–0 pupae (straw cocoons in a row nearest the door). Each chamber draws its share. | Nurses turn and feed (see Nursing); the brood itself is still | At each day start, cocoons shift a body length towards the door as their stage moves on (0.5 s). | Three colours, three sizes. A full chamber is packed wall to wall. |
| **Hatching** | `BroodHatched` (A = n), `WorkersRaised` delta | At day start (midnight), up to 3 cocoons split open. A pale worker stands, darkens to the colony's brown over 3 s, and walks out to the landing. Shown hatches are capped at 3; the other cocoons are simply gone in the next snapshot. | — | ≈ 5 s per new worker | One clear moment a day. |
| **Nursing** | `Nursing`, `NurseDemand` | Nurse bodies among the brood: lift a larva, turn it, set it down, bend to feed one; a new spot every 6–10 s. | `Tend` | `NursesPulled`: one nurse per brood chamber walks out to the shaft. | Few nurses among a lot of brood reads as short; the card's warning remains. |
| **Under-nursed** | `NurseCoverage01 < 1` | Clusters with no nurse near them lie untouched and dull: desaturated by `1 − coverage`. | — | On a `BroodLost` delta (cause 1): a nurse carries a still larva up to the midden. | Dull brood beside cream brood. |
| **Stores** | `Food`, `FoodCapacity`, split per store | One pile per store pocket, and a few grains in the queen's chamber for the base room, filled by each chamber's share (base first). The top layer shows the kinds most recently delivered (§4.2): sugar grains, seed pebbles, leaf squares, insect pieces. Height smoothed with τ = 1 s. | — | `ItemDelivered`: carriers come down the shaft, tip their load on the first pile with room, and the top slumps. | Pile height is the biggest signal on the screen; it reads at 60 px. |
| **Stores full / lost** | `Fill01 ≥ 0.98`; `StoresFull` (A = lost) | Full: the pile reaches the doorway, and carriers stand in the tunnel holding loads. Lost: grains proportional to `A` (≤ 20 instanced pieces) spill out of the door and down the tunnel floor, where they are left, fading over one game day. | — | 2 s spill | The spill is the moment; the toast says how much. |
| **Processing** | `ProcessingJobs` (§4.2) | The cutting room (settled below). When a dead insect is delivered it is dragged in whole, on its back, and cut at the joints over 90 s in four steps: legs, head, shell pried off, soft parts portioned. Up to 3 workers; pieces are carried to the stores, and shell plates go to the midden. Between insects the room is swept and empty, with two workers scraping the floor if any are idle. | Scraping | Carcass dragged in; each cut step; pieces carried out | A carcass on its back in the chamber is unmistakable. |
| **Digging** | Build `Slot`, `Kind`, `Progress01`, `Crew`, `SecondsLeft` | A rough stub with a crumbling face, its floor darker and damper than finished rooms. The cavity grows outward from the tunnel mouth: a shader clips the pocket beyond radius `Progress01`. The crew scrape at the face, roll pellets, climb the shaft and drop them over the rim. | `Dig` loop, a pellet trip every 5–8 s | `BuildStarted`: the crew walk in from the landing. `ChamberBuilt`: a dust fall, the crew return to the landing, the room gets its kind's dressing. `BuildCancelled`: the crew walk out and the stub crumbles shut over 1 s. | A growing hole with four ants in it. `Crew == 0`: the face is still and a pellet lies where it was dropped — "waiting for workers". |
| **Empty / locked slots** | `UnlockedSlots`, `SlotUnlockTier` | Solid soil, nothing drawn: the card alone marks them (a faint hairline and "Room to dig" when unlocked, a faint caption when locked). | — | — | Only dug rooms and the dig are holes. |
| **Tier** | `Tier`; `TierChanged` | Shown as the nest's extent: the slots a tier opens change from dense to ordinary soil, and the mound at the top of the frame is a crater rim (Young), a low mound (Established) or a tall mound (Mature). | — | `TierChanged`: soil falls and the newly opened slots lighten over 2 s (darken on a drop). | The mound height reads; the tier label stays. |
| **Starving** | `Starving` (Food = 0 and upkeep > 0); `WorkersStarved` delta | `_NestStarve` (0 → 1 over 3 s): the section desaturates 30%, workers' gasters narrow (bone scale 0.85), and they move at 0.6× and stall mid-step. The store floors are bare. | Faltering | On a `WorkersStarved` delta: a landing body is carried up to the midden. | A thin, grey, sparse nest. |
| **Low food** | `FoodDaysLeft < 2` | Low piles (already true), and the queen's breathing slows with her rate. | — | — | Carried by the piles. |
| **Shelter** | `Shelter`, `ShelterMax` | Pinecone scales layered over the entrance, one per shelter point. In winter the frost line stops at them (see Winter). | — | `ItemDelivered` of a pinecone: a scale drops into place. | Visible all year. |
| **Raid alarm** | `Raid.Phase == Marching` | The landing packs tight below the entrance, every head towards the top. Nurses stay with the brood. | Agitated | `ThreatSpawned` (raid): bodies hurry at 1.5× | Everyone at the top of the frame. |
| **Raid fight** | `Raid.Phase == Fighting`, `Raiders`, `InNest`, `PlayerAtNest` | Defenders jam the shaft mouth, grappling `min(Raiders, 6)` red-brown raiders in the entrance. While `PlayerAtNest`, one defender is your ant, picked out by its cool rim. | `Spar` | On a raid `AntsLost` delta: a defender goes down (carried out later). Each `RaidersLost` step: a raider body backs out. | One scrum at the entrance; the HUD gives the counts. |
| **Breach** | `Raid.Phase == Looting` | Raiders pour down the shaft to the store nearest it and carry pieces back up and out. The piles drop with `Food`. Nurses close over the brood; the queen and brood are never touched. | Looting trips | `RaidBreached`: they pour in. `RaidLootingEnded` / `RaidRepulsed`: raiders leave up the shaft, the dead are dragged to the midden. | Red-brown ants inside the stores. |
| **Midden** | Deaths and brood losses in the last 2 game days; shell plates from processing | A heap of husks, plates and curled bodies, sized by recent losses, decaying over 2 game days. | — | Carry-outs (above) | A growing heap says "we are losing ants". |
| **Winter** | `Season == Winter`; `Shelter` | The section desaturates 25%. A pale frost line creeps down the cut face from the surface; it reaches `1 − Shelter / ShelterMax` of the way to the queen's chamber, and at full thatch stops at the scales. The landing empties: everyone packs into the queen's chamber around her. Animation at 0.6×. | `Huddle`, a slow shift at the edges | `SeasonChanged` to winter: frost creeps down over 5 s, and the bodies walk to the queen. | The cold palette, the frost depth and the dark mass read at once. |
| **Night** | `TimeOfDay01`, `IsNight` | By day a warm wedge of light comes down the shaft and the upper rooms sit in it. At night the wedge turns low and blue-grey and the section drops a stop (`_NestKeyColor`, `_NestExposure`, from the theme's day and night keys, on the solar curve `DayNightLighting` uses). The work goes on; the landing settles. | Landing ants fold their legs at night and shuffle by day | — | The light in the shaft is the clock. |
| **Rain** | `Raining` | The shaft mouth is dark and wet, with drips running down. The recall shows as shaft traffic coming in (`AntReturned`). | Drips | `WeatherChanged` to rain: traffic coming in | A wet shaft mouth. |
| **Ants outside** | `AntsOutside`; `AntLeftNest`, `AntReturned` | Shaft traffic: bodies climbing out and coming in. The only sign of the trails underground. | — | per event | Movement in the shaft means work is going on. |

**Assumed — what processing means** (with [NEST_VIEW_THEME.md](NEST_VIEW_THEME.md)). The
processing chamber is the cutting room: a dead insect is too big to store whole and rots where it
lies, so it is taken apart there into pieces that keep. That is why it gates dead insects and
nothing else; seeds, leaves and sugar come in store-sized. In the simulation the food is stored the
moment it is delivered, as today, and the chamber is only a requirement for tier 2. The cutting is
shown after the fact and delays nothing. Cost to change: if you want processing to be a real
throughput step (an insect stored raw and turned into food over time), that is a Sim change. It
needs a raw-food count, a cutting rate per chamber, a save version bump, and the insect economy in
SIM_M2.md §14 redone: about two days. The view would then read the real backlog instead of the
event recorder, a small change on the view side.

**Assumed** — the midden, the mound by tier, the night rhythm of the landing, and frost depth as the
shelter readout. Pure presentation; none is a rule of the colony. Cheap to change.

**Assumed** — the brood stage boundaries (2 days egg, 3 larva, 2 pupa, for `M = 7`). If
`BroodMaturationDays` changes they scale with it: eggs are the first 2/7 of the pipeline, pupae
the last 2/7. Cheap.

---

## 3. Interaction and how the view sits with the UI

Interaction does not change. Move left/right walks `NavOrder`. Interact opens the build options,
digs, or asks to stop a dig. Mark or Pause closes the panel. On touch, a tap selects and a second
tap confirms. The diorama element is `picking-mode="Ignore"`, so every tap still lands on a card.

What changes in the UI (for `scene-builder`; layout only):

- **`#nest-cutaway`**: its flat soil colour, `.nest-ground` and `.nest-shaft` are replaced by the
  RT. The RT element sits under `#queen-card` and `#slot-layer`.
- **Slot cards**: transparent bodies. The title and one status line sit on a dark scrim plate at
  the card's top edge (16 px on compact, about 34 px on desktop). The chamber art shows below it.
  Selection stays a UI border, crisp at any resolution. A digging slot keeps its progress bar as a
  thin strip under the caption, because the number matters. An empty slot is plain soil marked by a
  faint hairline and its "Room to dig" caption; a locked slot by a faint caption only.
- **Queen card**: the same treatment. Its plate keeps the name, eggs a day and brood, as compact
  does today. On desktop the plate also keeps the status line (full, winter, no food), and the
  brood-by-day bars and the nurse lines move under the queen's chamber. On compact they stay
  hidden, as today: the queen's pose and the brood itself now carry those states.
- **Opening the panel**: the diorama is built from the current snapshot with every body already at
  its post — no walk-in parade — and the event baselines are reset (§4.1), so nothing that
  happened while the panel was closed replays.
- **Captions are the truth.** Bodies are capped and smoothed. The numbers on the plates are not.

**Assumed** — the queen's chamber takes the foot of her column, at most 55% of it and no taller on
screen than 0.8 of its width (on desktop that is about a third of the column), with the landing
just above it and plain soil up to her caption plate. A tall pocket stretched to her whole card read
as a capsule, and idle workers in the thin band above it stood in one overlapping line; at 55% the
room was still taller than wide. Cheap: two constants in `NestGeometry`.

**Assumed** — empty and locked slots are solid soil, marked only by their cards: a faint hairline
and "Room to dig" for an empty slot, a faint caption for a locked one. Ten dark holes made dug and
undug rooms look alike. Cheap: a pocket per state in `NestDiorama`, the rest USS.

**Assumed** — the cards give up their full-height backgrounds for caption plates, and on desktop
the queen's long lines move below her chamber. Cheap to change: USS only. It is a trade of text
area for picture, which the request asks for.

---

## 4. Data contract

The diorama reads two things, both built in the Game layer: a **snapshot** of the world, rebuilt
when `World.Clock.Tick` changes (so at most 10 Hz, the same trigger `NestPanelPresenter` already
uses), and an **activity recorder** fed by `SimRunner.SimEventRaised`. Between snapshots,
continuous values are smoothed (τ 0.3–1 s) and progress values are extrapolated with
`SimRunner.Alpha`.

**No Sim changes.** Everything below is on the existing read API (SIM_M2.md §12.1, SIM_M3.md
§14.1) or is a public `ColonyState`/`RaidState` field. `World.Items[e.A].Kind` is still valid in
the event drain for `ItemDelivered`, because a delivered item's slot is freed only at the start of
the next `ItemSystem` step. Nothing is saved: the recorder starts empty on load.

### 4.1 `NestSnapshot` (one instance, arrays allocated once, filled in place)

| Field | Source | Notes |
|---|---|---|
| `Tick`, `Season`, `TimeOfDay01`, `IsNight` | `World.Clock` | |
| `Tier`, `UnlockedSlots`, `SlotUnlockTier[10]` | `Colony.Tier`, `UnlockedSlots`, `SlotUnlockTier(i)` | |
| `Chamber[10]`: `Kind`, `State`, `Locked` | `World.Chambers` | `Locked = i >= UnlockedSlots && !Active` |
| `Chamber[i].BroodCap`, `Eggs`, `Larvae`, `Pupae` | `BroodHatchIn`, `BroodCapacity`, `ChamberDef.BroodCapacity` | Split base first, then slot order. Stages are dealt youngest first, so eggs sit by the queen and pupae end up in the outer brood chambers (the theme's egg clump at her side). Uses the same `NestSplit` helper the presenter uses, moved there so both agree. |
| `Chamber[i].Food`, `FoodCap`, `Fill01` | `Colony.Food`, `FoodCapacity`, defs | Same split. Index −1 = the queen's chamber (base). |
| `BuildSlot` (−1 none), `BuildKind`, `Progress01`, `Crew`, `SecondsLeft` | `TryGetBuild`, `RequiredTicks` | `SecondsLeft = (Required − Progress) · dt / Crew`; ∞ at `Crew == 0` |
| `Idle`, `Nursing`, `NurseDemand`, `Digging`, `AntsOutside`, `InNest` | `Colony`, `NurseDemand`, `Digging`, `AntsOutside`, `InNest` | |
| `NurseCoverage01` | `Brood > 0 ? Nursing / NurseDemand : 1` | |
| `EggsPerDay`, `EggAccum`, `QueenBlocked` | `EggsPerDay`, `Colony.EggAccum`, `Colony.QueenBlocked` | |
| `Food`, `FoodCapacity`, `FoodDaysLeft`, `Starving` | `Colony.Food`, `FoodCapacity`, `FoodDaysLeft`; `Food <= 0 && UpkeepPerDay > 0` | |
| `Shelter`, `ShelterMax` | `Colony.Shelter`, `Config.ShelterMax` | |
| `Raining` | `Raining` | |
| `RaidPhase`, `Raiders`, `StartRaiders`, `PlayerAtNest` | `Raid`, `PlayerAtNest` | |
| Counter deltas: `EggsLaid`, `WorkersRaised`, `BroodLost`, `WorkersStarved`, `WorkersKilled` | `World.Stats`, minus the previous snapshot's | The baseline is reset on open, so deltas never replay what happened while the panel was closed. Monotonic counters survive a missed tick, where per-tick flags would not. |

`QueenNurses`, `BroodMaturationDays` and `ShelterMax` are read once from `World.Config` on open
and when the world is replaced.

### 4.2 `NestActivity` (always on, fixed ring buffers, no allocation)

It subscribes to `SimRunner.SimEventRaised` for the life of the scene, even with the panel closed,
because the player usually opens the panel just after a haul arrives. It keeps:

| Kept | From | Used for |
|---|---|---|
| The last 8 deliveries: kind, food stored, tick | `ItemDelivered` (kind via `World.Items[A].Kind`) | Carriers coming in; the colour of the stores' top layer; pinecone thatch drops |
| Processing jobs (≤ 4): start tick | `ItemDelivered` of a dead insect (90 s each, one after another) | §2.3 processing |
| Last spill: amount, tick | `StoresFull` | The spill |
| Shaft traffic owed: out, in | `AntLeftNest`, `AntReturned` | Up to 4 bodies, played over 3 s; the rest dropped |
| Per-slot build moments | `BuildStarted`, `BuildCancelled`, `ChamberBuilt` | Crew arriving, the dust puff, refilling |
| Tier moment | `TierChanged` | Outlines fade in or out |
| Raid moments | `ThreatSpawned` (raid), `RaidContact`, `RaidBreached`, `RaidRepulsed`, `RaidLootingEnded`, `AntsLost` (cause raid) | §2.3 raid rows |
| Losses for the midden | `BroodLost`, `WorkersStarved`, `AntsLost` (cause raid), finished processing jobs, with ticks | The midden size over 2 game days |

Moments from before the panel opened are shown as their end state, not played. A processing
job still running when the panel opens is shown part-way through.

**Assumed** — the recorder does not survive a save and load: after loading, processing is idle,
the midden is empty and the top layer of the stores is a neutral mix. Cheap. Keeping these would
mean presentation fields in the save, which this design avoids on purpose.

---

## 5. Milestones

**NV1 — first shippable slice: queen, brood, stores, diggers.**
Infrastructure: the RT, the camera, the layer, the `NestLit` shader, the layout read from the
UI's rects, the soil, pockets, tunnels and shaft, the snapshot and recorder, body pools with
posts and walking. Rows from §2.3: queen laying, blocked and not laying; brood stages and
hatching; nursing; stores, full and spill; digging, with the empty and locked slots; idle ants on
the landing; shaft traffic; the day and night light (it is the shader's key, so it costs nothing
extra). UI: the caption-plate restyle. Assets: the queen, the worker clips
`Dig`, `Carry`, `Tend` and `Rest`, the brood, the pile, the soil kit and the dirt pellet.
Done when: the panel opens with a living nest on desktop and on a landscape phone, within the
budget in §1.2, with zero per-frame allocation in the profiler.

**NV2 — the colony's condition.** Processing (the carcass cut in steps), under-nursed and starving
looks, the midden, the tier mound, shelter thatch, rain, winter huddle and frost. Assets: `Huddle`,
the cut-carcass stages, shell plates and husks, the pinecone scale, and the queen's `Still`.

**NV3 — raids.** Alarm, fight, breach and looting, repulse, the dead carried out. Assets: `Spar`.

**Assumed** — this split. Cheap to change. NV1 is the part the request names. NV2 and NV3 are
additive rows on the same machinery.

---

## 6. Failure modes

- **Reading bodies as counts.** The landing shows 12 ants when 140 are idle. Mitigation: the
  captions carry the numbers, and a full landing means "plenty". A player who counts ants will be
  wrong. That is accepted.
- **Lag that looks like a rule.** Heaps smooth over 1 s, egg carries take 3 s, and the carcass sits
  for 90 s. Processing in particular looks like a delay that does not exist (§2.3 Assumed). If
  players read it as "my food is not in yet", shorten it or make processing real.
- **Layout drift.** Chamber positions come from the cards' rects, so a USS change cannot misplace
  a chamber. A tunnel graph that no longer fits a rearranged layout can still look wrong. Check
  screenshots of both layouts after any change to `NestPanel.uss`.
- **First-open hitch.** Building pools or compiling shader variants on the first open stalls a
  WebGL frame. Prewarm when the scene loads, and include `NestLit` in the preloaded shader
  variant collection.
- **Phone heat.** A second camera running continuously while the panel stays open. The touch
  frame skip (every 2nd frame) and the hand-stepped resting ants are the knobs, and the backdrop
  still is the relief valve.
- **Graphics context loss.** The RT content is gone after a context loss. Re-create it on open
  when `IsCreated()` is false.
- **Crowding.** Posts are fixed points per chamber, with no avoidance, so bodies never jostle or
  overlap. The cost is that a chamber looks arranged rather than busy. Small idle offsets (±0.2
  units) hide the grid.
