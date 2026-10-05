# UI copy — M1 vertical slice

Every string the player reads in the first playable slice. Code references the **key**; the
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
| `tier.1` / `.2` / `.3` | Young / Established / Mature | Colony tier: below 25 workers / 25–74 / 75 or more | 12 |

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
| `hud.out` | On trails | Always; value `AntsOutside`. Makes the three counts sum to Workers | 12 |
| `hud.nursing` | Nursing | Always, dimmed; value from `hud.nursing.value` | 12 |
| `hud.nursing.value` | {n} reserved | Value of the nursing row. Nurses never leave in M1 | 14 |
| `hud.food` | Food | Always; value `floor(Colony.Food)` | 12 |
| `hud.food.empty` | Empty | Replaces the food value while `Food == 0` | 12 |
| `hud.tier` | Colony | Always; value `tier.{Colony.Tier}` | 12 |
| `hud.day` | Day {day} | Always; `day = Clock.Day + 1` | 12 |
| `hud.party` | At the find: {here}/{need} | While the player's task is `Recruiting`. `here` is `AtTarget` plus 1 if the player is in reach | 26 |
| `time.dawn` … `time.night` | Dawn / Morning / Midday / Afternoon / Dusk / Night | Beside the day. Time of day 0.25 / 0.30 / 0.45 / 0.55 / 0.70 / 0.75 onward. Night runs 0.75–0.25 and matches `Clock.IsNight` | 12 |

**Assumed** — time of day shows as a word, not a clock. The bands are presentation only:
night uses the simulation's `IsNight` edges, and the daylight words are split by eye. Cost to change: cheap.

## Finds (world label over the targeted find)

| key | text | when shown | max |
|---|---|---|---|
| `item.sugar_cube` | Sugar cube | The interaction probe targets the find | 20 |
| `item.seed` `.leaf` `.dead_insect` `.pinecone` | Seed / Leaf / Dead insect / Pinecone | Same (M2 content) | 20 |
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
| `toast.food.empty` | The stores are empty. | `FoodRanOut` | 60 |
| `toast.tier.up.2` | The colony is established now. | `TierChanged`, A = 2, up | 60 |
| `toast.tier.up.3` | The colony is mature now. | `TierChanged`, A = 3 | 60 |
| `toast.tier.down` | The colony has shrunk. | `TierChanged`, down | 60 |
| `toast.night` | Night. Out on the trails, everything slows. | `NightStarted` | 60 |
| `toast.dawn` | Dawn. Day {day} | No event. The Game layer raises it when time of day crosses `NightEnd01` | 60 |
| `toast.season.spring` … `.winter` | Spring begins. / Summer begins. / Autumn begins. / Winter begins. | `SeasonChanged`, A = season | 60 |
| `toast.trail.cancelled` | Trail abandoned. The find stays where it is. | `TrailAbandoned`, `Cancelled` | 60 |
| `toast.trail.too_long` | Trail lost. Too far from the nest. | `TrailAbandoned`, `TooLong` | 60 |
| `toast.trail.too_short` | Too close to the nest to need a trail. | `TrailAbandoned`, `TooShort` | 60 |
| `toast.trail.item_gone` | Trail lost. The find is gone. | `TrailAbandoned`, `ItemUnavailable` | 60 |
| `toast.trail.no_slot` | Too many trails. Let one fade first. | `TrailAbandoned`, `NoFreeSlot` | 60 |

M1 never raises `TierChanged` or `SeasonChanged`. Population is fixed at 12, and a season lasts
an hour. The strings exist so that M2 needs no copy pass.

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
| `reject.nothing.idle` | No idle workers left in the nest. | `NothingToSend` and `Colony.Idle == 0` | 60 |
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
