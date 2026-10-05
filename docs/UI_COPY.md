# UI copy

Every string the player reads. The sections from *Names* to *First-run hints* are the M1
vertical slice; [M2 — colony building](#m2--colony-building) adds the nest, the year and saves. Code references the **key**; the
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

- `{name}` is a parameter. `[Mark]`, `[Tap]` and `[Interact]` are key glyphs drawn from the
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
If death becomes "respawn as a fresh worker" (**Undecided** in GAME.md), a worker number is
the natural way to show a new body. That means adding a number field and a format string.

**Assumed** — the tier names are Young, Established and Mature. Cost to change: three strings.
If tier comes to depend on chambers (M2), the names still read correctly.

## HUD

| key | text | when shown | max |
|---|---|---|---|
| `hud.workers` | Workers | Always; value `Population` | 12 |
| `hud.idle` | Idle | Always; value `Colony.Idle` | 12 |
| `hud.out` | On trails | Always; value `AntsOutside`. With Idle, Nursing and Digging it sums to Workers | 12 |
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

**Silent** events get sound and visuals, but no text. `DayStarted` fires at midnight, in the dark: the HUD day updates and `toast.dawn` greets the day. `AntLeftNest` and `AntReturned` fire six at a time, so a toast would be spam. `TaskCreated` and `TaskComplete` share a tick with `TrailLaid` and `ItemDelivered`. `TrailFaded`: the line fades on screen. `ItemSpoiled` is followed by `toast.task.item_gone` if the find was marked. `AntsLost`, `WeatherChanged` and `ThreatSpawned` are not raised in M1 and have no copy.

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
| `chamber.processing.desc` | Needed to become established, and so to take dead insects. | Same | 60 |
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
| `chamber.processing.effect` | Dead insects, once established | Built processing slot | 32 |

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
| `nest.prompt.close` | [Mark] Back to the garden | Panel open, the build options not showing. Mark (or Pause) closes the panel | 40 |
| `nest.prompt.back` | [Mark] Back to the slots | The build options are showing. Mark hides them and returns to slot selection | 40 |

The two back-out lines name where Mark takes you: from the options to the slots, from the slots to
the garden. Pause closes the panel too, but its glyph is not shown.

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
| `toast.winter.warning` | Winter is close. Stores: {food}. Winter needs: {need} | No event. At dawn of the 28th day of the year (`Clock.Day % YearDays == 27`): `floor(Food)`, `ceil(WinterFoodNeed)` | 60 |
| `toast.dawn.year` | Dawn. Year {year}, day {day} | Replaces `toast.dawn` from day 41 | 60 |
| `toast.colony.died` | The colony is gone. | `ColonyDied` after the outcome is recorded (in the sandbox), alongside the death card. Before that, the death card shows alone | 60 |

`BuildStarted`, `ChamberBuilt` and the rest name the chamber with one key per kind, so no
sentence is built from a chamber name.

**Assumed** — the winter warning comes three days before winter and is raised by the Game layer.
Cost to change: one constant. The text does not say "three days", so the constant can move
freely.

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
does not settle the **Undecided** on the player ant's or the queen's death.

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
