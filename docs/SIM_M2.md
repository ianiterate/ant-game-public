# Simulation spec — M2 colony building

Implementation spec for M2 in `Assets/Scripts/Sim`. It builds on [SIM.md](SIM.md) (M1, which
the code matches) and replaces only what it says it replaces. [GAME.md](GAME.md) is what the
game is; this file is how the simulation does it. Run `/open-questions` for everything marked
below.

Units as in M1: distance in **u** (1 u = 1 cm), time in **s**, rates per second unless a name
says per day. `dt` = `TickSeconds` (0.1 s). A day is `DayLengthSeconds` (360 s); `perDay · dt /
DayLengthSeconds` is the per-tick amount of any per-day rate. "Day `d`" means `Clock.Day == d`.

M2 adds: chambers and digging (§1), queen, brood and nursing (§2), real upkeep and starvation
(§3), tier with chambers and hysteresis (§4), finds that spawn, expire and spoil on their own
(§5), the year and its outcome (§6), saves (§10), and the panel's API (§12). There are no scout
ants (§11).

---

## 0. Ground rules and what changes from M1

All M1 ground rules hold (SIM.md §0): slot arrays, index order, no allocation per tick, gen bumped
when a slot is taken, terminal states freed next tick by the owning system.

### 0.1 Tick order (replaces SIM.md §6.3)

```csharp
public void Tick(in SimInput input)
{
    Events.Clear();
    StepClock();                                  // 1  clock; sets DayStartedThisTick
    CommandSystem.Step(this, in input, ref _rng); // 2  commands (+StartBuild, CancelBuild), draft sampling
    TrailSystem.Step(this);                       // 3  decay, floor, fade
    TaskSystem.Step(this, in input, ref _rng);    // 4  haul tasks as M1, then build tasks (§1.4)
    AntSystem.Step(this);                         // 5  movement, arrival, join, deposit, return
    ItemSystem.Step(this);                        // 6  free, age, spoil, expire, haul, delivery (§5)
    SpawnSystem.Step(this, ref _rng);             // 7  new finds (§5.2)
    ColonySystem.Step(this);                      // 8  hatch, upkeep, starvation, brood, laying, nursing, tier, outcome (§7)
    // Weather (M3) goes between 1 and 2; Threats (M3) after 8.
}
```

`DayStartedThisTick` is an internal bool set by `StepClock` (`_clock.Day != before.Day`). It is
not state: it is false between ticks and is neither hashed nor saved.

### 0.2 New files

`Sim/Colony/`: `ChamberKind.cs` (`ChamberKind`, `ChamberKinds`, `ChamberState`, `ChamberDef`,
`ChamberCounts`), `Chamber.cs`, `ColonyStats.cs` (`ColonyStats`, `YearOutcome`), `NestQueries.cs`
(the derived reads of §12, as `World` partial or static helpers).
`Sim/Items/SpawnSystem.cs`. `Sim/Save/SaveData.cs`, `SaveCodec.cs`, `SaveMigrations.cs`,
`SaveBits.cs`. Brood and building have no separate system: the queen and brood live in
`ColonySystem`, digging in `TaskSystem` (the plan's `Brood`, `Queen`, `BuildSystem` files are
folded in; there is too little in each to justify a file).

### 0.3 `SimLimits` additions

```csharp
public const int MaxChambers = 10;           // panel slots; SlotsByTier may not exceed it
public const int SpawnPlacementTries = 8;    // rejection-sampling attempts per spawn
```

### 0.4 `SimConfig` additions

Existing fields keep their meaning. Add:

| Field | Default | Meaning |
|---|---|---|
| **Queen and brood** | | |
| `QueenEggsPerDay` | `5f` | Laying rate when fully fed, before the season factor. |
| `QueenFedDays` | `2f` | The queen lays fully while stores hold this many days of current upkeep (§2.3). |
| `LayFactorBySeason` | `{1, 1, 0.5f, 0}` | Laying multiplier, Spring…Winter. Length 4. |
| `BroodMaturationDays` | `7` | Days from egg to worker. Also the length of the brood array. |
| `BroodUpkeepPerDay` | `0.5f` | Food per brood per day. |
| `QueenNurses` | `1` | Nurses the queen needs with no brood. |
| `BroodPerNurse` | `5` | Brood one nurse can tend. |
| `NurseReserveFraction` | `0.5f` | A tap may take nurses down to `ceil(this × demand)`, never lower (§2.5). |
| `UnderNursedBroodLossPerDay` | `2f` | Brood loss per day per brood at zero nursing coverage (§2.6). |
| **Upkeep and starvation** | | |
| `WinterUpkeepFactor` | `1.25f` | Worker upkeep multiplier in winter with no shelter (§3.1). |
| `ShelterMax` | `4` | Shelter at which winter costs no more than summer. |
| `StarveWorkerFractionPerDay` | `0.1f` | Fraction of workers in the nest that die per day while `Food == 0`. |
| `StarveMinDeathsPerDay` | `1f` | Floor on that death rate. |
| `StarveBroodFractionPerDay` | `0.5f` | Fraction of brood lost per day while `Food == 0`. |
| **Chambers** | | |
| `ChamberDefs` | see §1.1 | Indexed by `(int)ChamberKind`, length `ChamberKinds.Count`. |
| `BroodCapacityBase` | `4` | Brood the queen's chamber holds with no brood chamber. |
| `FoodCapacityBase` | `50f` | Food the queen's chamber holds with no store chamber. |
| `BuildCrew` | `4` | Workers who dig one chamber. |
| `SlotsByTier` | `{4, 7, 10}` | Unlocked panel slots at tier 1, 2, 3. Length `TierThresholds.Length + 1`. |
| `StartingChambers` | `{Brood, Store}` | Built at start in slots 0, 1, … |
| **Tier** | | |
| `TierChambers` | `{(0,0,1), (2,2,1)}` | Built chambers (Brood, Store, Processing) needed for tier 2, tier 3. Parallel to `TierThresholds`. |
| `TierDownMargin` | `5` | A tier is lost only when population falls this far below its threshold (§4). |
| **Finds** | | |
| `SpawnMinSpacing` | `8f` | u between a new find and any live find. |
| `SpawnJitter` | `0.5f` | Each spawn interval is the mean interval × `[1 − j, 1 + j)`. |

**Changed defaults** (both were 8 and 4 in M1): `StartingIdle = 10`, `StartingNursing = 2`. With
`StartingBrood = 4` the nursing rule (§2.4) needs 2 nurses at start, so the config now states
the true starting split. The total is still 12 workers. `Assets/Settings/SimConfig.asset` holds
serialized 8/4: set them in the inspector. Unity keeps a serialized value over a new initialiser.
Unchanged: `UpkeepPerAntPerDay 0.5`, `TierThresholds {25, 75}`, `StartingFood 20`,
`StartingBrood 4`.

The `World` constructor validates these arrays and throws `ArgumentException` on a bad one.
It checks: `LayFactorBySeason.Length == 4`; `SlotsByTier.Length == TierThresholds.Length + 1`,
non-decreasing, every value `<= MaxChambers`; `TierChambers.Length == TierThresholds.Length`;
`StartingChambers.Length <= SlotsByTier[0]`; `ChamberDefs` indexed by kind; and
`BroodMaturationDays >= 1`. Then it clamps `StartingFood` to `FoodCapacity`.

**Assumed** — the changed defaults above. Cost to change: none in code. The HUD shows 10 idle and
2 nursing at start instead of M1's 8 and 4.

### 0.5 M1 tests keep M1 rules

M2 changes the M1 numbers: hatching adds idle workers on day 1, upkeep includes brood, and tier 2
needs a chamber. The M1 tests isolate M1 behaviour with a test-only subclass:

```csharp
internal sealed class M1Config : SimConfig   // Tests/EditMode/SimTestKit.cs
{
    public M1Config()
    {
        StartingIdle = 8; StartingNursing = 4; StartingBrood = 0;
        QueenEggsPerDay = 0f; QueenNurses = 4;          // demand stays 4 with no brood
        NurseReserveFraction = 1f;                      // taps never take nurses
        StarveWorkerFractionPerDay = 0f; StarveMinDeathsPerDay = 0f; StarveBroodFractionPerDay = 0f;
        WinterUpkeepFactor = 1f; TierDownMargin = 0;
        FoodCapacityBase = 1e6f;                        // M1 had no store cap
        TierChambers = new[] { new ChamberCounts(), new ChamberCounts() };
    }
}
```

`SimTestKit.NewWorld` defaults to `new M1Config()`. In `ColonyUpkeepTests`, `RecruitmentTests`,
`HaulTests`, `TrailTests`, `CommandTests` and `ItemTests`, every `new SimConfig {…}` becomes
`new M1Config {…}`. The constructor runs before the initialiser, so each test's overrides
still win. `ClockTests` is untouched. With M1's `Defs()` (sugar cube only, no spawn data) the
spawner makes no Rng draws, so M1 determinism is unchanged. One exception:
`ColonyUpkeepTests.StartingCounts` moves to the new default config (§13).

`CommandTests.NotImplementedCommandsAreInert` drops `StartBuild`. `PickUp` and `Deposit` stay
`NotImplemented`: there is still no one-ant find.

---

## 1. Chambers

### 1.1 Kinds

```csharp
public enum ChamberKind { None = 0, Brood, Store, Processing }
public static class ChamberKinds { public const int Count = 4; }
public enum ChamberState { None = 0, Digging, Built }

[Serializable] public struct ChamberDef
{
    public ChamberKind Kind;
    public float FoodCost;       // paid when digging starts
    public float WorkerSeconds;  // digging work; BuildCrew workers do WorkerSeconds / BuildCrew s
    public int BroodCapacity;    // added when Built
    public float FoodCapacity;   // added when Built
}

[Serializable] public struct ChamberCounts { public int Brood, Store, Processing; }  // + ctor (b, s, p)

public struct Chamber
{
    public ChamberId Id;      // Index == panel slot
    public bool Active;       // Digging or Built
    public ChamberKind Kind;
    public ChamberState State;
}
```

| Kind | FoodCost | WorkerSeconds | Dig time with 4 | BroodCapacity | FoodCapacity | What it does |
|---|---|---|---|---|---|---|
| Brood | 30 | 1440 | 1 day | +12 | — | Room for brood. The queen stops laying when brood fills it. |
| Store | 20 | 1440 | 1 day | — | +150 | Room for food. Deliveries beyond capacity are lost. |
| Processing | 60 | 2160 | 1.5 days | — | — | Needed for tier 2 and up, and so for dead insects. |

`BroodCapacity = BroodCapacityBase + Σ BroodCapacity of Built chambers` (16 at start).
`FoodCapacity = FoodCapacityBase + Σ FoodCapacity of Built chambers` (200 at start). A chamber that
is digging counts for nothing. Both are derived by scanning the 10 chamber slots and are not stored.

The queen's own chamber is not a slot. The panel always draws it, and it holds the base
capacities.

**Assumed** — three kinds; the costs and capacities in the table. The starting nest has one brood
and one store chamber. Cheap to change: data. A fourth kind costs one enum value, one def and the
panel art.

### 1.2 Slots

`World` owns `Chamber[MaxChambers]`. The slot index is the chamber's fixed position on the
cutaway. `UnlockedSlots = SlotsByTier[Tier − 1]`: 4, 7, 10. A chamber can be dug only in an
unlocked, inactive slot. Starting chambers take slots 0, 1, … in `StartingChambers` order,
`State = Built`, `Gen = 1`.

If the tier drops, chambers in now-locked slots stay and keep working. They are just not offered
for new digging. M2 never removes a chamber (collapse is M3; that is what the gen is for).

**Assumed** — slots unlock by tier (4/7/10) and have fixed positions. Medium cost to change: the
panel art is drawn around the slot layout. The rule itself is one array.

### 1.3 Digging is a `Build` task

`TaskKind` gains `Build`; `TaskPhase` gains `Building` (both appended). `ColonyTask` gains:

```csharp
public ChamberId Chamber;   // Build only
public int Crew;            // workers digging; out of Idle, still population
public int ProgressTicks;   // worker-ticks done; Build is complete at RequiredTicks
```

`RequiredTicks(kind) = Mathf.RoundToInt(def.WorkerSeconds / TickSeconds)`. That is 14 400 for
Brood and Store, and 21 600 for Processing.

Diggers are counts, like nurses. They never become ant records. `World.Digging` = Σ `Crew` over
active build tasks (derived). **`Population = Idle + Nursing + Digging + AntsOutside`** (replaces
M1's definition). Diggers pay upkeep like any worker.

One build at a time. Haul-task code skips `Kind != Haul`. `TrailSystem`'s `users` count,
`CommandSystem.Complete`'s single-player-trail loop, `FindPlayerTask` and `Recount` are already
safe: a build task has no trail, no item and `PlayerLaid = false`. The engineer should check
each one anyway.

### 1.4 `TaskSystem.Step` (tick step 4), build part

After the M1 haul loop, for each active task in index order with `Kind == Build`:

1. If `Phase` is `Done` or `Aborted`: already freed in the M1 free pass; skip.
2. Resolve the chamber (defensive: if it fails, `Idle += Crew`, `Crew = 0`, `Phase = Aborted`; continue).
3. Top up: `add = Min(BuildCrew − Crew, Colony.Idle)`; `Crew += add`; `Idle −= add`.
4. `ProgressTicks += Crew`.
5. If `ProgressTicks >= RequiredTicks(kind)`: chamber `State = Built`; `Idle += Crew`; `Crew = 0`;
   `Phase = Done`; `Stats.ChambersBuilt++`; raise `ChamberBuilt(A = slot, B = kind)`.

The haul loop runs first, so hauling recruits get idle workers before the diggers' top-up.
Topping up matters only after starvation has killed a digger (§3.2). Digging is not slowed at
night or in winter, because it happens underground.

With 4 diggers a brood chamber started on tick `T` (crew taken in step 2, first progress in
step 4 of the same tick) completes on tick `T + 3599`.

### 1.5 Commands

`StartBuild` keeps its `CommandKind` value. `CancelBuild` is appended. New `RejectReason`s are
appended after `NotImplemented`: `NoSuchChamberKind, SlotLocked, SlotTaken, BuildInProgress,
NotEnoughFood, NoSuchBuild, NoTaskSlot`.

| Command | Arguments | Validation, in order (first failure wins) | Effect |
|---|---|---|---|
| `StartBuild` | `A` = `ChamberKind`, `B` = slot | not within `NestRadius` of the nest → `NotAtNest`; `A` not in `1..ChamberKinds.Count−1` → `NoSuchChamberKind`; `B` outside `0..UnlockedSlots−1` → `SlotLocked`; slot active → `SlotTaken`; a build task active → `BuildInProgress`; `Food <= def.FoodCost` → `NotEnoughFood`; `Idle == 0` → `NothingToSend`; no free task slot → `NoTaskSlot` | below |
| `CancelBuild` | `A` = slot | not at nest → `NotAtNest`; slot not `Digging` → `NoSuchBuild` | below |

**StartBuild effect:** `Food −= FoodCost`; `crew = Min(BuildCrew, Idle)`, `Idle −= crew`. The
chamber slot is taken (`Gen++`, `Active`, `Kind`, `State = Digging`). A task slot is taken:
`Kind = Build`, `Phase = Building`, `Chamber`, `Crew = crew`, `ProgressTicks = 0`. Raise
`BuildStarted(A = slot, B = kind)`.

The starting 20 food pays for no chamber: brood costs 30, processing 60, and a store's 20 would
leave nothing, which `<=` forbids. The first hauls pay for the nest, and that is intended.

`NotEnoughFood` uses `<=` on purpose. A build can never spend the stores to exactly zero, which
would start starvation without a `FoodRanOut` event.

**CancelBuild effect:** `Idle += Crew`; `Food = Min(FoodCapacity, Food + FoodCost)` (a full
refund; progress is lost); the chamber slot `Active = false`; task `Phase = Aborted`. Raise
`BuildCancelled(A = slot, B = kind)`. It raises no `TaskAborted`, whose copy is about hauls.

`World.CanBuild(kind, slot, playerPos)` runs the same validation without side effects and returns
the first `RejectReason` (or `None`). The panel uses it to grey out buttons (§12).

**Assumed** — one dig at a time; a crew of 4; a cancel refunds all the food and loses the digging.
Cheap to change.

---

## 2. Queen, brood and nursing

### 2.1 State

`ColonyState` keeps its M1 fields and gains:

```csharp
public int Shelter;          // 0..ShelterMax, from pinecones (§5.4)
public bool Dead;            // §6.3; absorbing
public bool QueenBlocked;    // the last egg due could not be laid: brood full
public float EggAccum;       // [0, 1]
public float BroodLossAccum; // >= 0
public float StarveAccum;    // >= 0
```

`World` owns `int[] BroodHatchIn`, length `BroodMaturationDays` (`M`). `BroodHatchIn[i]` is the
number of brood that hatch at the start of day `Clock.Day + 1 + i`. Eggs laid today go into
`[M − 1]`, so they hatch `M` day-starts later. `Colony.Brood` stays as a stored field and always
equals `Σ BroodHatchIn`; every change goes through both.

**Starting brood** is `StartingBrood`, clamped to `BroodCapacity`. It is dealt round-robin into
`BroodHatchIn[0 .. M−2]`, one per slot. With 4 that gives `[1,1,1,1,0,0,0]`: one worker hatches
at the start of each of days 1 to 4.

### 2.2 Hatching

In step 8, if `DayStartedThisTick`: `n = BroodHatchIn[0]`; shift the array left by one;
`BroodHatchIn[M−1] = 0`; `Brood −= n`; `Idle += n`; `Stats.WorkersRaised += n`. If `n > 0` raise
`BroodHatched(A = n)`.

Day 1 starts at tick 2 700 (the game starts at dawn of day 0). Day `d` starts at tick
`(d − 0.25) × 3600`.

### 2.3 Laying

In step 8, after upkeep (so `daily` is known, §3.1), and only if `!Dead`:

```
fed  = Food <= 0 ? 0 : (daily <= 0 ? 1 : Clamp01(Food / (QueenFedDays · daily)))
rate = QueenEggsPerDay · LayFactorBySeason[season] · fed            // eggs per day
if rate <= 0: EggAccum = 0                                          // winter, or no food: nothing owed
else: EggAccum += rate · dt / DayLengthSeconds
while EggAccum >= 1:
    if Brood >= BroodCapacity: QueenBlocked = true; EggAccum = 1; break
    BroodHatchIn[M−1]++; Brood++; Stats.EggsLaid++; EggAccum −= 1; QueenBlocked = false
```

When the rate is 0 (winter, or empty stores) `EggAccum` is cleared, so no egg that was nearly
due is laid later when room frees or laying resumes. While she is only blocked by a full brood
chamber the rate is above 0, `EggAccum` holds at 1, and she lays the moment a hatch frees room.

`BroodCapacityReached` is raised on the tick `QueenBlocked` turns from false to true. The flag
changes only on ticks when an egg is due, so it does not flicker.

At full rate an egg is laid every 72 s (720 ticks). Laying is fed by the stores. With 8 food a
day of upkeep (the start), the queen lays fully above 16 food and at a third of the rate at
5 food. This is the colony's main brake: growth slows as income stops covering upkeep.

**Assumed** — the queen's rate depends on stores measured in days of upkeep, and on the season.
The rate is 5 a day, halved in autumn, and zero in winter. Cheap to change: retune.

### 2.4 Nursing (settles SIM.md's Undecided)

```
NurseDemand(brood) = QueenNurses + ceil(brood / BroodPerNurse)        // 2 at start; 7 at 28 brood
```

**Rebalance** runs at the end of step 8 (and once in the constructor):
- if `Nursing > demand`: `Idle += Nursing − demand`; `Nursing = demand`
- else: `m = Min(demand − Nursing, Idle)`; `Idle −= m`; `Nursing += m`

Nursing comes first among in-nest duties. After step 8, either `Nursing == demand` or `Idle == 0`.
**Trail recruitment draws only from `Idle`** and never takes a nurse (as in M1). Digging also
draws only from `Idle`.

**Assumed** — nursing demand is `1 + ceil(brood / 5)`. Nurses are refilled from idle workers
before anything else, each tick. Only a tap at the nest can take nurses away (§2.5). Under-nursed
brood dies (§2.6). Cost to change: cheap in code. Medium in balance: the nursing share decides
how many idle workers the early game has. At 3 brood per nurse, idle drops to 2 by day 4, too
few to lift a sugar cube.

### 2.5 Taps may take nurses

`TapAnt` (SIM.md §4.5) becomes:

```
floor     = ceil(NurseDemand(Brood) · NurseReserveFraction)
pullable  = Max(0, Nursing − floor)
n = Min(TapBatch, Idle + pullable, def.AntsRequired − Assigned, free ant slots)
```

The first `Min(n, Idle)` ants come from `Idle` and the rest from `Nursing`. `AntSystem.Spawn` gains
`bool fromNursing`, which decrements `Nursing` instead of `Idle`. The Rng draws are unchanged.
If any came from nursing, raise `NursesPulled(A = count, B = task index)` after the
`AntLeftNest` events. Ants return to `Idle` as in M1, and the next rebalance moves them back
to nursing. `NothingToSend` now means `n == 0` with `Idle + pullable == 0`, or a full party.

This is the "big haul starves the brood" choice from GAME.md. It is deliberate: only a tap,
pressed at the nest when no idle worker is left, can do it. The nest prompt names it.

### 2.6 Brood loss

Brood dies in two ways. Both go through `BroodLossAccum`:

- **Starving** (`Food == 0`): `+= StarveBroodFractionPerDay · Brood · dt / Day`.
- **Under-nursed** (`Brood > 0 && Nursing < demand`, `coverage = Nursing / demand`):
  `+= UnderNursedBroodLossPerDay · (1 − coverage) · Brood · dt / Day`.

If neither applies this tick, `BroodLossAccum = 0`. Otherwise: while `BroodLossAccum >= 1 && Brood > 0`,
remove one brood from the highest non-zero index of `BroodHatchIn` (the youngest goes first),
`Brood−−`, `Stats.BroodLost++`, `BroodLossAccum −= 1`. If `Brood == 0`, `BroodLossAccum = 0`.
If any were lost, raise `BroodLost(A = count, B = cause)`, where cause is 1 = under-nursed and
2 = starving (starving wins if both).

Here is the price of a tap that takes nurses. With 15 brood and half the nurses gone, the
colony loses 15 brood a day: one every 24 s. A minute-long haul costs about 2–3 future workers,
against 60–120 food for the find. The trade is close enough to be a choice.

**Assumed** — under-nursed brood dies at `2 × shortfall × brood` per day, and the youngest goes
first. Cheap to change.

---

## 3. Upkeep, starvation, stores

### 3.1 Upkeep (replaces SIM.md §2.1 step 2)

```
winterFactor = 1 + (WinterUpkeepFactor − 1) · (1 − Shelter / ShelterMax)
seasonFactor = Season == Winter ? winterFactor : 1
daily  = UpkeepPerAntPerDay · Population · seasonFactor + BroodUpkeepPerDay · Brood    // food per day
before = Food; Food = Max(0, Food − daily · dt / DayLengthSeconds)                       // double
if before > 0 && Food == 0: raise FoodRanOut
```

At the start: 0.5 × 12 + 0.5 × 4 = 8 food a day. One brood costs 0.5 a day for 7 days, so 3.5
food raises a worker. After that the worker costs 0.5 a day for as long as it lives.

**Assumed** — brood costs the same as a worker, 0.5 a day. There is no separate queen upkeep. Cheap.

### 3.2 Starvation

Step 8, right after upkeep:

```
if Food == 0 and daily > 0:
    inNest = Idle + Nursing + Digging
    StarveAccum += Max(StarveMinDeathsPerDay, StarveWorkerFractionPerDay · inNest) · dt / Day
    while StarveAccum >= 1 and inNest > 0: kill one — Idle first, else Nursing, else one digger
        (Crew−− on the build task); StarveAccum −= 1; deaths++; inNest−−
    if inNest == 0: StarveAccum = Min(StarveAccum, 1)
    if deaths > 0: Stats.WorkersStarved += deaths; raise WorkersStarved(A = deaths)
else:
    StarveAccum = 0
```

Ants outside never starve. They are on a job and return to `Idle`. With 12 workers in the nest, the
first death comes 3 000 ticks (300 s) after the stores empty. A starving colony loses about 10% a
day, so it is down to 60% after 5 days.

**Assumed** — starvation kills about 10% of the workers in the nest per day (at least one), idle
workers first. The queen never dies in M2. Cheap to change; it sets the tone of a bad winter.

### 3.3 Stores

Food never exceeds `FoodCapacity`. Delivery caps it (§5.4) and a cancel refund caps it.
Nothing removes stored food except upkeep and build costs. Stored food does not spoil.

**Assumed** — stores cap; food delivered beyond the cap is lost and `StoresFull` says how much.
Cheap.

---

## 4. Colony tier (replaces SIM.md §2 `TierFor`)

Tier `t + 1` needs `Population >= TierThresholds[t − 1]` **and** built chambers
`>= TierChambers[t − 1]` (component-wise). With defaults:

| Tier | Name | Population | Built chambers | Slots |
|---|---|---|---|---|
| 1 | Young | — | — | 4 |
| 2 | Established | 25 | 1 processing | 7 |
| 3 | Mature | 75 | 1 processing, 2 brood, 2 store | 10 |

**Hysteresis**: a tier is kept until population falls below its threshold minus `TierDownMargin`
(tier 2 holds down to 20, tier 3 down to 70), or its chambers are no longer met.

```csharp
internal static int TierStep(int tier, int pop, ChamberCounts built, SimConfig cfg)
{
    int max = cfg.TierThresholds.Length + 1;
    while (tier < max && pop >= cfg.TierThresholds[tier - 1] && Meets(built, cfg.TierChambers[tier - 1])) tier++;
    while (tier > 1 && (pop < cfg.TierThresholds[tier - 2] - cfg.TierDownMargin
                        || !Meets(built, cfg.TierChambers[tier - 2]))) tier--;
    return tier;
}
```

The constructor uses `TierStep(1, …)`, so it only moves up. Step 8 uses `TierStep(Colony.Tier, …)`
and, on a change, raises `TierChanged(A = new, B = old)` (B is new; A unchanged from M1).

A find's `TierRequired` is still checked only at Mark (SIM.md §6.2). A haul already claimed
finishes if the tier drops.

**Assumed** — the tier needs chambers as in the table, and drops back only 5 below a threshold.
Cheap to change: config arrays.

---

## 5. Finds: spawning, expiry, spoilage, delivery

### 5.1 `ItemDef` additions

```csharp
public float SpawnSpring, SpawnSummer, SpawnAutumn, SpawnWinter; // mean finds per day
public int SpawnMax;            // live finds of this kind (Lying, Claimed, Hauling) at most
public float SpawnMinDistance;  // u from the nest
public float SpawnMaxDistance;  // u from the nest
public int StartCount;          // placed at construction
public float StartMaxDistance;  // u; start placements land no farther out; 0 = SpawnMaxDistance
public float LingerSeconds;     // a spawned find left Lying this long is gone; 0 = never
public int ShelterValue;        // added to Colony.Shelter on delivery
```

`WorldItem` gains `bool Expires`. It is true for finds placed by the spawner and false for
`SpawnItem` (scene setup and tests), so the tutorial's sugar cube waits for the player.

A kind is **spawnable** if `AntsRequired > 0` and any season rate or `StartCount` is above 0.
M1's test `Defs()` has none.

The five shipped defs (`Assets/Settings/Items/*.asset`; SugarCube updated, four new):

| Kind | Ants | Tier | Food | Spoil s | Linger s | Radius | Spr | Sum | Aut | Win | Max | Min u | Max u | Start | Shelter |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| SugarCube | 6 | 1 | 60 | 0 | 1080 | 1.2 | 0.15 | 0.3 | 0.15 | 0 | 1 | 40 | 110 | 0 | 0 |
| Seed | 2 | 1 | 15 | 0 | 720 | 0.4 | 2.5 | 2.5 | 4 | 0 | 10 | 10 | 90 | 2 | 0 |
| Leaf | 4 | 1 | 10 | 0 | 540 | 1.5 | 2 | 2 | 3 | 0 | 8 | 10 | 90 | 1 | 0 |
| DeadInsect | 10 | 2 | 120 | 1080 | 0 | 1.5 | 0.25 | 0.4 | 0.35 | 0 | 2 | 25 | 110 | 1 | 0 |
| Pinecone | 16 | 3 | 0 | 0 | 1440 | 3.0 | 0 | 0.1 | 0.4 | 0 | 2 | 50 | 110 | 0 | 1 |

The tutorial cube placed by the scene counts toward the sugar cube's `SpawnMax` of 1. No second
cube spawns while it lies there.

**Assumed** — this table: rates, lifetimes, distances, and radii for the new finds (a seed
0.4 u, a leaf and a beetle 1.5 u, a pinecone 3 u). Cheap to change: data. The meshes must match
the radii.

**Assumed** (M5) — seeds and leaves come in to 10 u (was 15) and up to 10 seeds and 8 leaves lie
at once (was 6 and 5), so more of what the garden offers is within reach; spawn *rates* are
unchanged, so the food offered per day (§6.1) is unchanged. Cheap to change: data.

**Assumed** (M5) — one dead insect is placed at construction (`StartCount 1`, min 25 u, was 0 and
40 u) so a Young colony sees a find it cannot yet take and reads the tier-2 prompt on it.
Spawning is not tier-gated; hauling is. It spoils from `Lying` after 3 days (`ItemSpoiled`), well
before any colony is Established (day ~8, §14.2), so it is a signpost, not food. Start placements of a kind use `StartMaxDistance` in place of `SpawnMaxDistance` when it is set
(> 0); DeadInsect's is 40, so the starting insect always lies 25–40 u out, inside the 45 u
antennae range, while spawned insects keep 25–110 u. Placing it adds draws at construction, so every
seed's post-construction RNG stream shifts: tests that compare same-seed worlds are unaffected;
any test that hard-codes positions or tick counts from a seeded start must be re-derived.

### 5.2 `SpawnSystem.Step` (tick step 7)

`World` owns `float[] SpawnProgress`, `float[] SpawnThreshold`, each length `ItemKinds.Count`.

```
for k in 1 .. ItemKinds.Count−1 (ascending):
    def = defs[k]; if not spawnable: continue
    rate = def.Spawn<Season>;            if rate <= 0: continue
    SpawnProgress[k] += rate · dt / DayLengthSeconds
    if SpawnProgress[k] < SpawnThreshold[k]: continue
    if LiveCount(k) >= def.SpawnMax or no free item slot:
        SpawnProgress[k] = SpawnThreshold[k]; continue      // no banking
    TryPlace(k, ref rng)                                    // may fail; the spawn is used up either way
    SpawnProgress[k] −= SpawnThreshold[k]
    SpawnThreshold[k] = rng.Range(1 − SpawnJitter, 1 + SpawnJitter)
```

`LiveCount(k)` = active items of kind `k` in `Lying`, `Claimed` or `Hauling`.

**`TryPlace(k)`**: up to `SpawnPlacementTries` times:
`a = rng.Range(0, 2π)`; `u = rng.Range(0, 1)`; `r = max − (max − min) · sqrt(u)`;
`p = NestPos + (cos a, sin a) · r`. Both draws are made on every try, so the RNG shape (two draws
per try) is the same as M2's uniform-ring law and only the values change. Reject `p` if
`!SpawnArea.Contains(p)` or it is within `SpawnMinSpacing` of the `Pos` of any active item not
`Delivered`/`Gone`. Otherwise take the lowest free item slot as `SpawnItem` does, set
`Expires = true`, and raise `ItemSpawned(A = index, B = kind, P = p)`. If all tries fail, nothing
spawns; with every kind at its `SpawnMax` this is about 2 in 10⁵ placements (Monte Carlo,
default patch), up from effectively never under the uniform ring, because the inner band is
denser.

**Near-biased radius (M5).** The radial density is linear, falling to zero at the outer edge:
`f(r) = 2 (max − r) / (max − min)²` on `[min, max]`, so a find is most likely just outside `min`
and never likelier farther out. Mean `r = min + (max − min)/3`; median
`r = max − (max − min)/√2`. Per unit *area* the density is ∝ `(max − r)/r`, so the inner band
is several times denser than the outer one. (The M2 law, `r = sqrt(Range(min², max²))`, was
uniform per unit area, which put most finds near the outer edge.)

| Kind | Min–max u | Mean r, M2 → M5 | Median, M5 | Within 45 u, M2 → M5 |
|---|---|---|---|---|
| Seed, Leaf | 15–90 → 10–90 | 61 → 37 | 33 | 23% → 68% |
| DeadInsect | 40–110 → 25–110 | 80 → 53 | 50 | 4% → 42% |
| SugarCube | 40–110 | 80 → 63 | 60 | 4% → 14% |
| Pinecone | 50–110 | 84 → 70 | 68 | 0 → 0 |

Figures are for the radius draw before rejection at the patch edge (`SpawnArea`, ±95 u), which
trims the far end of the 110 u kinds a little. The law applies to
every kind, so spawned sugar cubes and pinecones also come nearer, though their minimum keeps
them off the nest's doorstep.

**Construction**: after the colony is set up, for each spawnable `k` ascending, call
`TryPlace(k)` `StartCount` times (with `max = StartMaxDistance` when it is set), then `SpawnThreshold[k] = rng.Range(1 − j, 1 + j)` and
`SpawnProgress[k] = 0`. Construction raises no events (the queue is cleared at the start of
every tick). The Game layer scans `Items` after construction or load.

Mean interval between spawns of a kind is exactly `1 / rate` days. Spawns stop in winter because
every winter rate is 0, and progress resumes in spring where it stopped.

**Spawn area.** The constructor gains it:

```csharp
public World(SimConfig config, ItemDef[] itemDefs, Vector2 nestPos, Rect spawnArea, int seed)
public World(SimConfig config, ItemDef[] itemDefs, Vector2 nestPos, int seed)   // spawnArea = World.DefaultSpawnArea (−100, −100, 200, 200)
```

`SimRunner` passes the ground plane's bounds inset by 5 u: `(−95, −95, 190, 190)` for the
current patch.

**Assumed** — finds land in a ring around the nest, near-biased as above, anywhere in the patch
rectangle, stones included, since stones are walkable. Each interval is jittered by ±50%. Cheap
to change: one function, and tests compare same-seed worlds rather than golden positions.
Authored spawn points are the alternative if finds land somewhere silly.

### 5.3 Expiry and spoilage — `ItemSystem.Step` (tick step 6)

Order per item, extending SIM.md §1.2:
1. free `Delivered`/`Gone`; 2. `Age += dt`; 3. spoil (unchanged: `Lying`/`Claimed`,
`SpoilSeconds > 0`, `Age >= SpoilSeconds` → `Gone`, `ItemSpoiled`);
4. **expire**: `State == Lying && Expires && LingerSeconds > 0 && Age >= LingerSeconds`, and the
item is not `Draft.Item` while `Draft.Active` → `State = Gone`, raise
`ItemExpired(A = index, P = Pos)`; continue. 5. haul (unchanged).

A find you have marked never expires: a `Claimed` item is not `Lying`, and a find you are laying
a trail from is exempt. Only the dead insect spoils. Over its 3 days it loses value linearly
(M1 rule), so one found late and hauled slowly is worth little. A dead insect delivered
promptly averages about 65% of its value.

`Gone` items abort their task through the M1 `ItemGone` path. That only happens to a spoiled
`Claimed` insect.

### 5.4 Delivery (extends SIM.md §1.2 deliver)

```
value  = FoodValue · (1 − spoil01)
added  = Min(value, FoodCapacity − Food)      // in double; >= 0
Food  += added
Shelter = Min(ShelterMax, Shelter + def.ShelterValue)
Stats.FoodGathered += added; FindsHauled[kind]++
raise ItemDelivered(A = index, B = payload), TaskComplete
    // payload = shelter gained if def.ShelterValue > 0 (1 for a pinecone below ShelterMax, 0 at it),
    //           else round(added), the food stored
if value − added >= 0.5: raise StoresFull(A = round(value − added), B = index)
```

`ItemDelivered.B` is now the food actually stored (below the cap, the M1 value), or the shelter
gained for a shelter find.

**Assumed** — a pinecone brings no food. Each one hauled home adds 1 shelter, and 4 shelter cancel
winter's extra upkeep (§3.1). Shelter is permanent. Medium cost to change: it is the only reason
a pinecone exists. The alternative is a second resource for building, which GAME.md does not
have ("paid in food and worker-time").

---

## 6. The year

### 6.1 Seasons at the simulation level

| | Spring (days 0–9) | Summer (10–19) | Autumn (20–29) | Winter (30–39) |
|---|---|---|---|---|
| New finds | per-kind rates | most dead insects, first pinecones | most seeds and leaves, most pinecones | **none** |
| Queen laying | × 1 | × 1 | × 0.5 | × 0 |
| Worker upkeep | × 1 | × 1 | × 1 | × 1.25, less with shelter |
| Night | as M1 (0.6× speed) | | | |

Food available per game day if every find were taken (dead insect at 78 delivered):

| | All finds | Tier 1 only | Finds per day |
|---|---|---|---|
| Spring | 86 | 66.5 | 4.9 |
| Summer | 107 | 75.5 | 5.3 |
| Autumn | 126 | 99 | 7.9 |
| Winter | 0 | 0 | 0 |

**Assumed** — winter costs more, not less. Shelter softens it. Autumn laying is halved and winter
laying stops. Cheap to change: three config values. (Real colonies eat little in winter; if
species fidelity is chosen, revisit.)

### 6.2 The outcome

`YearDays = 4 · DaysPerSeason` (40). The year ends at the start of day 40, as winter breaks:
tick `(40 − 0.25) × 3600 = 143 100`, 3 h 58.5 min of play.

```csharp
public struct ColonyStats
{
    public double FoodGathered;
    public int EggsLaid, WorkersRaised, WorkersStarved, BroodLost, ChambersBuilt;
    public int PeakPopulation, PeakDay;
}
// World also owns int[] FindsHauled, length ItemKinds.Count.

public struct YearOutcome
{
    public bool Recorded, Survived;
    public int EndDay, Workers, Brood, Tier, Chambers /* built */, Shelter;
    public double Food;
    public ColonyStats Stats;      // snapshot
    public int FindsHauled;        // total over kinds
}
```

In step 8 (§7), on the tick when a day starts, if `Clock.Day == YearDays && !Outcome.Recorded`,
the outcome is snapshotted with `Survived = true`, and `YearEnded(A = Population,
B = floor(Food))` is raised. It is recorded once. The headline is **workers alive as winter
breaks**. Stores, peak and the rest are the supporting lines.

**Assumed** — the headline measure is the worker count at the start of day 40. There is no
composite score. Cheap to change: the record holds everything a score would need.

### 6.3 Death

At the end of step 8, if `!Dead && Population == 0 && Brood == 0`: `Dead = true`, raise
`ColonyDied(A = Day)`. If the outcome is not yet recorded, record it with `Survived = false`,
`EndDay = Day`. A dead colony lays no eggs and stays dead. Its stores are zero, because workers
only starve at zero food, and the player alone cannot haul.

**Assumed** — a colony is dead when it has no workers and no brood. The queen does not die in M2,
and M3 assumes she never does (SIM_M3.md §8, where the player ant's death is assumed too). What
her death would mean is left until the user decides it.

### 6.4 Sandbox

After day 40 nothing changes in the rules. `Season = (Day / DaysPerSeason) % 4` already cycles,
so day 40 is spring of year 2, with spawns, laying and upkeep by season. The outcome stays as
recorded, and death in the sandbox raises `ColonyDied` but does not change it. The HUD's
"Day {day}" needs a year-aware form after day 40 (UI copy, not sim).

---

## 7. `ColonySystem.Step` (tick step 8, replaces SIM.md §2.1)

1. **Hatch** (§2.2) if `DayStartedThisTick`.
2. **Upkeep** (§3.1) → `daily`; `FoodRanOut`.
3. **Starvation** (§3.2) and its brood term (§2.6).
4. **Under-nursing** brood term (§2.6), then apply brood loss; `BroodLost`.
5. **Laying** (§2.3); `BroodCapacityReached`.
6. **Nursing rebalance** (§2.4).
7. **Tier** (§4); `TierChanged`.
8. **Peak**: if `Population > Stats.PeakPopulation`: `PeakPopulation = Population`, `PeakDay = Day`.
9. **Death** (§6.3); `ColonyDied`.
10. **Year end** (§6.2); `YearEnded`.

Coverage in step 4 uses the nurses as they stand before the rebalance. A tap made this tick
therefore costs brood from the next tick on, and a returning ant fixes coverage in the same
step it is rebalanced.

---

## 8. Events

New kinds, **appended after `TierChanged`**:

| Kind | A | B | P | Raised in |
|---|---|---|---|---|
| `ItemSpawned` | item index | `ItemKind` | pos | 7 |
| `ItemExpired` | item index | | pos | 6 |
| `StoresFull` | food lost (rounded) | item index | | 6 |
| `BuildStarted` | slot | `ChamberKind` | | 2 |
| `BuildCancelled` | slot | `ChamberKind` | | 2 |
| `ChamberBuilt` | slot | `ChamberKind` | | 4 |
| `NursesPulled` | count | task index | | 2 |
| `BroodHatched` | count | | | 8 |
| `BroodLost` | count | 1 under-nursed, 2 starving | | 8 |
| `WorkersStarved` | count | | | 8 |
| `BroodCapacityReached` | | | | 8 |
| `ColonyDied` | day | | | 8 |
| `YearEnded` | workers | floor(food) | | 8 |

Changed: `TierChanged` gains `B = old tier`; `ItemDelivered.B` is the food stored, or for a find
with `ShelterValue > 0` the shelter gained (1 for a pinecone below the cap, 0 at it) (§5.4).
`SeasonChanged` is now raised in normal play (every 3 600 s).

---

## 9. `StateHash`

M1's fold (SIM.md §6.5) keeps its order with two in-section additions. Then the new sections
follow.

- Section 3 (**Items**), per active item, after `Trail`: `Expires`.
- Section 5 (**Tasks**), per active task, after `Haulers`: `Chamber` (index, gen), `Crew`, `ProgressTicks`.
- 7. **Colony M2**: `Shelter, Dead, QueenBlocked, EggAccum, BroodLossAccum, StarveAccum`, then
  `BroodHatchIn[0..M−1]`.
- 8. **Chambers**, every slot: `Gen`; if `Active`: `Kind, State`.
- 9. **Spawner**, `k = 0..ItemKinds.Count−1`: `SpawnProgress[k], SpawnThreshold[k]`.
- 10. **Stats**: `FoodGathered, EggsLaid, WorkersRaised, WorkersStarved, BroodLost, ChambersBuilt,
  PeakPopulation, PeakDay`, then `FindsHauled[0..Count−1]`.
- 11. **Outcome**: `Recorded`; if recorded: `Survived, EndDay, Workers, Brood, Tier, Chambers,
  Shelter, Food`, the stats snapshot in section-10 order (without the array), `FindsHauled`.

`NestPos` and `SpawnArea` are constants of a world, like the defs. They are saved, not hashed.

---

## 10. Saves

### 10.1 What is saved

Exactly the state hashed in §9 plus `Seed`, `Tick`, `Rng.State`, `NestPos` and `SpawnArea`. Not
saved: events (a save is taken between ticks, after the Game layer drains them); derived values
(`Cum`, `Assigned`, `AtTarget`, capacities, `Digging`, `DayStartedThisTick`); config and defs (the
build supplies them at load); the player's pose (the Game layer wraps the sim save with its own
fields — sim does not know the player ant's body).

Floats and doubles are stored as their **raw bits** (`int` from `SingleToInt32Bits`, `long` from
`DoubleToInt64Bits`), so a round trip is exact on every runtime and culture. The save is not meant
to be read by hand.

### 10.2 `SaveData` (Sim/Save, `[Serializable]`, public fields, JsonUtility-friendly)

```csharp
[Serializable] public sealed class SaveData
{
    public const int CurrentVersion = 1;
    public const int MinLoadableVersion = 1;

    public int Version;            // 0 when absent → Corrupt
    public string SavedBy;         // build id from the Game layer; informational
    public int Seed;
    public long Tick;
    public long RngState;          // ulong reinterpreted
    public int NestX, NestY;       // float bits
    public int AreaX, AreaY, AreaW, AreaH;   // float bits

    public ColonySave Colony;
    public int[] BroodHatchIn;     // index i hatches at the start of day (Day + 1 + i)
    public int[] SpawnProgress, SpawnThreshold;   // float bits, length ItemKinds.Count
    public StatsSave Stats;
    public int[] FindsHauled;
    public OutcomeSave Outcome;
    public DraftSave Draft;
    public ItemSave[] Items;       // every slot, index order
    public TrailSave[] Trails;
    public TaskSave[] Tasks;
    public AntSave[] Ants;
    public ChamberSave[] Chambers;
}

[Serializable] public struct ColonySave  { public int Idle, Nursing, Brood, Tier, Shelter; public long Food;
                                           public bool Dead, QueenBlocked; public int EggAccum, BroodLossAccum, StarveAccum; }
[Serializable] public struct StatsSave   { public long FoodGathered; public int EggsLaid, WorkersRaised, WorkersStarved,
                                           BroodLost, ChambersBuilt, PeakPopulation, PeakDay; }
[Serializable] public struct OutcomeSave { public bool Recorded, Survived; public int EndDay, Workers, Brood, Tier,
                                           Chambers, Shelter, FindsHauled; public long Food; public StatsSave Stats; }
[Serializable] public struct DraftSave   { public bool Active; public int ItemIndex, ItemGen, Count, Length; public int[] Points; }
[Serializable] public struct ItemSave    { public int Gen; public bool Active, Expires; public int Kind, State,
                                           PosX, PosY, S, PrevS, Age, TaskIndex, TaskGen, TrailIndex, TrailGen; }
[Serializable] public struct TrailSave   { public int Gen; public bool Active, PlayerLaid; public int Strength; public int[] Points; } // x,y bits interleaved, Count = Points.Length / 2
[Serializable] public struct TaskSave    { public int Gen; public bool Active, PlayerLaid; public int Kind, Phase, ItemIndex, ItemGen,
                                           TrailIndex, TrailGen, RecruitAccum, Taps, Haulers, ChamberIndex, ChamberGen, Crew, ProgressTicks; }
[Serializable] public struct AntSave     { public int Gen; public bool Active; public int Phase, TaskIndex, TaskGen, TrailIndex, TrailGen,
                                           S, PrevS, CarryIndex, CarryGen, Lane, WobblePhase, Timer, Slot; }
[Serializable] public struct ChamberSave { public int Gen; public bool Active; public int Kind, State; }
```

Inactive slots store only `Gen` (everything else default). The worst case is about 130 KB of
JSON (16 full trails dominate), well inside WebGL `PlayerPrefs`' 1 MB.

### 10.3 API

```csharp
public SaveData World.Save()                                            // between ticks only
public static LoadResult World.TryLoad(SaveData d, SimConfig cfg, ItemDef[] defs, out World world)

public static class SaveCodec
{
    public static string Encode(SaveData d);                            // JsonUtility.ToJson
    public static LoadResult Decode(string json, out SaveData d);       // parse, check version, migrate
    internal static LoadResult Decode(string json, int current, int minLoadable,
                                      IReadOnlyList<MigrationStep> steps, out SaveData d);   // for tests
}
public enum LoadResult { Ok, Migrated, Empty, Corrupt, TooOld, TooNew }
internal delegate void MigrationStep(SaveData d);
```

- `Rng` gains `internal static Rng FromState(ulong state)` (throws on 0). `SimClock` gains
  `internal static SimClock At(SimConfig cfg, long tick)` (sets `Tick`, derives the rest).
- `World` splits its constructor into `Allocate(cfg, defs)` and either `InitNew(seed)` (colony,
  starting chambers, starting finds, thresholds) or `Restore(save)` (no Rng draws: every field
  is overwritten). `TryLoad` calls `TaskSystem.Recount` at the end.
- `Trail` gains `Restore(ReadOnlySpan<Vector2> points)`: it copies points exactly and rebuilds
  `Cum` **without** merging near-duplicates. `Set` would merge them and change the state.
- **Decode**: whitespace or null → `Empty`. Unparseable → `Corrupt` (catch, never throw). Parse
  with `var d = new SaveData(); JsonUtility.FromJsonOverwrite(json, d);` so missing fields keep
  their C# initialisers. Then: `Version == 0` → `Corrupt`; `> CurrentVersion` → `TooNew`;
  `< MinLoadableVersion` → `TooOld`; `< CurrentVersion` → run `Steps[v − MinLoadableVersion]`
  for each version up to current, setting `Version` after each, and return `Migrated`.
  Otherwise `Ok`.
- **TryLoad validation** (`Corrupt` on any failure): arrays non-null; slot arrays no longer than
  `SimLimits` (shorter ones are padded with `Gen = 0` inactive slots); every active slot has
  `Gen >= 1`; every id an active record or the active draft holds is none or names a slot whose
`Gen` is at least the id's `Gen` (an id may be stale: an ant still depositing names its finished
task); the ids the systems follow without checking must resolve to active slots — an active
ant's trail, a carrier's item, a hauled item's trail, and a recruiting or hauling task's trail; every active
  item's kind has a def; every active trail has 2 to `MaxTrailPoints` points; `RngState != 0`;
  counts `>= 0`; `Brood == Σ BroodHatchIn`. **Brood array length mismatch** (config changed `M`)
  is not an error. Index `i` goes to `Min(i, M − 1)`; the total is preserved.
- Saved `NestPos` and `SpawnArea` win over the scene's. A mismatch is logged by the Game layer,
  not an error.

### 10.4 Version and migration policy

- `CurrentVersion` is bumped by any change to `SaveData`'s shape or meaning.
- **Additive change** (a new field whose initialiser is the right value for old saves): bump,
  and add a step to `SaveMigrations.Steps`. The step may be empty, but it is still written.
- **Changed meaning** (renamed, split, reinterpreted): bump, and add a step that converts the old
  fields. The old fields stay on the DTO, marked `// v1 only`, until `MinLoadableVersion` passes
  them.
- **Unconvertible**: raise `MinLoadableVersion`. The Game layer starts a new colony, keeps the
  old JSON once under a backup key, and says so.
- `TooNew` (an older cached build opened a newer save): the Game layer starts halted (the sim
  does not tick), offers to reload the page, and **never overwrites** the save. If the player
  chooses `StartNewColony` instead, that colony runs, but saving stays locked for the session:
  the newer save is still never written over.
- `Corrupt` (unparseable JSON or `Version == 0`): handled exactly like `TooOld` — new colony, old
  text kept once under the backup key — but reported with its own reason so the UI can word it.
  **Assumed**; cheap to change.
- **Colony death** (no workers, no brood) is final under the sim's rules; the Game layer offers
  `SimRunner.StartNewColony()`, which discards the save and rebuilds the World with a fresh seed.
  **Assumed**; this does not touch the open question about the player ant's or queen's death.
- **First-run hints** are Game-layer state persisted beside the save (a `HintsSeen` bitmask in
  PlayerPrefs) so they do not replay after a load.
- Retuning config needs no migration. A loaded save continues under the new numbers.
  Round-trip equality is only promised under the same config and defs.

**Assumed** — one save slot; autosave by the Game layer at each `DayStarted`, on focus loss,
pause and quit. Floats stored as bits. Older-than-loadable starts a new colony (as GAME.md states).
Cheap to change, except that the bit encoding makes saves unreadable by hand.

---

## 11. Scout ants: not in M2

M2 has no scout ants. Every find is marked by the player. GAME.md's premise is that the player's
movement is the order. Scouts that found and claimed finds on their own would earn food without
the player, so a neglected colony could survive. The economy in §14 is tuned on the player's
trips alone.

**Assumed** — no scouts in M2. Cost to add later: structurally cheap (a second source of haul
tasks with `PlayerLaid = false`, which TaskSystem already supports). The balance cost is medium:
the spawn rates in §5.1 would need retuning so scouts do not replace the player.

---

## 12. Panel contract

The panel opens when the player stands at the nest (`|PlayerPos − NestPos| <= NestRadius`, the
same test the sim applies to builds and taps). It reads the world and sends commands. It never
writes state.

### 12.1 Read API (no allocation)

```csharp
ReadOnlySpan<Chamber> Chambers;           // MaxChambers slots; index = panel position
int UnlockedSlots;                        // SlotsByTier[Tier − 1]
int SlotUnlockTier(int slot);             // smallest tier whose SlotsByTier exceeds slot
ChamberDef GetChamberDef(ChamberKind k);
ChamberCounts BuiltChambers;
bool TryGetBuild(out ColonyTask build);   // the active Build task, if any
int RequiredTicks(ChamberKind k);
RejectReason CanBuild(ChamberKind k, int slot, Vector2 playerPos);

ColonyState Colony;                       // Idle, Nursing, Brood, Food, Tier, Shelter, Dead, QueenBlocked
int Digging, Population, AntsOutside;   // Digging = Σ Crew of active build tasks
int TapPreview(int taskIndex, out int fromNursing);   // ants a tap would send now, and how many are nurses; 0 if refused for lack of ants or task not Recruiting
Rect SpawnArea; static readonly Rect World.DefaultSpawnArea;   // (−100, −100, 200, 200)
int NurseDemand;                          // NurseDemand(Colony.Brood)
int BroodCapacity; double FoodCapacity;
ReadOnlySpan<int> BroodHatchIn;           // brood by days to hatch
float EggsPerDay;                         // current laying rate, §2.3
double UpkeepPerDay;                      // `daily`, §3.1
double FoodDaysLeft;                      // Food / UpkeepPerDay
double WinterFoodNeed;                    // UpkeepPerAntPerDay · Population · winterFactor · DaysPerSeason
int TierThreshold(int tier); ChamberCounts TierChambersNeeded(int tier); int MaxTier;
ColonyStats Stats; int FindsHauled(ItemKind k); YearOutcome Outcome; int YearDays;
```

### 12.2 What the panel shows

- **Queen's chamber** (always): eggs per day now. Brood fill shown as Brood / BroodCapacity
  (`QueenBlocked` → "full", to prompt a brood chamber). Brood by days to hatch. Nurses as
  `Nursing / NurseDemand`, shown as a warning when short.
- **Built slot**: the kind and its contribution (+12 brood, +150 food, or "dead insects can be
  taken"). Brood and store chambers draw a fill level. The Game layer splits the totals across
  same-kind chambers in slot order, base capacity first. That is presentation, not state.
- **Digging slot**: the kind, `ProgressTicks / RequiredTicks`, crew, time left
  (`(RequiredTicks − ProgressTicks) · dt / Crew`, or "waiting for workers" at `Crew == 0`), and
  a Cancel button.
- **Empty unlocked slot**: one button per kind with its food cost and dig time with a full crew.
  Each is enabled when `CanBuild(...) == None`; otherwise the reason's copy is shown.
- **Locked slot**: "Tier n" from `SlotUnlockTier`.
- **Colony line**: workers; idle / on trails / nursing / digging; food / capacity; days of food;
  in autumn, food needed for winter against stores; shelter / 4; tier, and what the next tier
  still needs (population and missing chambers).

### 12.3 Commands

`StartBuild(A = (int)kind, B = slot)` and `CancelBuild(A = slot)` (§1.5). The rejections the
panel can meet are `NotAtNest`, `NoSuchChamberKind` (a bug; development builds only),
`SlotLocked`, `SlotTaken`, `BuildInProgress`, `NotEnoughFood`, `NothingToSend` (no idle workers),
`NoTaskSlot`, and `NoSuchBuild`.

**Assumed** — the panel issues build orders only; recruiting stays embodied (trail and tap). This
leaves GAME.md's Undecided on how much strategy game there is open. Adding panel orders later is
additive.

### 12.4 Copy needed (for `world-builder`, not written here)

Chamber names and one-line effects; the reject reasons above; toasts for `BuildStarted`,
`ChamberBuilt`, `BuildCancelled`, `BroodHatched`, `BroodLost` (both causes), `WorkersStarved`,
`BroodCapacityReached`, `NursesPulled`, `StoresFull`, `YearEnded`, `ColonyDied`; new item names
are already in UI_COPY. Also: `hud.nursing.value` changes from "{n} reserved" to a have/need form;
a `hud.digging` row; a nest tap prompt for when the tap will take nurses; a year-aware day label.
`ItemSpawned` and `ItemExpired` are silent (the find appears or fades).

### 12.5 Game-layer contract

- `SimRunner` builds `new World(cfg, defs, nestPos, spawnArea, seed)`, or `World.TryLoad` from the
  stored JSON. Either way it creates a `WorldItemView` for every active item by scanning `Items`,
  then on `ItemSpawned`. It removes a view when its slot goes inactive.
- The scene's finds (the tutorial cube) are placed with `SpawnItem` only for a new colony. After
  a load, a scene find is rebound to a saved item only if the save holds a `Lying`, non-expiring
  find of the same kind at the same position; otherwise its body is hidden.
- `Game/Saves/SaveSlot` owns the one slot in `PlayerPrefs` under three keys: `AntGame.Save` (the
  save), `AntGame.Save.Backup` (the last save discarded as `Corrupt` or `TooOld`, kept once) and
  `AntGame.HintsSeen` (first-run hints, which survive loads and new colonies).
- The stored save is a `GameSave` wrapper: the `SaveCodec` JSON as a string (it carries its own
  version), plus the player's position and yaw.
- Autosave on `DayStarted`, focus loss, pause and quit; never while saving is locked (§10.4).

---

## 13. Tests (`Assets/Tests/EditMode`)

`SimTestKit` gains `M1Config` (§0.5), `M2Defs()` (the §5.1 table), and `Rich()`: the default
config with `FoodCapacityBase = 100 000`, `StartingFood = 10 000`, so food never limits.
`Fast()` is the default config with `DayLengthSeconds = 36`: a day is 360 ticks and the year
ends at tick 14 310. Every per-day rate scales with it. Unless stated, a test uses the default
config and `Defs()` (no spawning).

### `ColonyUpkeepTests` (update)
- `StartingCounts` (default config): Idle 10, Nursing 2, Brood 4, Food 20, Tier 1, Population 12;
  slot 0 Brood Built, slot 1 Store Built, others inactive; `BroodCapacity 16`, `FoodCapacity 200`,
  `UnlockedSlots 4`, `BroodHatchIn == [1,1,1,1,0,0,0]`.
- All other tests: unchanged under `M1Config`.

### `BroodTests`
- `StartingBroodHatchesDays1To4`: `Rich()`, `QueenEggsPerDay 0`. `BroodHatched(A = 1)` on ticks
  2 700, 6 300, 9 900, 13 500 and no other; Idle + Nursing = 16 after.
- `FirstEggAt72Seconds`: `Rich()`. `Stats.EggsLaid == 1` first on tick 720 (±1).
- `EggsHatchAfterSevenDays`: `Rich()`, `StartingBrood 0`. `BroodHatched(A = 3)` on tick 24 300
  (the start of day 7; eggs laid on ticks ~720, 1 440, 2 160 of day 0).
- `NoEggsAtZeroFood`: `StartingFood 0`. 3 600 ticks → `EggsLaid == 0` and `EggAccum == 0`.
- `WinterClearsEggAccum`: `Fast()` with `Rich()`'s food values, `DaysPerSeason 1`. On the first
  winter tick `EggAccum == 0`, and the first egg of spring comes a full interval after spring
  starts.
- `FedFactorScalesRate`: config holds upkeep at 8 a day (`UpkeepPerAntPerDay = 2/3` × 12 workers,
  `BroodUpkeepPerDay 0`, `StartingBrood 0`, `BroodMaturationDays 20` so nothing hatches), and a
  hook sets `Food = 4` before every tick. Over 36 000 ticks: population 12, eggs per day
  1.25 (±0.1).
- `CapacityBlocksLaying`: `Rich()`. Brood never exceeds 16; exactly one `BroodCapacityReached`
  until a hatch frees room; `QueenBlocked` true while blocked.
- `SeasonFactors`: `Fast()` with `Rich()`'s food values, `DaysPerSeason 1`. Eggs on day 2 (autumn)
  are half of day 1 (summer) (±1); day 3 (winter) has none.
- `NurseDemandFollowsBrood`: demand 2, 2, 3, 5, 7 for brood 4, 5, 6, 16, 28. After any tick,
  `Nursing == demand || Idle == 0`.
- `UnderNursedLosesYoungestFirst`: `StartingIdle 0`, `StartingNursing 4`, `StartingBrood 15`
  (demand 4). Set `Nursing = 2` and `Idle = 0` via a hook. The first `BroodLost(B = 1)` comes
  240 ticks later (±1), and it is taken from the highest non-zero `BroodHatchIn` index.

### `TapNurseTests`
- `TapTakesNursesOnlyWhenIdleEmpty`: `StartingIdle 0`, `StartingNursing 4`, `StartingBrood 15`
  (floor 2). Lay a trail and tap at the nest → 2 `AntLeftNest`, `NursesPulled(A = 2)`,
  `Nursing == 2`.
- `TapPrefersIdle`: Idle 2, Nursing 4, floor 2 → one tap sends 3: Idle 0, Nursing 3.
- `TrailNeverTakesNurses`: `StartingIdle 0` → 600 ticks after `TrailLaid`, no `AntLeftNest`.
- `ReturningAntsRefillNursing`: after the tapped ants return, `Nursing == demand` on the
  `AntReturned` tick.

### `ChamberTests`
- `BuildBroodChamber`: `Rich()`, player at the nest, `StartBuild(Brood, 2)` on tick T → that tick
  `Food` −30, `Digging 4`, `BuildStarted(2, Brood)`; `ChamberBuilt(2, Brood)` on tick `T + 3599`;
  `Digging 0`; `BroodCapacity 28`; `Stats.ChambersBuilt 1`.
- `ProcessingTakesOneAndAHalfDays`: `ChamberBuilt` on tick `T + 5399`.
- `StartBuildRejections`: each case raises its reason and leaves `StateHash` equal to a control
  world ticked identically without the command. Cases: 10 u from the nest → `NotAtNest`; kind 0
  and kind 4 → `NoSuchChamberKind`; slot 4 at tier 1 → `SlotLocked`; slot 0 → `SlotTaken`; a
  second build → `BuildInProgress`; `Food == 30` for a brood chamber → `NotEnoughFood`;
  `Idle == 0` → `NothingToSend`.
- `CancelRefunds`: 100 ticks into a build, `CancelBuild(2)` → `BuildCancelled`, Food +30 (net of
  upkeep), Idle +4, slot inactive, task freed next tick. `CancelBuild(3)` → `NoSuchBuild`.
- `CrewToppedUpAfterStarvation`: kill a digger via a hook (Crew 3) with Idle 2 → next tick Crew 4.
- `UnlockedSlotsFollowTier`: tier 1 → 4; force tier 2 → 7; tier 3 → 10.

### `TierTests`
- `Tier2NeedsProcessing`: `Rich()`, `StartingIdle 30` → Tier 1. Build Processing → `TierChanged(2, 1)`
  on the `ChamberBuilt` tick.
- `Tier3NeedsTwoBroodTwoStore`: population 80 with processing → Tier 2 until a second brood and a
  second store chamber are built.
- `Hysteresis`: tier 2 at population 25. Via hooks to 21 → still 2; to 19 → `TierChanged(1, 2)`;
  back to 24 → still 1; 25 → 2.
- `ConstructionOnlyMovesUp`: no `TierChanged` at construction.

### `SpawnTests` (with `M2Defs()`)
- `StartingFinds`: after construction, 2 seeds, 1 leaf and 1 dead insect, all `Lying`, `Expires`,
  each within its kind's min–max distance of the nest, inside the area, and pairwise ≥ 8 u apart;
  the insect's `TierRequired` is above the starting tier.
- `NearBias`: seeds only, 10–90 u, `StartCount 5`, no spawn rates; 400 worlds (seeds 1–400), so
  2 000 placements at construction, five to a world so the spacing rule barely skews them → every
  `r` in `[min, max]`, finds in a world pairwise ≥ `SpawnMinSpacing` apart, mean `r` within ±3 u of
  `min + (max − min)/3` (36.7 u; the standard error is about 0.4 u, so ±3 u fails only on a wrong
  law), and two worlds from the same seed place every slot identically.
- `NoSpawnDefsNoDraws`: with `Defs()`, `Rng.State` after construction equals `new Rng(seed).State`.
- `MeanRate`: only seeds spawnable (`SpawnMax 64`, `LingerSeconds 36`), spring, 36 000 ticks →
  22–28 `ItemSpawned` for seeds.
- `WinterHasNoSpawns`: `Fast()`, `DaysPerSeason 1` → no `ItemSpawned` on day 3; `SpawnProgress`
  unchanged across it.
- `MaxConcurrent`: seeds `SpawnMax 2`, no linger → never more than 2 live seeds;
  `SpawnProgress[Seed] == SpawnThreshold[Seed]` while capped.
- `LyingFindsExpire`: a spawned seed → `ItemExpired` at `Age` 720 s (±1 tick); a `Claimed` seed
  of the same age does not expire; a seed that is the active draft's item does not expire.
- `SceneFindsNeverExpire`: `SpawnItem(Seed, …)` → `Expires == false`, still `Lying` after 10 000 ticks.

### `EconomyTests`
- `UpkeepIncludesBrood`: `M1Config` with `StartingBrood 4` placed and `QueenEggsPerDay 0` → food
  used over the first 2 700 ticks = (12 × 0.5 + 4 × 0.5) × 270/360 = 6.0 (±1e-3).
- `WinterUpkeepAndShelter`: `Fast()` with `Rich()`'s food values, `DaysPerSeason 1`,
  `StartingBrood 0`, `QueenEggsPerDay 0` (12 workers) → food used on day 3 = 7.5 (±1e-3); with `Shelter 2`, 6.75.
- `StarvationKillsIdleFirst`: `StartingFood 0`, `StartingBrood 0`, `QueenEggsPerDay 0` → first
  `WorkersStarved(1)` on tick 3 000 (±1); `Idle` dropped and `Nursing` did not.
- `StarvationFloor`: 5 workers in the nest → one death every 3 600 ticks (±1).
- `StoresCapDelivery`: `Food 190`, cap 200; deliver the cube → `ItemDelivered.B == 10`,
  `StoresFull(A = 50)`, `Food == 200`.
- `PineconeAddsShelter`: `StartingIdle 20`, pinecone def at tier 1 for the test → after delivery
  `Shelter 1`, `ItemDelivered.B == 0`.
- `NeglectDies`: default config, `M2Defs()`, no input → `ColonyDied` with `A` in 13–19;
  `Outcome.Recorded`, `!Survived`.
- `YearEndsOnceOnDay40`: `Fast()`, `M2Defs()`, a hook sets `Food = FoodCapacity` at every
  `DayStarted` → exactly one `YearEnded` on tick 14 310; `Outcome.Survived`. A further 720 ticks →
  no second `YearEnded`, `Season == Spring`.

### `DeterminismTests` (add)
- `M2_SameSeedIdentical`: default config, `M2Defs()`, seed 2024, 40 000 ticks. Script
  (`M2Script`): the first build order, `StartBuild(Brood, 2)`, goes at the first tick ≥ 100
  when the player is at the nest and `CanBuild` returns `None`. The start has 20 food and the
  chamber costs 30, so this waits for the first delivery. Marking cycles start at tick 200 and
  repeat every 1 200 ticks: take the lowest-index `Lying` item the tier allows, put the player
  2 u from it on its nest side, `MarkTrailStart`, walk home along M1's bent path, and tap once
  from the nest at cycle tick 400.
  Two worlds agree on `StateHash` every tick. At least 1 `ChamberBuilt` and at least 5
  `ItemDelivered`.
- `M2_DifferentSeedDiffers`: seed 2025 → the starting finds' positions differ.

### `SaveRoundTripTests` (the M2 determinism script, seed 11)
- `RoundTrip_HashEqual` at three save points: mid-draft (tick of `TrailMarkStarted` + 20),
  mid-haul (tick of `ItemPickedUp` + 50), mid-build (`BuildStarted` + 500) →
  `Save → Encode → Decode → TryLoad` gives `Ok` and an equal `StateHash`.
- `RoundTrip_Next1000TicksEqual`: from each save point, the original and loaded worlds tick 1 000
  more with identical input → equal `StateHash` on every tick and identical event sequences.
- `RoundTrip_JsonStable`: `Encode(Load(Decode(Encode(s))).Save()) == Encode(s)`, string-equal.
- `RoundTrip_AfterYearEnd`: `Fast()`, fed by hook, run to tick 15 000 → outcome survives the
  round trip field for field.
- `Load_RecountsDerived`: recount the original world (`TaskSystem.Recount`), then every task's
  `Assigned`/`AtTarget` equal the loaded world's, with at least one task that has ants.

### `SaveMigrationTests`
- `EmptyIsEmpty`: `""` and `null` → `Empty`.
- `GarbageIsCorrupt`: `"{not json"` → `Corrupt`, no exception.
- `MissingVersionIsCorrupt`: `"{}"` → `Corrupt`.
- `FutureVersionIsTooNew`: `Version = CurrentVersion + 1` → `TooNew`.
- `BelowMinIsTooOld`: the internal overload with `current 3, min 2` on a v1 save → `TooOld`.
- `StepsRunInOrder`: internal overload, `current 3, min 1`, two recording steps → step 0 then
  step 1, `Version == 3`, `Migrated`.
- `V1FixtureLoads`: the golden file `Assets/Tests/EditMode/Fixtures/save_v1.json` (written once
  by an `[Explicit]` maker test using a fixture config with literal time fields) → `Ok`,
  `StateHash` equal to the constant recorded in the test, then 100 ticks without exception.
  It stays forever; later versions must load it through their migrations.
- `BroodArrayRebuckets`: a save with a 9-long `BroodHatchIn` loaded with `M = 7` → indices 6–8
  summed into 6; `Brood` unchanged.
- `ShortSlotArraysPad`: `Ants` of length 128 → loads; slots 128–255 inactive, `Gen 0`.
- `DanglingIdIsCorrupt`: an active ant's `TaskGen` raised by 7, past any generation that slot has
  reached → `Corrupt`, no world.

---

## 14. Economy over a year

### 14.1 The model

This is an expected-value model at 1 s resolution, using exactly the rules above. Income is a
fraction η of the food the garden offers at the colony's tier (§6.1 table), with the dead insect
taken at 78 food (65% of fresh). Taps never take nurses. A player who builds digs one chamber at a
time: brood when the queen is blocked (up to 3), processing at 16 workers, then stores until
capacity covers 1.1 × winter's need. The sim itself is the real check. `DeterminismTests`' bot
script is a start, and a headless "bot player" balance run is a worthwhile M2 follow-up.

What η means in play. A player takes the best finds first: cubes and insects, then seeds. At
η = 0.6 that is about two finds per game day in spring and summer and three in autumn: one find
every two to three minutes. η = 0.8 is closer to four or five a day.

### 14.2 Reasonable player (η = 0.6, builds)

| End of day | Workers | Idle | Nursing | Brood | Food | Tier | Chambers B/S/P | What happens |
|---|---|---|---|---|---|---|---|---|
| 0 | 12 | 5 | 3 | 8 | 21 | 1 | 1/1/0 | First hauls. Brood chamber dug from tick ~2 400. |
| 2 | 14 | 9 | 5 | 16 | 76 | 1 | 2/1/0 | Starting brood hatching, one a day. |
| 4 | 16 | 6 | 6 | 24 | 60 | 1 | 2/1/0 | Processing chamber started (60 food). |
| 6 | 16 | 9 | 7 | 28 | 97 | 1 | 2/1/1 | Brood full at 28; queen blocked. |
| 8 | 25 | 14 | 7 | 28 | 109 | **2** | 2/1/1 | Day-1 eggs hatching ~4/day. Third brood chamber. |
| 10 | 35 | 24 | 7 | 28 | 146 | 2 | 3/1/1 | Summer. Dead insects open up. |
| 13 | 44 | 36 | 8 | 34 | 212 | 2 | 3/3/1 | Two stores dug. |
| 16 | 58 | 50 | 8 | 35 | 275 | 2 | 3/3/1 | |
| 20 | 78 | 66 | 8 | 33 | 316 | **3** | 3/3/1 | Mature. Pinecones open up. |
| 25 | 103 | 98 | 5 | 20 | 362 | 3 | 3/6/1 | Autumn laying halved. Stores to 950. |
| 29 | 116 | 111 | 5 | 17 | 408 | 3 | 3/6/1 | Shelter 2.4. Winter need ≈ 640. |
| 33 | 126 | 123 | 3 | 7 | 118 | 3 | 3/6/1 | Winter: no finds, no eggs. |
| 35 | 127 | 125 | 2 | 2 | 0 | 3 | 3/6/1 | Stores run out. |
| **39** | **86** | 85 | 1 | 0 | 0 | 3 | 3/6/1 | ~10% a day starve. **Year ends with 86.** |

Check of the winter arithmetic. At the end of autumn there are 116 workers and 17 brood, which
hatch in early winter to about 127. `winterFactor = 1 + 0.25 · (1 − 2.4/4) = 1.1`, so upkeep is
`0.5 · 127 · 1.1 ≈ 70` a day. 408 food lasts 5.8 days, and the remaining 4.2 days at ~10% loss
leave `127 · 0.9^4.2 ≈ 82–86`.

### 14.3 Other players

| Player | Peak (day) | Workers as winter breaks | Stores left |
|---|---|---|---|
| Keen, η = 0.8, builds | 133 (36) | **133** | 122 |
| Reasonable, η = 0.6, builds | 128 (34) | **86** | 0 |
| Measured, η = 0.6, only 2 brood chambers | 123 (36) | **94** | 0 |
| Casual, η = 0.45, builds | 108 (30) | **46** | 0 |
| Reasonable hauling, never builds | 78 (33) | **45** | 0 (tier 1 all year, 200 cap) |
| Neglect (no hauls) | 13 (2) | **dead on day ~16** | — |

**Neglect starves.** 20 food at 8 a day lasts 2.5 days. Then laying stops, brood is lost at 50% a
day, and workers die at about one a day (the floor). The colony is gone by day 16, mid-summer.

**Convergence.** Growth is negative feedback with a delay. Income above upkeep keeps the stores
over `QueenFedDays` of upkeep, so the queen lays fully. As population approaches income ÷ 0.5,
the stores fall and laying slows. The delay is the 7-day brood pipeline, so the overshoot is
bounded by one pipeline's worth: at most `BroodCapacity` extra mouths, 20 food a day at 40 brood.
The stores damp it. In spring and summer the reasonable colony is limited by brood capacity, not
food. Food is the limit in autumn, and winter is a forced decline paid for from stores. The
year's real decision is how much summer growth to convert into autumn stockpile. A colony that
grows on full stores until autumn is culled by winter. One that hauls hard in autumn, digs
stores and roofs itself with pinecones is not.

**SimConfig changes**: only `StartingIdle` 8 → 10 and `StartingNursing` 4 → 2 change existing
defaults (§0.4). `UpkeepPerAntPerDay` stays 0.5, so the M1 slice timeline is unchanged apart
from idle never dropping below 3 instead of 2 (10 idle at start, less the nurses that 2 new
eggs call for, less the 6 haulers).

---

## 15. Known degenerate strategies and failure modes

- **Workers beyond what hauls need are pure cost in M2.** About 30 workers cover every party; the
  rest are idle upkeep that count toward tier and the final number. M3 threats (ants lost on
  trails, raids) give them a use. Until then the optimal play is to grow late, not early.
- **The queen cannot be restrained.** The only brake the player controls is how many brood
  chambers to dig. A well-run colony can still lose a third of itself in winter (§14.2). If
  playtests find that unfair, the cheapest fix is a queen who lays in autumn only on food above
  `WinterFoodNeed`.
- **Taps that take nurses are worth it for insects.** Spending 2–3 brood for 80 food is usually a
  good trade. That is intended, but if it becomes routine, raise `UnderNursedBroodLossPerDay`.
- **Store chambers are cheap and capacity is the only thing they do.** A player can fill every
  late slot with stores. That is the correct winter play, and slots are what stop it.
- **Pinecones arrive late.** Tier 3 comes around day 20 for a reasonable player, and most
  pinecones fall in autumn. A slow colony never gets shelter, which compounds its winter.
  Intended pressure; watch it.
- **Spawn placement ignores obstacles.** A find can sit on a stone top or behind one, which is
  walkable, but may look odd. Authored spawn points are the fix if it does.
- **Stale claimed finds** (SIM.md §9) still never expire. A marked seed with too few idle workers
  holds its trail at the floor. Expiry deliberately does not apply to claimed finds.
- **Save under a retuned config** continues on the new numbers. That is intended, but a big
  retune can make an in-progress year easier or harder mid-run.
