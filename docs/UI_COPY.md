# UI copy

Every string the player reads. The sections from *Names* to *First-run hints* are the M1
vertical slice; [M2 — colony building](#m2--colony-building) adds the nest, the year and saves, and
[M3 — weather and threats](#m3--weather-and-threats) adds weather, spiders, birds, raids and the
player ant's death, and [M5 — Purpose](#m5--purpose) adds the opening card, the winter bar,
the season goals and the antennae sense, [Touch](#touch) adds the on-screen buttons of
phones and tablets, and [Nest view](#nest-view) adds the labels of the animated cutaway, and [Processing](#processing) adds the cutting of dead insects, and [Years](#years) adds the run of years after the first. Code references the **key**; the
text here is the source for the string table. The rules behind each line are in
[GAME.md](GAME.md) and [SIM.md](SIM.md). This file only puts them into words.

## Voice

- Terse and plain, present tense, full stops, no exclamation marks.
- Let the scale do the wonder. Call things what a person would (sugar cube, grass, the
  step) and let the gap in size speak for itself. Never say "tiny" or "mighty", and no puns or ant jokes.
- The colony is "the colony", "the nest", "workers". The player is "you". Workers have no
  names or numbers.
- Claim no biology the simulation does not model. No species, no Latin, no chemistry.
  "Trail", not "pheromone".

## Format

- `{name}` is a parameter. `[Mark]`, `[Tap]`, `[Interact]` and `[Pause]` are key glyphs drawn from the
  active device. They match the action names in `AntControls.inputactions`.
- Max length is in English characters, and a glyph counts as 3. Layouts must allow translations to run 30% longer.
- Every string is a whole label or a whole sentence, never assembled from other strings.
  Numbers follow a label, a colon or a sign, so no string needs plural agreement.
- `[Interact]` appears in no M1 string. `PickUp` and `Deposit` are not implemented, so E/A
  does nothing in the slice.

## Names

| key | text | when shown | max |
|---|---|---|---|
| `patch.1.name` | The Patio Edge | Loading card, pause header | 24 |
| `player.name` | You | Wherever the player is referred to | 12 |
| `tier.1` / `.2` / `.3` | Young / Established / Mature | Colony tier, `Colony.Tier`. From M2 a tier needs chambers as well as workers ([SIM_M2.md](SIM_M2.md) §4) | 12 |

Patch naming rule: name the patch after the human-scale thing that dominates it, in the
words a gardener would use for that spot. Examples: The Patio Edge, Under the Bench, The Drain
Corner, The Back Step. No adjectives and no ant's-eye coinages.

**Assumed** — the first patch is "The Patio Edge". It explains a sugar cube in a garden,
because someone had tea outside. The scene should show a paving edge somewhere. If the art
has none, rename it. Cost to change: one string.

**Assumed** — the HUD and text call the player "you", not a worker number. NPC ants have no
identity (GAME.md), and a number would hint at one. Cost to change: cheap in M1, one string.
Death is now "respawn as a fresh worker" (assumed in GAME.md; SIM_M3.md §8), and a worker
number would be the natural way to show the new body. That means adding a number field and a format string.

**Assumed** — the tier names are Young, Established and Mature. Cost to change: three strings.
If tier comes to depend on chambers (M2), the names still read correctly.

## HUD

| key | text | when shown | max |
|---|---|---|---|
| `hud.workers` | Workers | Always; value `Population` | 12 |
| `hud.idle` | Idle | Always; value `Colony.Idle` | 12 |
| `hud.out` | On trails | Always; value `AntsOutside`. With Idle, Nursing, Digging and Cutting it sums to Workers | 12 |
| `hud.nursing` | Nursing | Always; value from `hud.nursing.value` (M2). Dimmed unless nurses are short | 12 |
| `hud.food` | Food | Always; value from `hud.food.value` (M2) | 12 |
| `hud.food.empty` | Empty | Replaces the food value while `Food == 0` | 12 |
| `hud.tier` | Colony | Always; value `tier.{Colony.Tier}` | 12 |
| `hud.day` | Day {day} | Always in year 1; `day = Clock.Day + 1`. From day 41, `hud.day.year` (M2) | 12 |
| `hud.party` | At the find: {here}/{need} | While the player's task is `Recruiting`. `here` is `AtTarget` plus 1 if the player is in reach | 26 |
| `time.dawn` … `time.night` | Dawn / Morning / Midday / Afternoon / Dusk / Night | Beside the day. Time of day 0.25 / 0.30 / 0.45 / 0.55 / 0.70 / 0.75 onward. Night runs 0.75–0.25 and matches `Clock.IsNight` | 12 |

**Assumed** — time of day shows as a word, not a clock. The bands are presentation only:
night uses the simulation's `IsNight` edges, and the daylight words are split by eye. Cost to change: cheap.

## Finds (world label over the targeted find)

| key | text | when shown | max |
|---|---|---|---|
| `item.sugar_cube` | Sugar cube | The interaction probe targets the find | 20 |
| `item.lift` | Takes {n} to lift | Under the name; `n = def.AntsRequired` | 20 |

## Interaction prompts

| key | text | when shown | max |
|---|---|---|---|
| `prompt.find.mark` | [Mark] Mark a trail home | Find is `Lying`, in reach, tier is high enough, no draft | 40 |
| `prompt.find.claimed` | Marked. Stay close: you count as one. | Find is `Claimed` and the player is in reach | 40 |
| `prompt.find.hauling` | On its way to the nest. | Find is `Hauling` | 40 |
| `prompt.find.tier.2` | [Mark] Needs an established colony | Find needs tier 2, colony below it. Glyph drawn disabled | 40 |
| `prompt.find.tier.3` | [Mark] Needs a mature colony | Find needs tier 3, colony below it. Glyph drawn disabled | 40 |
| `prompt.trail.laying` | Back to the nest. [Mark] to abandon | `Draft.Active`, anywhere | 40 |
| `prompt.nest.tap` | [Tap] Send {n} down the trail | At the nest with the player task `Recruiting`; `n = min(TapBatch, Idle, need − Assigned) > 0` and taps left | 40 |
| `prompt.nest.wait` | The party is called. Wait for it. | At the nest with the player task `Recruiting`, but a tap would send nobody or no taps are left | 40 |
| `prompt.nest.idle` | The nest. Mark a find to call workers. | At the nest, no player task `Recruiting` | 40 |

## Toasts — events

| key | text | when shown | max |
|---|---|---|---|
| `toast.trail.started` | Trail started. Walk it back to the nest. | `TrailMarkStarted` | 60 |
| `toast.trail.laid` | Trail laid. Workers will follow it out. | `TrailLaid` | 60 |
| `toast.item.lifted` | Lifted. The party carries it home. | `ItemPickedUp` | 60 |
| `toast.item.delivered` | Into the nest. Food +{food} | `ItemDelivered`; `food = B` | 60 |
| `toast.task.item_gone` | The find is gone. Its workers turn home. | `TaskAborted`, `ItemGone` | 60 |
| `toast.task.replaced` | Old trail dropped. Its workers turn home. | `TaskAborted`, `Replaced` | 60 |
| `toast.night` | Night. Out on the trails, everything slows. | `NightStarted` | 60 |
| `toast.dawn` | Dawn. Day {day} | No event. The Game layer raises it when time of day crosses `NightEnd01`. Year 1; then `toast.dawn.year` (M2) | 60 |
| `toast.trail.cancelled` | Trail abandoned. The find stays where it is. | `TrailAbandoned`, `Cancelled` | 60 |
| `toast.trail.too_long` | Trail lost. Too far from the nest. | `TrailAbandoned`, `TooLong` | 60 |
| `toast.trail.too_short` | Too close to the nest to need a trail. | `TrailAbandoned`, `TooShort` | 60 |
| `toast.trail.item_gone` | Trail lost. The find is gone. | `TrailAbandoned`, `ItemUnavailable` | 60 |
| `toast.trail.no_slot` | Too many trails. Let one fade first. | `TrailAbandoned`, `NoFreeSlot` | 60 |

`FoodRanOut`, `TierChanged` and `SeasonChanged` change meaning in M2, so their strings live in the
M2 section.

**Silent** events get sound and visuals, but no text. `DayStarted` fires at midnight, in the dark: the HUD day updates and `toast.dawn` greets the day. `AntLeftNest` and `AntReturned` fire six at a time, so a toast would be spam. `TaskCreated` and `TaskComplete` share a tick with `TrailLaid` and `ItemDelivered`. `TrailFaded`: the line fades on screen. `ItemSpoiled` is followed by `toast.task.item_gone` if the find was marked. `AntsLost`, `WeatherChanged` and `ThreatSpawned` are first raised in M3; their strings are in the [M3 section](#toasts--events-2).

## Toasts — command rejected

Keyed by `RejectReason`. Prompts head off most of these before the button is pressed.

| key | text | when shown | max |
|---|---|---|---|
| `reject.no_such_item` | Nothing here to mark. | `NoSuchItem` | 60 |
| `reject.item_not_available` | That find is already taken. | `ItemNotAvailable` | 60 |
| `reject.out_of_reach` | Too far. Get closer to mark it. | `OutOfReach` | 60 |
| `reject.tier_too_low` | The colony is too small for this. | `TierTooLow` | 60 |
| `reject.draft_active` | Finish the trail you are laying first. | `DraftActive` | 60 |
| `reject.no_draft` | No trail to finish. | `NoDraft` | 60 |
| `reject.not_at_nest` | You need to be at the nest. | `NotAtNest` | 60 |
| `reject.no_such_task` | No trail is calling for workers. | `NoSuchTask` | 60 |
| `reject.task_not_recruiting` | That haul is already under way. | `TaskNotRecruiting` | 60 |
| `reject.tap_limit` | No more will come at a tap. The trail does the rest. | `TapLimit` | 60 |
| `reject.nothing.idle` | No idle workers left in the nest. | `NothingToSend`, `Colony.Idle == 0` and `Colony.Nursing == 0`. With nurses left, `reject.nothing.nurses` (M2) | 60 |
| `reject.nothing.full` | Enough are on their way already. | `NothingToSend`, any other cause | 60 |
| `reject.not_implemented` | Not in this build. | `NotImplemented`. Development builds only; silent in release | 60 |

## First-run hints

One hint at a time, pinned until its end condition. M1 has no save, so every session counts as a first run.

| key | text | when shown | max |
|---|---|---|---|
| `hint.find` | Somewhere out on the patch lies a sugar cube. Find it. | From the start until the probe first targets the cube | 70 |
| `hint.mark` | Too heavy to move alone. Press [Mark], then walk home. | Probe on the cube, until `TrailMarkStarted`. Shown again after `TrailAbandoned` or `TaskAborted` | 70 |
| `hint.wait` | Workers follow your trail out. Wait for enough to lift it. | `TrailLaid` until `ItemPickedUp` | 70 |
| `hint.home` | They have it. Walk beside it home. | `ItemPickedUp` until `ItemDelivered` | 70 |

---

## M2 — colony building

The strings for the nest panel, the queen and brood, the year and its outcome, and saves. The
rules are in [GAME.md](GAME.md) ("The nest", "The session") and [SIM_M2.md](SIM_M2.md). Voice and
format are as above. Two additions:

- **Parameters are counted at their widest value** when checking max length. Counts (`{n}`,
  `{crew}`, `{pop}`, `{have}`, `{peak}`, …) are 3 digits. Food (`{food}`, `{cap}`, `{cost}`, `{need}`)
  is 4 digits. `{day}` and `{year}` are 2 digits, and `{d}` is days to one decimal place (`12.5`).
- **Days shown to the player are 1-based.** Every `{day}` is `Clock.Day + 1` within year 1, and
  `Clock.Day % YearDays + 1` after it. That includes event payloads such as `ColonyDied.A` and
  `Stats.PeakDay`.

These M1 keys change text in M2 and are listed here only: `hud.nursing.value`,
`toast.food.empty`, `toast.tier.up.2`, `toast.tier.up.3`, and `toast.season.*`. These keys move
here unchanged: `item.seed`, `item.leaf`, `item.dead_insect`, `item.pinecone`. `toast.tier.down` is
split into `toast.tier.down.1` and `.2`. `hud.out`, `hud.nursing`, `hud.food`, `hud.day`,
`toast.dawn`, `tier.*` and `reject.nothing.idle` stay in the M1 tables with updated "when
shown" notes.

### Names

| key | text | when shown | max |
|---|---|---|---|
| `nest.title` | The nest | Panel header, while the panel is open | 20 |
| `chamber.queen.name` | Queen's chamber | Panel, always drawn first. Not a slot | 20 |
| `chamber.brood.name` | Brood chamber | Panel slot, build option, tier needs | 20 |
| `chamber.store.name` | Store chamber | Same | 20 |
| `chamber.processing.name` | Processing chamber | Same | 20 |
| `season.spring` | Spring | HUD beside the day; days 1–10 of each year | 12 |
| `season.summer` | Summer | Days 11–20 | 12 |
| `season.autumn` | Autumn | Days 21–30 | 12 |
| `season.winter` | Winter | Days 31–40 | 12 |
| `shelter.name` | Thatch | Wherever `Colony.Shelter` is shown | 12 |

Chamber naming rule: the chamber is named after what it holds or does, plus "chamber". No
coinages. A fourth kind follows the same rule.

**Assumed** — the queen's base chamber is called "Queen's chamber". It names the one place the
queen is, without saying anything about her biology. Cost to change: one string.

**Assumed** — the player-facing word for `Shelter` is "Thatch", which follows GAME.md's "thatched
with pinecones". The fiction is that winter costs more because the nest lets the cold in, and
each pinecone hauled home closes some of it. Cost to change: the six strings that say "thatch".

### HUD additions

| key | text | when shown | max |
|---|---|---|---|
| `hud.nursing.value` | {have}/{need} | Value of the nursing row: `Nursing` / `NurseDemand`. Highlighted while `Nursing < NurseDemand`. **Changed from M1** | 14 |
| `hud.digging` | Digging | Always; value `Digging`. Dimmed at 0 | 12 |
| `hud.brood` | Brood | Always; value from `hud.brood.value` | 12 |
| `hud.brood.value` | {n}/{cap} | `Colony.Brood` / `BroodCapacity`. Highlighted while `QueenBlocked` | 14 |
| `hud.food.value` | {food}/{cap} | `floor(Food)` / `floor(FoodCapacity)`. `hud.food.empty` still replaces it at 0 | 14 |
| `hud.season` | {season} | Always, beside the day: `season.{Clock.Season}` | 12 |
| `hud.day.year` | Year {year}, day {day} | Replaces `hud.day` from day 41: `year = Clock.Day / YearDays + 1` | 18 |
| `hud.saved` | Saved | Small and fading, for 2 s after an autosave at `DayStarted`. Not shown for the tab-hide save | 12 |

### Nest panel — the queen's chamber and colony lines

The panel opens when the player is within `NestRadius` of the nest ([SIM_M2.md](SIM_M2.md) §12).

| key | text | when shown | max |
|---|---|---|---|
| `chamber.queen.desc` | The queen lays here. It holds a little brood and food. | Queen's chamber, under its name | 60 |
| `nest.queen.eggs` | Eggs a day: {d} | Queen's chamber, always: `EggsPerDay`, one decimal place | 24 |
| `nest.queen.slow` | Laying slows as food runs low. | `EggsPerDay` is above 0 but below the season's full rate, because the stores hold under `QueenFedDays` of upkeep | 40 |
| `nest.queen.hungry` | No food. The queen has stopped laying. | `Food == 0`, not winter | 40 |
| `nest.queen.autumn` | Autumn: she lays half as many. | Autumn, unless one of the lines above or below applies | 40 |
| `nest.queen.winter` | Winter. No eggs until spring. | Winter | 40 |
| `nest.queen.full` | Brood full. Dig a brood chamber. | `QueenBlocked`. Takes precedence over every other queen line | 40 |
| `nest.brood` | Brood: {n}/{cap} | Queen's chamber, always | 24 |
| `nest.brood.axis` | Days to hatch | Label under the brood-by-age bars (`BroodHatchIn`, one bar per day) | 20 |
| `nest.brood.hatch` | Hatching at midnight: {n} | `BroodHatchIn[0] > 0` | 32 |
| `nest.nurses` | Nurses: {have}/{need} | Queen's chamber, always | 24 |
| `nest.nurses.short` | Too few nurses. Brood is being lost. | `Nursing < NurseDemand`, drawn as a warning | 40 |
| `nest.food` | Food: {food}/{cap} | Colony line, always | 24 |
| `nest.food.days` | Days of food: {d} | Colony line, always: `FoodDaysLeft`, one decimal place | 24 |
| `nest.winter.need` | Food for winter: {food}/{need} | Colony line, autumn only: `floor(Food)` / `ceil(WinterFoodNeed)` | 32 |
| `nest.shelter` | Thatch: {n}/{max} | Colony line, always: `Shelter` / `ShelterMax` | 24 |
| `nest.shelter.hint` | Each pinecone hauled home makes winter cheaper. | Under `nest.shelter` while `Shelter < ShelterMax` | 50 |

### Nest panel — tier

| key | text | when shown | max |
|---|---|---|---|
| `nest.tier.next.2` | To become established | Heading of the next-tier list at tier 1 | 32 |
| `nest.tier.next.3` | To become mature | Heading of the next-tier list at tier 2 | 32 |
| `nest.tier.need.workers` | Workers: {have}/{need} | Next-tier list: `Population` / `TierThreshold(next)`. Ticked when met | 32 |
| `nest.tier.need.processing` | Processing chambers: {have}/{need} | Next-tier list, when `TierChambersNeeded(next).Processing > 0` | 40 |
| `nest.tier.need.brood` | Brood chambers: {have}/{need} | Next-tier list, when `.Brood > 0`. Counts built slot chambers only, not the queen's | 40 |
| `nest.tier.need.store` | Store chambers: {have}/{need} | Next-tier list, when `.Store > 0`. Same | 40 |
| `nest.tier.max` | Mature. Every slot is open. | Tier 3, in place of the list | 32 |
| `nest.tier.slipping` | Too few workers. Falls back below: {n} | Population is below the current tier's threshold but the tier is held (§4 margin). `n = TierThreshold(tier) − TierDownMargin` | 40 |

### Nest panel — slots

| key | text | when shown | max |
|---|---|---|---|
| `nest.slot.empty` | Room to dig | Empty unlocked slot, above its build options | 24 |
| `nest.slot.locked.2` | Opens when established | Locked slot, `SlotUnlockTier(slot) == 2` | 24 |
| `nest.slot.locked.3` | Opens when mature | Locked slot, `SlotUnlockTier(slot) == 3` | 24 |
| `nest.build.cost` | Food: {cost} | On each build option: `def.FoodCost` | 16 |
| `nest.build.time` | Days to dig: {d} | On each build option, with a full crew | 20 |
| `chamber.brood.desc` | The queen lays only while there is room for the brood. | Build option detail, when the option is selected | 60 |
| `chamber.store.desc` | Holds more food. What will not fit is lost at the door. | Same | 60 |
| `chamber.processing.desc` | Cuts dead insects into food. Needed to become established. | Same. **Changed in Processing** | 60 |
| `nest.why.busy` | One dig at a time | Build option disabled: `CanBuild == BuildInProgress` | 28 |
| `nest.why.food` | Not enough food | `CanBuild == NotEnoughFood` | 28 |
| `nest.why.idle` | No idle workers | `CanBuild == NothingToSend` | 28 |
| `nest.why.tasks` | Too much under way | `CanBuild == NoTaskSlot` | 28 |
| `nest.slot.digging` | Digging: {pct}% | Digging slot: `ProgressTicks / RequiredTicks`, floored | 20 |
| `nest.slot.crew` | Diggers: {crew}/{need} | Digging slot: `Crew` / `BuildCrew` | 20 |
| `nest.slot.days_left` | Days left: {d} | Digging slot, `Crew > 0`: `(RequiredTicks − ProgressTicks) · dt / Crew / DayLengthSeconds`, one decimal place, at least 0.1 | 20 |
| `nest.slot.waiting` | Waiting for idle workers | Digging slot, `Crew == 0`, in place of days left | 28 |
| `chamber.brood.effect` | Brood room +{n} | Built brood slot: `def.BroodCapacity`, with its fill drawn as a bar | 24 |
| `chamber.store.effect` | Food room +{n} | Built store slot: `def.FoodCapacity`, with its fill drawn as a bar | 24 |
| `chamber.processing.effect` | Cuts dead insects into food | Built processing slot with nothing to cut. While it cuts, the lines of [Processing](#processing) replace it. **Changed in Processing** | 32 |

Locked slots that already hold a chamber (after the tier drops) are drawn as built. The chamber
keeps working, so they need no extra line.

### Nest panel — prompts

| key | text | when shown | max |
|---|---|---|---|
| `nest.prompt.dig.brood` | [Interact] Dig a brood chamber | An empty slot and the brood option are selected, and `CanBuild == None` | 40 |
| `nest.prompt.dig.store` | [Interact] Dig a store chamber | Same, store | 40 |
| `nest.prompt.dig.processing` | [Interact] Dig a processing chamber | Same, processing | 40 |
| `nest.prompt.cancel` | [Interact] Stop digging | A digging slot is selected | 40 |
| `nest.prompt.cancel.confirm` | [Interact] again to stop. Food back, work lost. | After one press of `nest.prompt.cancel`, for 3 s. A second press sends `CancelBuild` | 48 |
| `prompt.nest.tap.nurses` | [Tap] Send {n}. Nurses leave the brood. | Replaces `prompt.nest.tap` when the tap would take nurses: `n > Idle` (§2.5) | 40 |
| `prompt.nest.open` | [Interact] Open the nest | HUD, second prompt line under the nest prompt: at the nest, the panel closed, the colony alive, no card up. A dead colony shows `outcome.new_colony` instead | 40 |
| `nest.prompt.choose` | [Interact] Choose what to dig | Panel open, an empty unlocked slot is selected. Interact shows the build options | 40 |
| `nest.prompt.close` | [Mark] To the garden | Panel open, the build options not showing. Mark (or Pause) closes the panel | 40 |
| `nest.prompt.back` | [Mark] To the slots | The build options are showing. Mark hides them and returns to slot selection | 40 |

The two back-out lines name where Mark takes you: from the options to the slots, from the slots to
the garden. Pause closes the panel too, but its glyph is not shown.

**Assumed** — the back-out lines were "Back to the garden" and "Back to the slots". They lost
"Back" because on touch `[Mark]` reads as the Back button's label ([Touch](#touch)), which would
print "Back Back to the garden". With a key glyph they read as "[M] To the garden". Cost to
change: two strings.

**Assumed** — digging needs no confirmation, because a cancel refunds all of its food. Stopping a
dig needs a second press, because it throws away the work. Cost to change: one string either way.

**Assumed** — the panel acts on a selected slot and option with `[Interact]` and backs out with
`[Mark]`; Move left and right moves the selection (GAME.md "The nest", itself **Assumed**). If the
buttons change, only the glyphs in these strings change.

**Assumed** — Pause closes the panel, as Mark does, and also releases the cursor, as Pause does
anywhere else. Cost to change: Game layer only, no strings.

### Toasts — events

| key | text | when shown | max |
|---|---|---|---|
| `toast.build.started.brood` | Digging a brood chamber. Food spent: {cost} | `BuildStarted`, B = Brood | 60 |
| `toast.build.started.store` | Digging a store chamber. Food spent: {cost} | `BuildStarted`, B = Store | 60 |
| `toast.build.started.processing` | Digging a processing chamber. Food spent: {cost} | `BuildStarted`, B = Processing | 60 |
| `toast.build.cancelled` | Digging stopped. Its food goes back to the stores. | `BuildCancelled` | 60 |
| `toast.chamber.built.brood` | Brood chamber dug. Brood room +{n} | `ChamberBuilt`, B = Brood | 60 |
| `toast.chamber.built.store` | Store chamber dug. Food room +{n} | `ChamberBuilt`, B = Store | 60 |
| `toast.chamber.built.processing` | Processing chamber dug. | `ChamberBuilt`, B = Processing, and the colony is established on the same tick or already is | 60 |
| `toast.chamber.built.processing.wait` | Processing chamber dug. Established needs workers: {pop}/{need} | `ChamberBuilt`, B = Processing, and tier stays 1 for want of workers | 60 |
| `toast.brood.hatched` | Brood hatched. Workers +{n} | `BroodHatched` (at midnight) | 60 |
| `toast.brood.full` | Brood full. The queen waits for room: dig a brood chamber. | `BroodCapacityReached` | 60 |
| `toast.brood.lost.nurses` | Brood lost: {n}. Too few nurses. | `BroodLost`, B = 1. Sum A over 10 s and show once | 60 |
| `toast.brood.lost.starving` | Brood lost: {n}. The stores are empty. | `BroodLost`, B = 2. Same | 60 |
| `toast.nurses.pulled` | Nurses sent out: {n}. The brood is short until they return. | `NursesPulled` | 60 |
| `toast.food.empty` | The stores are empty. Workers will starve. | `FoodRanOut`. **Changed from M1** | 60 |
| `toast.workers.starved` | Workers starved: {n}. The stores are empty. | `WorkersStarved`. Sum A over 10 s and show once | 60 |
| `toast.stores.full` | The stores are full. Food lost: {n} | `StoresFull`, after `toast.item.delivered` | 60 |
| `toast.item.delivered.shelter` | Into the nest. Thatch: {n}/{max} | `ItemDelivered` for a find with `ShelterValue > 0`, in place of `toast.item.delivered` (whose Food +0 would be wrong) | 60 |
| `toast.item.delivered.shelter.full` | Into the nest, but the thatch was already whole. | Same, when `Shelter` was already `ShelterMax` before delivery | 60 |
| `toast.tier.up.2` | Established. More room to dig, and dead insects to take. | `TierChanged`, A = 2, B = 1. **Changed from M1** | 60 |
| `toast.tier.up.3` | Mature. The last slots open. Pinecones can be hauled. | `TierChanged`, A = 3. **Changed from M1** | 60 |
| `toast.tier.down.1` | Too few workers. No longer established. | `TierChanged`, A = 1, B = 2 | 60 |
| `toast.tier.down.2` | Too few workers. No longer mature. | `TierChanged`, A = 2, B = 3 | 60 |
| `toast.tier.blocked.2` | Enough workers to be established. Dig a processing chamber. | No event. The Game layer raises it when `Population` first reaches `TierThreshold(2)` at tier 1 without the chambers. Once per crossing | 60 |
| `toast.tier.blocked.3` | Enough workers to be mature. The nest still lacks chambers. | Same, for tier 3 | 60 |
| `toast.season.spring` | Spring begins. The queen lays at full pace again. | `SeasonChanged`, A = Spring. Not on the tick the year ends: the outcome screen covers it. **Changed from M1** | 60 |
| `toast.season.summer` | Summer begins. Dead insects are most common now. | `SeasonChanged`, A = Summer. **Changed from M1** | 60 |
| `toast.season.autumn` | Autumn begins. More finds, and the queen lays half as much. | `SeasonChanged`, A = Autumn. **Changed from M1** | 60 |
| `toast.season.winter` | Winter begins. No finds, no eggs. Workers eat more. | `SeasonChanged`, A = Winter, `Shelter < ShelterMax`. **Changed from M1** | 60 |
| `toast.season.winter.sheltered` | Winter begins. No finds, no eggs. The thatch holds. | Same, `Shelter == ShelterMax` | 60 |
| `toast.winter.warning` | Winter is close. Fill the winter bar: {food}/{need} | No event. At dawn of the 28th day of the year (`Clock.Day % YearDays == 27`): `floor(Food)`, `ceil(WinterFoodNeed)`, the same numbers the bar shows. **Changed in M5** | 60 |
| `toast.dawn.year` | Dawn. Year {year}, day {day} | Replaces `toast.dawn` from day 41 | 60 |
| `toast.colony.died` | The colony is gone. | `ColonyDied` after the outcome is recorded (in the sandbox), alongside the death card. Before that, the death card shows alone | 60 |

`BuildStarted`, `ChamberBuilt` and the rest name the chamber with one key per kind, so no
sentence is built from a chamber name.

**Assumed** — the winter warning comes three days before winter and is raised by the Game layer.
Cost to change: one constant. The text does not say "three days", so the constant can move
freely. From M5 it names the winter bar (`hud.winter`), so the warning sends the player to the
one place that keeps the number up to date.

**Assumed** — `BroodLost` and `WorkersStarved` toasts are summed over 10 s. Under-nursed brood can
be lost every 24 s, and one toast per loss is too many. Cost to change: one constant.

**Silent** in M2: `ItemSpawned` (the find appears), `ItemExpired` (the find fades), `YearEnded`
(the outcome screen opens).

### Toasts — command rejected

| key | text | when shown | max |
|---|---|---|---|
| `reject.no_such_chamber` | No such chamber. | `NoSuchChamberKind`. Development builds only; silent in release | 60 |
| `reject.slot_locked` | That slot opens when the colony is bigger. | `SlotLocked` | 60 |
| `reject.slot_taken` | Something is already dug there. | `SlotTaken` | 60 |
| `reject.build_in_progress` | One chamber at a time. Finish or stop this dig first. | `BuildInProgress` | 60 |
| `reject.not_enough_food` | Not enough food. The stores must hold more than {cost} | `NotEnoughFood`; `cost = def.FoodCost`. "More than" is exact: a dig never empties the stores | 60 |
| `reject.build.no_idle` | No idle workers to dig. Wait for some to come home. | `NothingToSend` from `StartBuild` | 60 |
| `reject.no_task_slot` | Too much under way. Let a haul finish first. | `NoTaskSlot` | 60 |
| `reject.no_such_build` | Nothing is being dug there. | `NoSuchBuild` | 60 |
| `reject.nothing.nurses` | The rest are nursing, and the brood cannot spare them. | `NothingToSend` from `TapAnt`, `Colony.Idle == 0` and `Colony.Nursing > 0` | 60 |

### Finds

Names and lift line for the four new finds. The world label shows the name, then `item.lift`
(M1), then one value line.

| key | text | when shown | max |
|---|---|---|---|
| `item.seed` | Seed | The interaction probe targets the find | 20 |
| `item.leaf` | Leaf | Same | 20 |
| `item.dead_insect` | Dead insect | Same | 20 |
| `item.pinecone` | Pinecone | Same | 20 |
| `item.value.food` | Food: {food} | Value line for a find with `FoodValue > 0` and `SpoilSeconds == 0` (sugar cube, seed, leaf) | 30 |
| `item.value.spoiling` | Spoiling. Food: {food} | Value line for a find with `SpoilSeconds > 0` (dead insect): current value, `round(FoodValue · (1 − spoil01))` | 30 |
| `item.value.shelter` | Thatch for winter, not food | Value line for a find with `ShelterValue > 0` (pinecone), `Shelter < ShelterMax` | 30 |
| `item.value.shelter.full` | The nest is already thatched | Same, `Shelter == ShelterMax` | 30 |
| `prompt.find.tier.2.processing` | [Mark] Needs a processing chamber | Replaces `prompt.find.tier.2` when `Population >= TierThreshold(2)`, so the chamber is all that is missing. Glyph drawn disabled | 40 |

The lift line stays `item.lift`, "Takes {n} to lift", for every find, the pinecone included. The
numbers (2, 4, 10, 16) carry the scale on their own.

### The year's outcome

The survived card is shown once, when `YearEnded` is raised. The death card is shown on
`ColonyDied`, whenever it comes, and again on `[Interact]` at the nest while the colony stays dead.
The headline is a big number under its label. The lines below it are the supporting lines.

| key | text | when shown | max |
|---|---|---|---|
| `outcome.title` | Winter is over | Outcome screen, `Survived` | 24 |
| `outcome.headline` | Workers alive | Label over the big number `Outcome.Workers` | 24 |
| `outcome.line.stores` | Food in the stores: {food} | `floor(Outcome.Food)` | 40 |
| `outcome.line.peak` | At its largest: {peak}, on day {day} | `Stats.PeakPopulation`, `Stats.PeakDay + 1` | 40 |
| `outcome.line.totals` | Hatched: {raised}. Starved: {starved}. Finds: {finds} | `WorkersRaised`, `WorkersStarved`, `Outcome.FindsHauled` | 48 |
| `outcome.keep_going` | That count is kept. The garden goes on, if you want to stay. | Under the lines, `Survived` | 70 |
| `outcome.continue` | [Interact] Keep going | Closes the screen; the sandbox runs on | 30 |
| `outcome.died.title` | The colony is gone | Outcome screen, `!Survived` | 24 |
| `outcome.died.headline` | Last day | Label over the big number `Outcome.EndDay + 1` | 24 |
| `outcome.died.keep_going` | The garden goes on without it. | Under the lines, `!Survived`. Shown with `outcome.line.peak` and `outcome.line.totals`; the stores line is left out, because it is always 0 | 70 |
| `outcome.new_colony` | [Interact] Start a new colony | Death card, first action. Also the HUD prompt at the nest while the colony is dead, where Interact brings the death card back | 30 |
| `outcome.new_colony.confirm` | [Interact] again to start over. The save is replaced. | Death card, in place of `outcome.new_colony` for 3 s after one press. A second press starts a new colony in the same garden | 48 |
| `outcome.died.continue` | [Mark] Stay in the garden | Death card, second action. Mark (or Pause) closes the card; the dead colony's garden runs on | 30 |

**Assumed** — the outcome screen's title is "Winter is over", which says the year ends as winter
breaks, not on a date. Cost to change: one string.

The death card offers a new colony because GAME.md assumes starting over is always offered; that
is separate from the player ant's own death, which GAME.md assumes ends in a respawn (SIM_M3.md §8).

**Assumed** — a colony that dies in the sandbox, after the year's outcome is recorded, opens the
death card as well as raising `toast.colony.died`, so starting over is offered the same way
whenever the colony dies. Cost to change: one branch in the Game layer, no strings.

**Assumed** — starting a new colony needs a second press within 3 s, the same window as
`nest.prompt.cancel.confirm`. It throws away the save, so it confirms like the other press that
throws work away. Cost to change: one constant.

### Saves

| key | text | when shown | max |
|---|---|---|---|
| `toast.save.restored` | The colony, as you left it. | After a load returns `Ok` or `Migrated`. A migrated save looks the same to the player | 60 |
| `dialog.save.too_old` | This version of the game cannot read your old colony. A new one begins. The old save is kept. | Load returns `TooOld`. The Game layer has backed the old save up and started a new colony | 100 |
| `dialog.save.too_old.ok` | [Interact] Begin | Button on `dialog.save.too_old` and `dialog.save.corrupt` | 30 |
| `dialog.save.too_new` | This colony was saved by a newer version. Reload the page to play it. Nothing has been overwritten. | Load returns `TooNew`. Nothing is started and the save is not touched | 100 |
| `dialog.save.too_new.reload` | [Interact] Reload the page | Button on `dialog.save.too_new` | 30 |
| `dialog.save.corrupt` | Your saved colony could not be read. A new one begins. The old save is kept. | Load returns `Corrupt` | 100 |

**Assumed** — a `Corrupt` save is handled like a `TooOld` one: it is backed up and a new colony
begins. SIM_M2.md §10.4 says what happens on `TooOld` and `TooNew` only. Cost to change: one
string and the Game layer's branch.

"Version" is used rather than "build", because players do not know the word "build".

---

## M3 — weather and threats

The strings for weather, spiders, birds, raids and the player ant's death. The rules are in
[GAME.md](GAME.md) ("Time, weather, threats") and [SIM_M3.md](SIM_M3.md). Voice, format and the
M2 parameter widths hold. New parameters, counted at their widest: `{s}` is whole seconds (3
digits); `{lost}`, `{def}`, `{won}`, `{raids}` and `{spiders}` are counts (3 digits). `{d}` keeps
its M2 meaning, days to one decimal place, so the defenders count that SIM_M3.md §14.4 calls
`{d}` is `{def}` here.

Every threat is generic: a spider, a bird, raiders from a rival colony. No species, and nothing
about how they hunt beyond what the simulation does.

### Threat words

The same few words carry every threat string, so the player learns one vocabulary for all of
them. A new threat string uses these words or adds a row here.

| word | means | never |
|---|---|---|
| taken | A worker killed by a spider or a bird | eaten, killed, died |
| lost | Workers killed in a fight at the nest; and the player ant ("You are lost") | killed, dead |
| driven off | A spider beaten by its defenders (GAME.md's phrase) | killed, defeated |
| fought off | A raid beaten at the nest | repulsed, won |
| defenders | Workers called to a spider, and every worker fighting at the nest | soldiers, guards |
| the shadow | The bird before it strikes. "The bird" once it has | hawk, any species |
| raiders | The rival colony's workers on a raid | enemies, invaders |
| heading out / turn home | Workers recalled by rain or a raid (M1's "turn home") | retreat, flee |
| washes out / walk it again | What rain does to a trail, and how the player renews it | fades (that is dry decay) |

**Assumed** — the rival is "the rival colony" and its workers "raiders". It has no name, because
the player never sees it. Cost to change: `threat.rival`, `threat.raiders` and the nine raid
strings below.

**Assumed** — the player ant is "lost", and workers are "taken". A worker taken by a spider and
the player lost to one are the same event to the simulation; the different word keeps "you"
distinct from the workers, as M1 did. Cost to change: the six player-death and respawn strings.

**Assumed** — the bird is "the shadow" until it strikes, because the shadow is all the view
draws (SIM_M3.md §14.2). Cost to change: five strings. If a bird model is ever drawn, "the shadow"
still reads correctly.

### Names

| key | text | when shown | max |
|---|---|---|---|
| `threat.spider` | Spider | World label over a spider the probe targets | 20 |
| `threat.bird` | Bird | Wherever the bird is named in a list (pause screen, outcome). Not in the world: the shadow has no label | 20 |
| `threat.raiders` | Raiders | Label of the HUD raid row while `Raid.Active` | 20 |
| `threat.rival` | The rival colony | Wherever the rival is named. M3 names it nowhere else | 24 |

### HUD additions

| key | text | when shown | max |
|---|---|---|---|
| `hud.weather.clear` | Clear | Beside the weather icon: `WeatherNow.Current == Clear` | 12 |
| `hud.weather.overcast` | Overcast | `Current == Overcast` | 12 |
| `hud.weather.rain` | Rain | `Current == Rain` | 12 |
| `hud.weather.rain_soon` | Rain soon | Forecast line under the weather: `Next == Rain` and `Current != Rain` | 16 |
| `hud.weather.dry_soon` | Rain ends soon | Forecast line: `Current == Rain` and `Next != Rain` | 16 |
| `hud.raid.marching` | Raiders: {n}. Time to the nest: {s} s | `Raid.Phase == Marching`: `Raiders`, `ceil(RaidSecondsToContact)` | 40 |
| `hud.raid.fighting` | Raiders: {n}. Defenders: {def} | `Phase == Fighting`: `Raiders`, `InNest` | 40 |
| `hud.raid.looting` | Raiders looting the stores: {n} | `Phase == Looting`: `Raiders` | 40 |
| `hud.defend` | At the spider: {here}/{need} | While a Defend task is `Recruiting` and the player is at its muster point or at the nest. `here = AtTarget` plus 1 if the player is within reach of `DefendMuster`; `need = SpiderDefenders`. Shown under `hud.party` if both apply | 26 |

**Assumed** — a "Rain ends soon" forecast as well as "Rain soon". The sim decides the next
interval's weather in advance either way (§1.1), and the end of rain is when the player should
go and walk a trail again. Cost to change: one string, no sim change.

The forecast lines carry no countdown. `SecondsToWeatherSample` would give one, but "soon" is a
quarter day at most, and a number would invite waiting by the clock instead of by the sky.

### Spider (world label)

The label shows `threat.spider`, then one status line, then the prompt.

| key | text | when shown | max |
|---|---|---|---|
| `spider.days_left` | Days until it leaves: {d} | Status line, unless hiding: `SpiderSecondsLeft / DayLengthSeconds`, one decimal place, at least 0.1 | 30 |
| `spider.hiding` | Hiding from the rain | Status line while `SpiderHiding(slot)` | 30 |

### Interaction prompts

| key | text | when shown | max |
|---|---|---|---|
| `prompt.spider.mark` | [Mark] Call defenders, then walk home | Probe on a spider within `SpiderReach + ThreatMarkMargin`, no draft, no Defend task on it, not `Fighting` | 40 |
| `prompt.spider.defended` | Defenders called. Keep clear of it. | Probe on a spider whose Defend task is `Recruiting` | 40 |
| `prompt.spider.fighting` | Defenders are fighting it. Keep clear. | Probe on a spider in `Fighting` | 40 |
| `prompt.trail.laying.defend` | Home to call defenders. [Mark] to abandon | Replaces `prompt.trail.laying` while the draft is for a spider (`Draft.Spider` set) | 40 |
| `prompt.trail.laying.defend.gone` | The spider has gone. [Mark] to abandon | Same, once the draft's spider no longer resolves or is `Gone`. Completing the walk would only raise `toast.trail.threat_gone` | 40 |
| `prompt.bird.alarm` | [Interact] Raise the alarm | `Bird.Active`, not `Alarmed`, the player within `AlarmReach` of the shadow and outside `BirdStrikeRadius` | 40 |
| `prompt.bird.alarm.under` | [Interact] Raise the alarm. Get clear. | Same, but the player is within `BirdStrikeRadius`: the strike would take you | 40 |
| `prompt.bird.raised.under` | Alarm raised. Get out from under it. | `Alarmed`, the player within `BirdStrikeRadius`. The alarm saves workers, not you | 40 |
| `prompt.find.refresh` | [Mark] Walk the trail again | Second prompt line under `prompt.find.claimed` while `CanRefresh(item)` and not raining | 40 |
| `prompt.find.refresh.rain` | [Mark] Walk it again. Rain washes it. | Same, while raining. Marking is accepted; the renewed trail washes back down in seconds | 40 |
| `prompt.find.mark.rain` | [Mark] Mark it. Nobody comes in rain | Replaces `prompt.find.mark` while raining. Marking is accepted; recruiting waits for dry weather | 40 |
| `prompt.nest.tap.defend` | [Tap] Send {n} to the spider | Replaces `prompt.nest.tap` when a tap would serve a Defend task (`TapAnt(−1)` prefers one, §5.2) | 40 |
| `prompt.nest.recalled.rain` | Raining. Nobody goes out until it stops. | At the nest while raining, in place of `prompt.nest.tap`, `prompt.nest.tap.defend` and `prompt.nest.wait` | 40 |
| `prompt.nest.recalled.raid` | A raid is on. Everyone stays in. | At the nest while `Raid.Phase` is `Marching` or `Looting`. Takes precedence over the rain line | 40 |
| `prompt.nest.rally` | Defenders fight harder while you stay. | At the nest while `Raid.Phase == Fighting` and `PlayerAtNest` | 40 |

`prompt.nest.tap.nurses` (M2) serves a defence as well as a haul: it names the nurses, not the
target.

**Assumed** — the find and nest prompts change in rain, though nothing is rejected but the tap.
Marking a find or re-walking a trail in rain is legal and nearly useless, and the prompt is the
only place the player learns that before trying. Cost to change: Game layer only, three strings.

### Respawn screen

Two lines, centred over the faded view while `!PlayerAlive`. The cause is in the toast that
opens it.

| key | text | when shown | max |
|---|---|---|---|
| `respawn.line` | A worker from the nest will take your place. | First line, always while down | 50 |
| `hud.respawn` | Out of the nest in: {s} s | Second line: `ceil(RespawnSecondsLeft)` while it is above 0 | 30 |
| `hud.respawn.waiting` | No worker in the nest to take your place. Waiting. | Second line in place of `hud.respawn` while `WaitingForWorker` | 50 |

A dead colony never returns the player (§8.3): the death card replaces this screen.

### Toasts — events

| key | text | when shown | max |
|---|---|---|---|
| `toast.weather.overcast` | Overcast. Fewer finds, and rain may follow. | `WeatherChanged`, A = Overcast, B = Clear. Not when `toast.weather.rain_soon` shows on the same tick | 60 |
| `toast.weather.clear` | The sky clears. | `WeatherChanged`, A = Clear, B = Overcast | 60 |
| `toast.weather.rain_soon` | Rain soon. Workers will turn home and trails wash out. | `WeatherForecast`, A = Rain. Wins over `toast.weather.overcast` raised on the same tick | 60 |
| `toast.weather.rain` | Rain. Workers heading out turn home. Trails wash out. | `WeatherChanged`, A = Rain | 60 |
| `toast.weather.dry` | The rain has stopped. Walk your trail again to renew it. | `WeatherChanged`, B = Rain, while the player has a `Recruiting` haul | 60 |
| `toast.weather.dry.plain` | The rain has stopped. Workers can go out again. | Same, with no `Recruiting` haul | 60 |
| `toast.trail.washed` | Your trail has washed out. Walk it again after the rain. | `TrailWashed`: rain drove the player's haul trail or a defend trail down to the floor, once per rain | 60 |
| `toast.trail.refreshed` | Trail renewed. Workers follow it out again. | `TrailRefreshed`. In rain, `toast.trail.washed` follows within seconds, which is the point | 60 |
| `toast.spider.arrived` | A spider has settled across a busy trail. | `ThreatSpawned`, A = Spider | 60 |
| `toast.spider.moved` | The spider has moved to a busier trail. | `SpiderMoved` | 60 |
| `toast.spider.left` | The spider has left the garden. | `SpiderLeft` | 60 |
| `toast.defend.marked` | Spider marked. Walk the trail back to the nest. | `TrailMarkStarted` with A < 0, in place of `toast.trail.started` | 60 |
| `toast.defend.laid` | Defenders called. They gather just short of the spider. | `TaskCreated` with B < 0, in place of `toast.trail.laid` on the same tick | 60 |
| `toast.defend.started` | The defenders close in on the spider. | `DefendStarted` | 60 |
| `toast.spider.driven_off` | The spider is driven off. The defenders head home. | `SpiderDrivenOff` | 60 |
| `toast.defend.failed` | The defenders were all taken. The spider stays. | `DefendFailed`, and `TaskAborted` with `DefendersLost`, which comes with it: shown once | 60 |
| `toast.ants_lost.spider` | Workers taken by a spider: {n} | `AntsLost`, B = Spider. Sum A over 10 s and show once, as M2's starvation toast does | 60 |
| `toast.bird.shadow` | A shadow on the trail. [Interact] near it to raise the alarm. | `ThreatSpawned`, A = Bird. The strike follows in 5 s | 60 |
| `toast.bird.alarm` | Alarm raised. The workers scatter. | `AlarmRaised` | 60 |
| `toast.ants_lost.bird` | Workers taken by the bird: {n}. The trail is scattered. | `AntsLost`, B = Bird. `BirdStrike` on the same tick is then silent | 60 |
| `toast.bird.missed` | The alarm saved them. The trail is scattered all the same. | `BirdStrike`, B = 1 | 60 |
| `toast.bird.empty` | The bird took nobody. The trail is scattered. | `BirdStrike`, A = 0, B = 0 (no worker in the strike) | 60 |
| `toast.raid.spotted` | Raiders spotted: {n}. Workers heading out turn home. | `ThreatSpawned`, A = Raid; `n` = B | 60 |
| `toast.raid.contact` | Raiders at the nest. Defenders fight harder with you there. | `RaidContact`, B > 0 | 60 |
| `toast.raid.contact.empty` | Raiders at the nest, and nobody home to fight them. | `RaidContact`, B = 0: the stores are looted at once | 60 |
| `toast.raid.broke_in` | The defence broke. Raiders are looting the stores. | `RaidBreached`, raised at the break-in. Not after `toast.raid.contact.empty` (an empty nest is looted at once) | 60 |
| `toast.raid.repulsed` | Raid fought off. Workers lost: {lost}. Food looted: {food} | `RaidRepulsed`: A, B | 60 |
| `toast.raid.breached` | The raiders have gone. Workers lost: {lost}. Food looted: {food} | Not shown: `toast.raid.looting_ended` replaced it at the end of the looting | 60 |
| `toast.raid.looting_ended` | The raiders leave with {food} food. {lost} workers lost. | `RaidLootingEnded`: A = lost, B = food. Raised when the looting ends, 30 s after the break-in | 60 |
| `toast.food.empty.raid` | The raiders have emptied the stores. | `FoodRanOut` while `Raid.Active`, in place of `toast.food.empty` | 60 |
| `toast.player.died.spider` | Too close to the spider. You are lost. | `PlayerDied`, A = Spider. Opens the respawn screen | 60 |
| `toast.player.died.bird` | Under the shadow when it struck. You are lost. | `PlayerDied`, A = Bird | 60 |
| `toast.player.died.raid` | Too close to the raiders. You are lost. | `PlayerDied`, A = Raid | 60 |
| `toast.player.respawned` | You are out again. The colony is one worker smaller. | `PlayerRespawned` | 60 |
| `toast.task.party_lost` | The party was taken. The find lies where it fell. | `TaskAborted`, `PartyLost`. The find can be marked again | 60 |
| `toast.task.threat_gone` | The spider has gone. The defenders come home. | `TaskAborted`, `ThreatGone` | 60 |
| `toast.trail.player_died` | The trail you were laying is lost with you. | `TrailAbandoned`, `PlayerDied`, after the `toast.player.died.*` of the same tick | 60 |
| `toast.trail.threat_gone` | The spider has gone. No need for defenders. | `TrailAbandoned`, `ThreatGone` | 60 |

**Silent** in M3: `AntsLost` with B = Raid (one a tick through a fight; the HUD counts and the
end-of-raid toast carry it). `BirdStrike` with A > 0, which follows `toast.ants_lost.bird`.
`WeatherForecast` for anything but rain (the HUD forecast line shows it). `WeatherChanged` from
Rain (the `toast.weather.dry` pair covers it).

**Assumed** — the bird warning names `[Interact]` in the toast itself. Five seconds is too short
to find a hint, and Interact does nothing else outside the nest. Cost to change: one string.

### Toasts — command rejected

| key | text | when shown | max |
|---|---|---|---|
| `reject.recalled.rain` | Not in the rain. Workers stay in until it stops. | `Recalled` while raining and no raid is active | 60 |
| `reject.recalled.raid` | Not during a raid. Workers stay in to fight. | `Recalled` while `Raid.Active` (wins over rain) | 60 |
| `reject.no_such_threat` | The spider has gone. | `NoSuchThreat` | 60 |
| `reject.defend_active` | Defenders are already called to this spider. | `DefendActive` | 60 |
| `reject.no_alarm_target` | Too far from the shadow, or it has passed. | `NoAlarmTarget` | 60 |

`PlayerDown` is silent: the respawn screen already says why nothing works. `OutOfReach` on a
spider and `DraftActive` reuse `reject.out_of_reach` and `reject.draft_active`, which read
correctly for a spider.

### The year's outcome — threat lines

Added under `outcome.line.totals` on both cards.

| key | text | when shown | max |
|---|---|---|---|
| `outcome.line.threats` | Lost to threats: {n}. Food looted: {food} | `Stats.WorkersKilled`, `floor(Stats.FoodLooted)` | 48 |
| `outcome.line.defence` | Raids fought off: {won}/{raids}. Spiders driven off: {spiders} | `RaidsRepulsed`, `RaidsRepulsed + RaidsBreached`, `SpidersDrivenOff` | 52 |
| `outcome.line.you` | Times you were lost: {n} | `Stats.PlayerDeaths` | 40 |

`outcome.line.defence` is the one 52-character line. The outcome card allows it; the rest stay
at 48.

### Field tap

Tap away from the nest calls nearby workers off other tasks onto your recruiting trail
([SIM_M3.md](SIM_M3.md) §5.6).

| key | text | when shown | max |
|---|---|---|---|
| `prompt.field.tap` | [Tap] Call {n} nearby to your trail | Away from the nest, no other prompt applies, and `TapPreview(task, PlayerPos) = n > 0` for the task `TapAnt(−1)` serves (a recruiting defence, else your recruiting haul) | 40 |
| `toast.ants.redirected` | Turning to your trail: {n} | `AntRedirected`, `n = A` | 60 |
| `reject.field.nothing` | No workers close enough to call. | `NothingToSend` from `TapAnt` away from the nest while a trail is recruiting | 60 |

A field tap with no recruiting trail reuses `reject.no_such_task` ("No trail is calling for
workers."). `reject.nothing.*` speak about the nest and are not shown in the field.

**Assumed** — all three lines above, written with the implementation rather than by the copy
pass. The prompt is the lowest priority away from the nest: a targeted find, a spider, the bird
or a trail being laid all win. Cost to change: three strings.

`toast.ants.redirected` leads with its number, against the Format rule above, and reads "1 workers"
for one. It is the text the field-tap brief asked for; a copy pass should reword it (for example
"Turned to your trail: {n}.").

---

## Credits and sound

One panel over the game, opened with Pause while the mouse is already free (the first Pause
releases it, the second opens the panel) and nothing else is open. Pause or `[Mark]` closes it, and
so does a click back into the game. Two volume rows at the top; below them the credits, which
are `Assets/ThirdParty/ATTRIBUTION.md` as shipped with the build.

| key | text | when shown | max |
|---|---|---|---|
| `hint.credits` | [Pause] Credits and sound | HUD, under the first-run hint, while the mouse is free and no panel or card is up. Not on touch | 30 |
| `credits.title` | Who made the garden | Panel title | 30 |
| `credits.volume.master` | Volume | First row: `AudioVolumes.Master` | 12 |
| `credits.volume.sounds` | Sounds | Second row: `AudioVolumes.Sfx`; Music follows it | 12 |
| `credits.volume.value` | {n}% | Each row's value, 0 to 100 in steps of 10 | 5 |
| `credits.prompt.move` | Up and down to read. Left and right to set the sound. | Panel foot | 60 |
| `credits.prompt.close` | [Mark] To the garden | Panel foot, under `credits.prompt.move` | 30 |
| `credits.missing` | The credits are missing from this build. | In place of the credits when the build has no attribution text | 50 |

**Assumed** — every line in this section. The title "Who made the garden" says the credits are
the people whose work is in the game, without the word "credits" twice on screen. The rows are
"Volume" and "Sounds", not "Master" and "Effects": a player does not think in mixer buses.
Written by the gameplay engineer, not the world-builder. Cost to change: one string each.

**Assumed** — there is no separate music row yet. The Sounds row moves Music with it, keeping
the default balance between them (Music is Sounds × 0.6 / 0.8). Cost to change: one row and one
string when the game gets music worth its own slider.

**Assumed** — `hint.credits` shows whenever the mouse is free, so a gamepad player, whose mouse
is never locked, always sees it. Cost to change: one condition in the HUD.

---

## M5 — Purpose

The strings that say what the year is for: the opening card, the winter bar, the season goals,
the antennae sense and what the nest can do. The rules are in [GAME.md](GAME.md) ("Purpose",
"Finds"). Voice, format and the M2 and M3 parameter widths hold. New parameters, counted at
their widest: `{season}` is a season name (12); `{done}` and `{total}` are counts (3); `{left}` is
whole days of winter left (2); `{slots}` is a slot count (2). `{n}` in `hud.find.distance` is at
most the sense range in body lengths, so 2 digits.

These keys change text in M5 and are listed in their M2 table only: `toast.winter.warning`.

### Words

| word | means | never |
|---|---|---|
| the winter bar | The HUD gauge labelled `hud.winter`: food stored against what winter will need | gauge, meter, progress |
| goal met | A season goal done | complete, achieved, unlocked |
| nearby / smell it | A find inside the antennae's range | detected, sensed, ping |
| body lengths | Distance to a find. One worker is about one unit long | cm, units, steps |

**Assumed** — the player calls the gauge "the winter bar", from its label "For winter" and the
nest panel's "Food for winter". Cost to change: the label, three goal and warning strings.

**Assumed** — the antennae sense is told as smell ("you can smell it from here"). The simulation
models a range, not a scent drifting on the air, so no string says which way the wind blows or
that a smell is strong or faint. Cost to change: the six scent toasts.

### Opening card

Shown over the garden on a new colony's first frame (`OutcomeScreenPresenter.Mode.Opening`). It
reuses the outcome card's layout: title, body, then the measure where the outcome card puts its
headline, then the action.

| key | text | when shown | max |
|---|---|---|---|
| `card.open.title` | The year ahead | Card title | 30 |
| `card.open.body` | A year is forty days. The last ten are winter: no new finds, and the colony lives on what it stored. What counts is the workers alive when spring returns. | Under the title | 160 |
| `card.open.measure` | Workers alive when spring returns | Where the outcome card's headline sits. It names the number that card will show as `outcome.headline` | 60 |
| `card.open.begin` | [Interact] Into the garden | The action. Interact, Mark or Pause closes the card; only Interact's glyph is drawn | 30 |

**Assumed** — the title "The year ahead", and "forty" and "ten" written into the body. If
`DaysPerSeason` changes, the body changes with it. Cost to change: one string.

### Winter bar

A row in the colony panel, under Food.

| key | text | when shown | max |
|---|---|---|---|
| `hud.winter` | For winter | Row label, always | 12 |
| `hud.winter.value` | {food}/{need} | Row value, spring to autumn: `floor(Food)` / `ceil(WinterFoodNeed)` | 14 |
| `hud.winter.cap_short` | Too little room. Dig a store chamber. | Under the row while `FoodCapacity < WinterFoodNeed` (class `winter--capped`), spring to autumn | 40 |
| `hud.winter.days` | Days of food: {d}/{left} | Row value in winter, in place of `hud.winter.value`: `FoodDaysLeft` to one decimal place, whole days of winter left | 30 |

"Room" in `hud.winter.cap_short` is the stores' food room, the word `chamber.store.effect` uses.

### Season goals

One line under the hint: the progress, the first goal not yet met, and its why, each a whole
string placed by the layout. Goals are guidance, not gates.

| key | text | when shown | max |
|---|---|---|---|
| `goal.progress` | {season}: {done}/{total} | Lead of the goals line: `season.{Clock.Season}`, goals met this season, goals this season | 24 |
| `goal.spring.cube` | Haul the sugar cube home | `FindsHauled(SugarCube) < 1` | 40 |
| `goal.spring.cube.why` | Its food pays for the first chamber you dig. | With the goal | 50 |
| `goal.spring.brood` | Dig a second brood chamber | `BuiltChambers.Brood < 2` | 40 |
| `goal.spring.brood.why` | The queen lays only while the brood has room. | With the goal | 50 |
| `goal.spring.workers` | Grow the colony. Workers: {have}/{need} | `Population < TierThreshold(2)`: `Population`, `TierThreshold(2)` | 40 |
| `goal.spring.workers.why` | Bigger finds need more workers to lift. | With the goal | 50 |
| `goal.summer.insect` | Haul a dead insect home | `FindsHauled(DeadInsect) < 1`, colony established | 40 |
| `goal.summer.insect.why` | Cut up in the nest, it feeds more than any find. | With `goal.summer.insect`. **Changed in Processing** | 50 |
| `goal.summer.insect.processing` | Become established to take dead insects | In place of `goal.summer.insect` while `Colony.Tier < 2` | 40 |
| `goal.summer.insect.processing.why` | The nest shows what is still missing. | With the processing line. The nest panel's next-tier list names the chamber and the workers | 50 |
| `goal.summer.store` | Dig a second store chamber | `BuiltChambers.Store < 2` | 40 |
| `goal.summer.store.why` | Winter can eat only what the stores can hold. | With the goal | 50 |
| `goal.summer.stores` | Store half the food winter needs | `Food < 0.5 · WinterFoodNeed` | 40 |
| `goal.summer.stores.why` | The winter bar shows how much is still to store. | With the goal | 50 |
| `goal.autumn.need` | Store all the food winter needs | `Food < WinterFoodNeed` | 40 |
| `goal.autumn.need.why` | No finds in winter. Only what is stored. | With the goal | 50 |
| `goal.autumn.thatch` | Thatch the nest with pinecones: {n}/{max} | `Shelter < ShelterMax`: `Shelter`, `ShelterMax` | 40 |
| `goal.autumn.thatch.why` | Each pinecone makes winter cheaper. | With the goal | 50 |
| `goal.autumn.thatch.mature` | Become mature to haul pinecones | In place of `goal.autumn.thatch` while `Colony.Tier < 3` | 40 |
| `goal.autumn.thatch.mature.why` | Pinecones take sixteen ants and the last slots. | With the mature line | 50 |
| `goal.autumn.spider` | Drive off a spider | `Stats.SpidersDrivenOff < 1` | 40 |
| `goal.autumn.spider.why` | A spider on a trail keeps taking workers. | With the goal | 50 |
| `goal.winter.alive` | Make the stores last until spring | Winter, `Food` has stayed above 0 | 40 |
| `goal.winter.alive.why` | Every worker alive in spring is counted. | With the goal | 50 |
| `goal.winter.failed` | The stores are empty. Workers starve. | Winter, once `Food` has reached 0. In place of `goal.winter.alive` for the rest of the winter | 40 |
| `goal.winter.failed.why` | Whoever lives to spring is still counted. | With the failed line | 50 |

The goals are written as orders and the whys as plain facts, so the line reads "do this, because
that" in any order the layout sets them.

`goal.autumn.spider.why` is the plan's reason: the next threat, raids, is set by the rival and the
colony's tier, so defending is worth learning before then. A spider's own cost would read more
directly ("A spider on a trail keeps taking workers.") if the plan's reason does not land in
the playtest.

### Toasts — goals

| key | text | when shown | max |
|---|---|---|---|
| `toast.goal.spring.cube` | Spring goal met. The sugar cube is in the stores. | `GoalTracker.JustCompleted`, that goal. After `toast.item.delivered` on the same frame | 60 |
| `toast.goal.spring.brood` | Spring goal met. The queen has room to lay. | Same. After `toast.chamber.built.brood` | 60 |
| `toast.goal.spring.workers` | Spring goal met. Workers enough for bigger finds. | Same | 60 |
| `toast.goal.summer.insect` | Summer goal met. The biggest find is home. | Same | 60 |
| `toast.goal.summer.store` | Summer goal met. More room for winter's food. | Same | 60 |
| `toast.goal.summer.stores` | Summer goal met. Half of winter's food is stored. | Same | 60 |
| `toast.goal.autumn.need` | Autumn goal met. Winter's food is stored, for now. | Same. "For now": the need grows with every worker hatched | 60 |
| `toast.goal.autumn.thatch` | Autumn goal met. The nest is thatched for winter. | Same | 60 |
| `toast.goal.autumn.spider` | Autumn goal met. A spider is driven off. | Same. After `toast.spider.driven_off` | 60 |
| `toast.goal.winter.alive` | — (not shown; the outcome card says it) | Dropped 2026-10-06: the card covers the frame it would appear on | — |
| `toast.goal.season.spring` | Every spring goal met. The colony is on its feet. | The last of a season's goals is met, after that goal's toast | 60 |
| `toast.goal.season.summer` | Every summer goal met. The stores are growing. | Same | 60 |
| `toast.goal.season.autumn` | Every autumn goal met. Let winter come. | Same | 60 |

A goal toast names its season, not the current one: a spider driven off in spring still reads
"Autumn goal met". Winter has one goal, so it has no season toast.

**Assumed** — the frame "{Season} goal met." at the start of every goal toast, so the player
learns to tell goal toasts from event toasts by their first words. Cost to change: thirteen strings.

### Antennae

| key | text | when shown | max |
|---|---|---|---|
| `hud.find.distance` | Body lengths: {n} | Small label under a find's name, in view and within `SenseRadius`: distance from the player in units, rounded | 16 |
| `hud.nest.marker` | Nest | The nest marker, more than `NestMarkerMinDistance` from the nest and not at it: over the mound when it is in view, else beside `hud.find.distance` under an edge chevron in the HUD's text colour. Hidden while a panel or card is up | 8 |
| `toast.scent.sugar_cube` | A sugar cube nearby. You can smell it from here. | `ItemSpawned` within the scent range, sugar cube | 60 |
| `toast.scent.seed` | A seed nearby. You can smell it from here. | Same, seed | 60 |
| `toast.scent.leaf` | A leaf nearby. You can smell it from here. | Same, leaf | 60 |
| `toast.scent.dead_insect` | A dead insect nearby. Haul it before it spoils. | Same, dead insect, `Colony.Tier >= 2` | 60 |
| `toast.scent.dead_insect.gated` | A dead insect nearby. Too big for a young colony. | Same, dead insect, `Colony.Tier < 2` | 60 |
| `toast.scent.pinecone` | A pinecone nearby. Thatch, once the colony is mature. | Same, pinecone, at any tier | 60 |

The three plain finds share one tail on purpose: the toast comes at most once a minute, and the
same words teach that the sense is a thing you have, not a lucky find.

`toast.scent.pinecone` holds at every tier: at Mature it is a reminder, below it a promise. It
needs no gated twin.

**Assumed** — the nest marker shows from 12 units (`NestMarkerMinDistance`, twelve body lengths)
out; inside that it is hidden, because the mound is 17 units across and the player is on or beside
it, where a marker would only cover it. It stays up while a trail is laid, since a trail is walked
toward the nest, though the find indicators step back then. Cost to change: one constant.

**Assumed** — the gated insect is "too big for a young colony", which is the player's sense of
it. The rule is a processing chamber and 25 workers, which the find's own prompt and the nest
panel spell out. Cost to change: one string.

### Nest panel — what the nest can do

One line under `nest-tier`.

| key | text | when shown | max |
|---|---|---|---|
| `nest.can.1` | Takes seeds, leaves and sugar cubes. Chamber slots: {slots} | Tier 1: `SlotsByTier[0]` | 70 |
| `nest.can.2` | Takes seeds, leaves, sugar cubes and dead insects. Chamber slots: {slots} | Tier 2: `SlotsByTier[1]` | 70 |
| `nest.can.3` | Takes every find, pinecones too. Chamber slots: {slots} | Tier 3: `SlotsByTier[2]` | 70 |

Each line lists everything the tier takes rather than "and dead insects too", so no line depends
on the player having read the one before.

**Assumed** — "Chamber slots" is the total the tier opens (4, 7, 10), dug or not. The cutaway
draws which are free. Cost to change: one parameter.

---

## Processing

The strings for the cutting room: a dead insect hauled home is cut up in the processing chamber
before its food reaches the stores. The rules are in [GAME.md](GAME.md) ("The nest") and
[SIM_M4_PROCESSING.md](SIM_M4_PROCESSING.md). Voice, format and parameter widths are as in M2.

These keys change text and are listed in their own tables only: `chamber.processing.desc`,
`chamber.processing.effect`, `goal.summer.insect.why`, and the "when shown" of
`nestview.processing.cutting`.

### Words

| word | means | never |
|---|---|---|
| cut, cut up | What the processing chamber does to a dead insect | process, butcher, prepare |
| to cut | Food in dead insects that are home but not yet cut (`RawFood`) | raw food, unprocessed |
| rot | A dead insect left uncut too long, lost | decay, expire, spoil (the garden's word for a find going off before it is hauled) |

**Assumed** — the player-facing word for raw food is "to cut", and for a carcass "a dead insect",
as in the garden. "Raw" is the code's word only. Cost to change: the strings below.

**Assumed** — the chamber keeps the name "Processing chamber" in every string; "the cutting room"
stays the fiction's name and is not shown. Renaming the chamber is one name and five strings.

### HUD additions

| key | text | when shown | max |
|---|---|---|---|
| `hud.cutting` | Cutting | Always; value `Colony.Processing`. Dimmed at 0. With Idle, On trails, Nursing and Digging it sums to Workers | 12 |
| `hud.raw` | To cut | Under Food, once a processing chamber is built; value `floor(RawFood)`. Dimmed at 0 | 12 |

### Nest panel

| key | text | when shown | max |
|---|---|---|---|
| `nest.raw` | Still to cut: {food} | Colony line, while `RawFood >= 1`: `floor(RawFood)` | 24 |
| `nest.raw.rot` | Rots if not cut. Days left: {d} | Processing card line (first built chamber), while the oldest dead insect has under half a day left: `CarcassDaysLeft(0)`, one decimal place | 40 |
| `nest.cut.short` | No idle workers free to cut. | Colony line, as a warning, while `Processing < ProcessingDemand` | 40 |
| `nest.cut.progress` | Cutting: {pct}% · {step} | Built processing slot with a dead insect in it: `Cut01`, floored; `step` is `nest.cut.step.{Step}` | 32 |
| `nest.cut.step.0` | On its back | `{step}` while `CutStep == 0`: dragged in whole, the legs coming off | 14 |
| `nest.cut.step.1` | Legs off | `CutStep == 1`: the head coming off | 14 |
| `nest.cut.step.2` | Head off | `CutStep == 2`: the shell being pried off | 14 |
| `nest.cut.step.3` | Shell pried | `CutStep == 3`: the soft parts carried to the stores | 14 |
| `nest.cut.crew` | Cutters: {crew}/{need} | Same: `CarcassCrew` / `ProcessingCrew` | 20 |
| `nest.cut.waiting` | Waiting for idle workers | Same, `CarcassCrew == 0`, in place of `nest.cut.crew` | 28 |
| `nest.cut.queue` | Next in line: {n} | First built processing slot, while dead insects wait for a free chamber: the count waiting | 20 |

### Toasts — events

| key | text | when shown | max |
|---|---|---|---|
| `toast.item.delivered.raw` | Into the processing chamber. Food to come: {food} | `ItemDelivered` of a `Raw` find, `B > 0`, in place of `toast.item.delivered`; `food = B` | 60 |
| `toast.carcass.cut` | A dead insect is cut up and in the stores. | `CarcassCut`, B = 4 | 60 |
| `toast.carcass.rotted` | A dead insect rotted before it was cut. Food lost: {n} | `CarcassRotted`, B = 1 | 60 |
| `toast.carcass.no_room` | The processing chamber is full. Food lost: {n} | `CarcassRotted`, B = 2, in place of `toast.item.delivered.raw` on the same tick | 60 |
| `toast.cut.short` | No idle workers to cut the dead insect. It may rot. | No event. The Game layer raises it when `Processing < ProcessingDemand` has held for 30 s. Once per shortage | 60 |

The step names say what the dead insect looks like now, the stage the cutaway draws
([SIM_M4_PROCESSING.md](SIM_M4_PROCESSING.md) §9.1), not the cut under way, so the card and the
picture always agree. The percentage says how far along it is. Each step name is a label on its own:
no article, no sentence, so `nest.cut.progress` can put it anywhere a translation needs it.

**Assumed** — the step names describe the insect's state ("Legs off"), not the work ("Cutting the
legs"); step 0 is "On its back" rather than "Whole", because "Cutting: 12% · Whole" reads as a
contradiction. `nest.cut.progress` gains `{step}` and a max of 32. Cost to change: four strings; the
card line must fit "Cutting: 100% · Shell pried" plus 30%.

A cut step is silent: the HUD's food rises and the panel's percentage moves. Food that does not fit
when a step is cut raises `toast.stores.full` as a delivery does.

### The year's outcome — rot line

| key | text | when shown | max |
|---|---|---|---|
| `outcome.line.rotted` | Food rotted before it was cut: {food} | Under the threat lines on both cards, only while `floor(Stats.FoodRotted) >= 1` | 40 |

### Nest view

| key | text | when shown | max |
|---|---|---|---|
| `nestview.processing.rotting` | Going off | Over a dead insect with under half a day before it rots | 16 |
| `nestview.processing.nohands` | Nobody to cut it | Over a dead insect in a processing chamber with no cutters | 20 |

Neither shows in the normal state: a dead insect waiting its turn by the door, or being cut, has no
label but `nestview.processing.cutting`.

---

## Touch

Phones and tablets, in landscape. A left-thumb joystick moves, a drag on the right half of the
screen looks, and round buttons in the bottom-right corner stand in for the keys; Menu sits in the
top-right corner. "Touch device" here means the web page's test: a touch screen and no fine
pointer, or `?touch=1` in the address. The labels are the device's button names, so the
`[Mark]`, `[Tap]`, `[Interact]` and `[Pause]` glyphs in every other string read as the same words
("Mark", "Tap", "Interact", "Menu") on touch. The exception is while the nest panel or a card is
open: the Mark and Interact buttons are labelled Back and Confirm, and the glyphs follow the
buttons, so `[Mark]` reads "Back" and `[Interact]` reads "Confirm" in every string shown then (the
nest panel's prompts, the outcome card). A glyph always names the word on the button it means.
Button labels are one word, no full stop.

| key | text | when shown | max |
|---|---|---|---|
| `touch.button.mark` | Mark | The largest button, while no panel is open | 8 |
| `touch.button.tap` | Tap | Button, while no panel is open | 8 |
| `touch.button.interact` | Interact | Interact button, while no panel is open and neither label below applies | 8 |
| `touch.button.interact.nest` | Nest | Interact button, at the nest, where Interact opens the nest panel | 8 |
| `touch.button.interact.alarm` | Alarm | Interact button, where Interact raises the alarm against the bird | 8 |
| `touch.button.sprint` | Run | The smallest corner button, while no panel is open. A latch: tap to run, tap again to walk | 5 |
| `touch.button.pause` | Menu | Top-right corner, always. Opens credits and sound on the first press | 5 |
| `touch.button.back` | Back | The Mark button, while the nest panel or a card (the outcome card, credits) is open. `[Mark]` reads as this word then | 8 |
| `touch.button.confirm` | Confirm | The Interact button, at the same times. `[Interact]` reads as this word then | 8 |

`hint.credits` is not shown on touch.

**Assumed** — `hint.credits` is hidden on touch because the Menu button says what it is and
is always on screen. Cost to change: one condition in the HUD.

**Assumed** — the button words are the action names, plus "Run" for Sprint (shorter, and what
a player calls it) and "Menu" for Pause (there is nothing to pause or release on touch: it opens
credits and sound). Written with the touch plan, not by a copy pass. Cost to change: one string
each; Sprint and Menu are the device's button names, so `InputGlyphs` follows them.

**Assumed** — inside the nest panel and on cards the prompts name the buttons as labelled
("Confirm Dig a brood chamber", "Back To the garden"), rather than the buttons taking the action
names. "Mark" on a button that closes a panel would read as laying a trail. This changes how earlier
settled prompts read on touch, not their keys. Cost to change: the glyph rule in `InputGlyphs` for
the two modes.

**Assumed** — Run is a latch rather than a button held down, because a thumb cannot hold it
and steer the camera at once. It turns itself off half a second after the joystick is let go.
Cost to change: cheap, one behaviour in the overlay.

---

## Nest view

Short labels pinned beside the animated cutaway: on the queen, a brood cluster, a store pile, the
cutting room, the shaft. What the view shows and why is in [NEST_VIEW_THEME.md](NEST_VIEW_THEME.md);
which counts drive it is in [NEST_VIEW.md](NEST_VIEW.md). Voice and format are as above, and none of
these take a parameter: the numbers stay in the panel lines.

The normal state has no label. An egg being laid says "laying" on its own; a label appears only
where something is wrong or about to change, so a label always means *look here*. Each one names
what the player can see, not what to do about it: the panel lines (`nest.queen.full`,
`nest.nurses.short`, …) carry the instruction.

| key | text | when shown | max |
|---|---|---|---|
| `nestview.queen.blocked` | No room for her eggs | Beside the queen while `QueenBlocked` | 24 |
| `nestview.queen.still` | Not laying | Beside the queen while `EggsPerDay == 0` and not `QueenBlocked` (no food, or winter) | 16 |
| `nestview.brood.hatching` | Hatching at midnight | Over the oldest brood cluster while `BroodHatchIn[0] > 0` | 24 |
| `nestview.brood.untended` | Too few nurses | Over the brood clusters while `Nursing < NurseDemand` | 20 |
| `nestview.store.full` | Full to the door | Over a store pile drawn at its chamber's room ([NEST_VIEW.md](NEST_VIEW.md) splits the food between piles) | 20 |
| `nestview.store.spilled` | Lost at the door | At a store doorway for a few seconds after a delivery that did not fit | 20 |
| `nestview.store.starving` | Starving | Over the store chambers while `Food == 0` | 12 |
| `nestview.processing.cutting` | Cutting up a dead insect | In a processing chamber while a carcass is in it with cutters at work (`ChamberSlot >= 0`, `Crew > 0`, [SIM_M4_PROCESSING.md](SIM_M4_PROCESSING.md) §9) | 32 |
| `nestview.winter.huddle` | Huddled against the cold | Over the queen's chamber in winter | 32 |
| `nestview.winter.gaps` | Gaps in the thatch | At the entrance in autumn and winter while `Shelter < ShelterMax` | 24 |
| `nestview.raid.defenders` | Defenders at the entrance | At the top of the shaft while `Raid.Phase == Fighting` | 32 |
| `nestview.raid.looting` | Raiders in the stores | Over the store chambers while `Raid.Phase == Looting` | 28 |

"Full to the door" and "Lost at the door" pick up `chamber.store.desc` ("What will not fit is lost at
the door"), so the player meets the same picture in the build option and in the nest.

**Assumed** — labels mark exceptions only, and there is no label for the queen laying, a dig, or the
landing. Cost to change: cheap, one string per state added.

**Assumed** — "Cutting up a dead insect" names what processing is (the cutting room, in
NEST_VIEW_THEME.md). It is the only string that says so. Cost to change: one string.

**Assumed** — `nestview.store.spilled` needs the view to know a delivery overflowed. If the sim does
not report the food lost at the door, the label is left out until it does. Cost: one event field.

---

## Years

The strings for carrying a colony into later years: the outcome card's two ways on, the run's lines,
the card that opens each new year, and the HUD year. The rules are in [GAME.md](GAME.md) ("The
session") and [SIM_M5_NEXT_YEAR.md](SIM_M5_NEXT_YEAR.md). Voice, format and parameter widths hold.
New parameters, at their widest: `{year}` and `{years}` are 2 digits, `{days}` and `{n}` 1.

Changed elsewhere, with no new text: the survived card no longer shows `outcome.continue`. In a run
the card offers the two actions below, and in the sandbox there is no card. `hud.day.year` and
`toast.dawn.year` read `year = World.Year` and `day = DayOfYear + 1` from year 2 on.

### The outcome card in a run

The survived card opens on `YearEnded` only while `AwaitingYear` (a run), and again when a save taken
at the hold is loaded. Interact carries on; Mark (or Pause) stays. Neither confirms twice: both keep
the colony.

| key | text | when shown | max |
|---|---|---|---|
| `outcome.carry_on` | [Interact] Carry on to year {year} | First action; `year = Year + 1`. Sends `BeginYear` | 30 |
| `outcome.stay` | [Mark] Stay in the garden | Second action. Sends `StayInGarden` | 30 |
| `outcome.run.keep_going` | Carry on into a harder year, or stay in this garden as it is. | In place of `outcome.keep_going` while `AwaitingYear` | 70 |
| `outcome.line.last_spring` | Last spring: {workers} | Under the headline from year 2: the previous record's `Workers` | 24 |

### The run's lines

Under the year's lines, on the survived card from year 2 and on the death card whenever `Years`
holds a survived year.

| key | text | when shown | max |
|---|---|---|---|
| `run.line.years` | Years survived: {years} | `YearsSurvived` | 24 |
| `run.line.best` | Best spring: {workers}, in year {year} | The survived record with the most `Workers`; the earliest on a tie | 40 |
| `run.line.totals` | All years. Hatched: {raised}. Lost to threats: {killed} | `RunStats + Stats`: `WorkersRaised`, `WorkersKilled` | 52 |
| `run.died.keep_going` | The run ends here. The garden goes on without it. | In place of `outcome.died.keep_going` when the colony dies in year 2 or later | 70 |
| `toast.year.garden` | Spring again, year {year}. Workers alive: {workers} | A year ends in the sandbox (`YearEnded`, `Mode == Sandbox`): no card | 60 |

### The new year's card

Shown on `YearBegan`, in the opening card's layout: title, body, one line on what is new, the
measure, the action. Interact, Mark or Pause closes it. It is not shown on a load.

| key | text | when shown | max |
|---|---|---|---|
| `card.year.title` | Year {year} | Card title | 30 |
| `card.year.body` | The colony wakes to a harder garden: fewer finds, more threats, a stronger rival colony and a colder winter. | Under the title | 160 |
| `card.year.new.winter_raids` | New this year: the rival colony raids in winter too. | `WinterRaidsPerDay` rose from 0 at this level | 70 |
| `card.year.new.berries` | New this year: fallen berries. Rich food, but they spoil in a day. | A find's `FromLevel` equals this level | 70 |
| `card.year.new.early_winter` | New this year: winter comes {days} days early. | `EarlyWinterDays` rose from 0; `days` = its value | 70 |
| `card.year.new.spiders` | New this year: up to {n} spiders at once. | `SpiderMax` rose; `n` = its value | 70 |
| `card.year.new.none` | Nothing new this year. The garden is as hard as it gets. | Level unchanged from last year (year 6 on) | 70 |
| `card.open.measure` | (unchanged) | Reused, where the headline sits | — |
| `card.open.begin` | (unchanged) | Reused as the action | — |

The "new" line is found by comparing this level's row with the last one, so the sim holds no
presentation field. If more than one thing changes at a level, the first in the table order wins.

### Finds

| key | text | when shown | max |
|---|---|---|---|
| `item.berry` | Fallen berry | The interaction probe targets the find; antennae label | 20 |

**Assumed** — "Fallen berry" as the new find's name, and "Rich food" rather than a number on its
card line. It is a placeholder for `world-builder`. Cost to change: two strings.

**Assumed** — the season toasts are reused for early winter: "Winter begins." on day 28 needs no new
string. Cost to change: one string and one condition.

---

## Web page

The loading screen, start prompt and error panel of the web build. These strings live in
`Assets/WebGLTemplates/AntGame/index.html`, not in `Strings.cs`: they show before the game runs.
`web.start.touch` and `web.rotate` are mirrored in `Strings.cs` as well; the template's copy is the
one players see.
The title is the product name, `ant-game`.

| key | text | when shown | max |
|---|---|---|---|
| `web.tagline` | You are one ant. The colony follows your trail. | Under the title, while loading and at the start prompt | 60 |
| `web.loading` | Loading | Under the bar while the build downloads | 16 |
| `web.starting` | Starting | Under the bar once the download is done and the engine starts | 16 |
| `web.start` | Click to start | Once the game is running. One click starts it and locks the mouse | 20 |
| `web.start.touch` | Tap to start | In place of `web.start` on a touch device. One tap starts it; there is no mouse to lock | 20 |
| `web.start.hint` | Esc releases the mouse. | Under `web.start`. Not on a touch device | 40 |
| `web.rotate` | Turn your device sideways | Over the whole page on a touch device held upright. The game keeps running underneath | 40 |
| ~~`web.mobile`~~ | ~~Desktop browsers only, for now.~~ | Removed in M6: phones and tablets play with on-screen controls | — |
| `web.error.title` | The game could not start. | The build failed to load | 40 |
| `web.error.text` | Reload the page to try again. A browser with WebGL 2 works best. | Under `web.error.title`, with the loader's message below in small type | 80 |
| `web.error.runtime.title` | Something went wrong. | A script error after the game started | 40 |
| `web.error.runtime.text` | The game hit an error and may have stopped. Reload the page to carry on from the last save. | Under `web.error.runtime.title` | 100 |
| `web.error.reload` / `web.error.dismiss` | Reload / Dismiss | Buttons on the error panel. Dismiss only after a runtime error | 12 |

**Assumed** — the tagline is "You are one ant. The colony follows your trail." It was written
with the loading screen, not by the world-builder. Cost to change: one string in the template.
