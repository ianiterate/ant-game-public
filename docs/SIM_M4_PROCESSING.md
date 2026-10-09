# Simulation spec — M4 processing: the cutting room

Implementation spec for making processing a real step in `Assets/Scripts/Sim`. It builds on
[SIM.md](SIM.md), [SIM_M2.md](SIM_M2.md) and [SIM_M3.md](SIM_M3.md), which match the code, and
replaces only what it says it replaces. [GAME.md](GAME.md) is what the game is; this file is how
the simulation does it. The fiction is [NEST_VIEW_THEME.md](NEST_VIEW_THEME.md)'s cutting room.
Run `/open-questions` for everything marked below.

Units as before: distance in **u**, time in **s**, `dt` = 0.1 s, a day is 360 s (3 600 ticks).

**In one paragraph.** A dead insect hauled home no longer goes into the stores. It is dragged
whole into the processing chamber as a **carcass** and waits there as raw food. Two workers per
processing chamber, drawn from idle like nurses, cut it at the joints in four steps over half a
day; each step puts a quarter of its food into the stores. A carcass not cut within two days of
delivery rots, and what is left of it goes to the midden. The gate is unchanged: a dead insect
still needs an Established colony, and so a processing chamber, before it can be marked. The
delay is new. At these numbers it costs a well-played colony nothing measurable at the year's end
(§12); what it adds is a step you can see, that needs hands, and that can fail when the nest has
none to spare.

---

## 0. What changes

| Area | Before | After |
|---|---|---|
| Dead insect delivered | `Food += value` (capped) | A carcass of `value` raw food enters the cutting room (§2.1) |
| Processing chamber | Tier-2 requirement only | Also where carcasses are cut, one at a time each (§2.3) |
| Workers | Idle, Nursing, Digging, outside | + **Processing** (cutters), drawn from Idle after nurses (§3) |
| `Population` | `Idle + Nursing + Digging + AntsOutside` | `+ Processing` |
| `InNest` (raid defenders, respawn) | `Idle + Nursing + Digging` | `+ Processing` |
| Kill order in the nest (starvation, raids, respawn) | idle, nursing, digger | idle, **processing**, nursing, digger |
| Raids | Loot `Food` | Loot `Food` only; carcasses are never taken (§3.4) |
| Save | v2 | **v3** (§8) |
| Nest view processing | stand-in jobs (removed with the view hookup) from the event recorder | Reads real carcasses from the snapshot (§9) |

Unchanged: the tier gate, the starting dead insect (it still rots in the garden before any colony
can take it), field spoilage, every spawn rate, every existing `SimConfig` default, and the RNG:
processing makes **no random draws**, so a world that never delivers a dead insect behaves
bit-for-bit as before (only its hash value differs, because the hash folds the new fields).

---

## 1. What goes raw

`ItemDef` gains one field:

```csharp
/// <summary>Delivered whole to the cutting room as a carcass, not into the stores (SIM_M4_PROCESSING.md §2).</summary>
public bool Raw;
```

Shipped data: `DeadInsect.Raw = true`; every other kind `false`.

**Assumed** — dead insects only. Seeds, leaves and sugar come in store-sized, as the cutting-room
fiction says. Husking seeds was considered and not taken: seeds are the Young colony's staple, and
a Young colony has no processing chamber, so raw seeds would either rot or force a 60-food chamber
before the first seed counts, which breaks the opening (the first hauls pay for the nest). A 15-food
seed cut in steps is also bookkeeping without a decision in it. Cost to change: data (`Raw = true`
on the seed def) plus a rule for raw finds at tier 1 — either the processing chamber moves to
tier 1 or seeds husk without a chamber. Medium: the spring economy (SIM_M2.md §14.2, days 0–8) would
need redoing, because seeds are most of it.

---

## 2. The cutting room

### 2.1 Carcasses

```csharp
// Sim/Colony/Carcass.cs
public struct Carcass
{
    public float Value;         // food it holds whole: FoodValue × (1 − spoil01) at delivery; fixed after
    public int Progress;        // worker-ticks of cutting done, 0..RequiredCutTicks
    public int Chamber;         // slot of the processing chamber cutting it; −1 while waiting
    public long DeliveredTick;  // it rots at DeliveredTick + RotTicks
}
```

`World` owns `Carcass[SimLimits.MaxCarcasses]` and `int CarcassCount`, kept **packed, oldest
first**: index 0 is the earliest delivery. Removing one shifts the rest down (at most 3 copies, no
allocation). Index order is delivery order, so every rule below that says "oldest" is index order.

Derived (never stored):

```
StepTicks        = Max(1, RoundToInt(CutWorkerSeconds / CutSteps / dt))     // 900
RequiredCutTicks = CutSteps · StepTicks                                       // 3 600 worker-ticks
StepsDone(c)     = c.Progress / StepTicks                                     // 0..4
Remaining(c)     = c.Value · (CutSteps − StepsDone(c)) / CutSteps             // raw food left in it
RawFood          = Σ Remaining(c)                                             // the raw pool, a double
RotTicks         = RoundToInt(CarcassRotDays · DayLengthSeconds / dt)         // 7 200
```

The "raw food pool" is real state, held as up to four carcass records rather than one number,
because each carcass rots on its own clock and is cut in its own steps. `RawFood` is its sum.

**Assumed** — once delivered, a carcass keeps the value it arrived with; the garden's linear
spoilage stops at the door, and the nest's own clock is a hard deadline (§2.4). Cheap to change.

### 2.2 Delivery (extends SIM_M2.md §5.4)

In `ItemSystem.StepHaul`, on delivery, when `def.Raw`:

```
value = FoodValue · (1 − spoil01)                      // as before
if CarcassCount < MaxCarcasses:
    _carcasses[CarcassCount++] = { Value = value, Progress = 0, Chamber = −1, DeliveredTick = Tick }
    payload = round(value)
else:                                                  // the cutting room is full
    Stats.CarcassesRotted++; Stats.FoodRotted += value; payload = 0
    raise CarcassRotted(A = round(value), B = 2) after ItemDelivered
FindsHauled[kind]++                                    // as before: the haul counts on delivery
raise ItemDelivered(A = index, B = payload), TaskComplete
```

No `Food` change, no `FoodGathered`, no `StoresFull` at delivery for a raw find: those happen as
it is cut (§2.3). `ItemDelivered.B` for a raw find is the raw food taken in, following the shelter
precedent (B means what the find put into the nest).

A full cutting room needs four uncut carcasses at once. With at most two dead insects in the
garden (`SpawnMax 2`) and a carcass cut in half a day, that does not happen in play; the rule is
there so the array has an answer.

### 2.3 Cutting — `ColonySystem` step 6b

`ColonySystem.Step` gains a sub-step between nursing rebalance (6) and tier (7):

```
1. Rot.    for i ascending: if Tick − c.DeliveredTick >= RotTicks:
               lost = Remaining(c); Stats.CarcassesRotted++; Stats.FoodRotted += lost
               raise CarcassRotted(A = round(lost), B = 1); remove i; i−−
2. Assign. for each Built Processing chamber, slot ascending, that no carcass has as Chamber:
               the oldest carcass with Chamber == −1 (if any) gets Chamber = slot
3. Staff.  demand = ProcessingCrew · (carcasses with Chamber >= 0)
           if Processing > demand: Idle += Processing − demand; Processing = demand
           else: m = Min(demand − Processing, Idle); Idle −= m; Processing += m
4. Cut.    left = Processing
           for i ascending with Chamber >= 0:
               crew = Min(ProcessingCrew, left); left −= crew
               before = StepsDone(c); c.Progress = Min(RequiredCutTicks, c.Progress + crew)
               for step = before+1 .. StepsDone(c):
                   q = c.Value / CutSteps                                   // double
                   added = Min(q, FoodCapacity − Food); Food += added; Stats.FoodGathered += added
                   raise CarcassCut(A = c.Chamber, B = step)
                   if q − added >= 0.5: raise StoresFull(A = round(q − added), B = −1)
           then for i ascending: if c.Progress == RequiredCutTicks:
               Stats.CarcassesCut++; Idle += its crew; Processing −= its crew; remove i; i−−
```

`ProcessingDemand` (read API) is `demand` from step 3.

Consequences engineers rely on:

- A carcass delivered on tick `T` (ItemSystem, step 7) is assigned and staffed in step 9 of the
  same tick. With 2 cutters, step 1 completes on tick `T + 449` and the carcass on `T + 1799`
  (180 s). With 1 cutter, `T + 899` and `T + 3599`.
- Nursing is rebalanced first, so nurses win a short Idle. Haul recruiting and the build top-up run
  in step 5, before this, so they also come first in a tick; cutters once assigned are kept,
  exactly as nurses are.
- Each step puts a quarter of the carcass's value into the stores, capped by `FoodCapacity` as a
  delivery is. Food that does not fit is lost at the door (`StoresFull`, `B = −1` because no item
  is involved).
- Cutting goes on at night, in rain and in winter: it is underground, like digging.
- With no built processing chamber nothing is assigned and the carcass rots. It cannot arise in
  play (a dead insect needs tier 2 at Mark, tier 2 needs the chamber, and chambers are never
  removed); it is defined so a hook or a save cannot wedge the room.
- More than one processing chamber cuts more than one carcass at a time, each with its own crew.

**Assumed** — `ProcessingCrew 2` per chamber, `CutWorkerSeconds 360` (half a day with a full crew,
45 s a step), four steps of a quarter each. Cheap: data. §12 shows what slower cutting would do.

**Assumed** — food from a cut step that does not fit the stores is lost, as at delivery, rather
than cutting pausing until there is room. Pausing would let raw food sit as storage beyond the
stores' room until it rots, which is a store the player does not control. Cheap to change.

### 2.4 Rot

A carcass rots `CarcassRotDays` (2) after delivery, wherever its cutting stands. What is left
(`Remaining`) is lost and counted in `Stats.FoodRotted`; the carcass goes to the midden. Steps
already cut stay in the stores.

With a full crew a carcass is done in half a day, so rot needs the room to go without cutters for
about a day and a half: no idle workers at all, which means a starving, breached or tap-drained
nest.

**Assumed** — 2 days, a hard deadline from delivery, and the remainder lost at once. Cheap: data.

---

## 3. Cutters

### 3.1 State

`ColonyState` gains `public int Processing;` — workers cutting, in the nest, out of Idle.

- `Population = Idle + Nursing + Digging + Processing + AntsOutside` (replaces SIM_M2.md §1.3).
- `InNest = Idle + Nursing + Digging + Processing` (replaces SIM_M3.md §7.3's definition).
- Cutters pay upkeep, count toward tier and the year's headline, and fight in a raid.

### 3.2 Kill order

One helper, `KillOneInNest`, used by starvation (SIM_M2.md §3.2), the raid fight (SIM_M3.md §7.4)
and the respawn's worker (SIM_M3.md §8.3): **idle, else processing, else nursing, else a digger.**
A cutter killed lowers `Processing`; step 6b refills it from Idle next tick if it can.

**Assumed** — cutters die before nurses. Losing a cutter costs time on food that keeps for two
days; losing a nurse costs brood. Cheap.

### 3.3 Taps and trails

Trails never take cutters. A tap at the nest never takes them either: it takes idle, then nurses
down to their floor (SIM_M2.md §2.5), as today. A field tap takes ants outside only.

**Assumed** — cutters are as protected as nurses, more so for taps. A tap that could empty the
cutting room would make "cutting" a resource the player manages by hand, which the embodied
controls (GAME.md, Settled) do not offer. Cheap.

### 3.4 Raids

Raiders loot `Food` only. Carcasses are never looted, and the cutters fight like any worker in the
nest.

**Assumed** — raw food is not lootable: a carcass is too big to carry off, which is the cutting
room's whole reason to exist. The effect is small (§12: about 2 food a year). Cheap to change: one
term in the loot rule.

---

## 4. `SimConfig` and `SimLimits`

| Field | Default | Meaning |
|---|---|---|
| `ProcessingCrew` | `2` | Cutters per carcass, so per built processing chamber. ≥ 1. |
| `CutWorkerSeconds` | `360f` | Worker-seconds to cut one carcass (180 s with 2). > 0. |
| `CarcassRotDays` | `2f` | Days after delivery that an uncut carcass rots. > 0. |

```csharp
public const int MaxCarcasses = 4;   // the cutting room's slots; not balance
public const int CutSteps = 4;       // legs, head, shell, soft parts; the view draws four
```

The constructor validates the three fields and throws `ArgumentException` on a bad one. No
existing default changes.

---

## 5. Events (appended after `AntRedirected`)

| Kind | A | B | P | Raised in |
|---|---|---|---|---|
| `CarcassCut` | chamber slot | step 1–4 (4 = done; the shell goes to the midden) | nest | 9 (6b) |
| `CarcassRotted` | food lost (rounded) | 1 rotted, 2 the cutting room was full | nest | 7 (cause 2), 9 (cause 1) |

Changed: `ItemDelivered.B` for a `Raw` find is the raw food taken in (0 when the room was full).
`StoresFull.B` is `−1` when the food came from a cut step; the Game layer must not index `Items`
with it.

---

## 6. Stats

`ColonyStats` gains, after `FoodLooted`: `int CarcassesCut, CarcassesRotted; double FoodRotted`.
`FoodGathered` keeps its meaning, food actually stored, so a dead insect now adds to it as it is
cut, not on delivery. `FindsHauled[DeadInsect]` still counts on delivery.

---

## 7. Read API (no allocation)

```csharp
ReadOnlySpan<Carcass> Carcasses;      // CarcassCount entries, oldest first
double RawFood;                       // Σ Remaining
int ProcessingDemand;                 // ProcessingCrew × carcasses assigned to a chamber
int CarcassCrew(int i);               // cutters on carcass i now (the step 6b split); 0 if waiting
int CutStep(int i);                   // the cut under way, 0..3
float CutStep01(int i);               // progress through that cut
float Cut01(int i);                   // Progress / RequiredCutTicks
double CarcassDaysLeft(int i);        // days until it rots
int RequiredCutTicks, StepTicks;
// Colony.Processing is the cutter count. Population and InNest include it.
```

---

## 8. Saves: version 3

### 8.1 Shape

```csharp
public const int CurrentVersion = 3;      // MinLoadableVersion stays 1
public CarcassSave[] Carcasses;           // packed, oldest first; null in a v1/v2 save

[Serializable] public struct CarcassSave { public int Value; /* float bits */ public int Progress, Chamber; public long DeliveredTick; }
// ColonySave gains: public int Processing;
// StatsSave gains:  public int CarcassesCut, CarcassesRotted; public long FoodRotted;   // double bits
```

### 8.2 Migration v2 → v3

`SaveMigrations.Steps` gains `V2ToV3`: `d.Carcasses ??= new CarcassSave[0];`. Every other new field
defaults to the right value for an old save (`Processing 0`, the three stats 0). A v1 save runs
`V1ToV2` then `V2ToV3`.

A v2 colony continues with an empty cutting room. Any insect it had already delivered is in its
stores as food, as the old rule said; one still being hauled when it was saved goes to the cutting
room when it arrives.

### 8.3 Validation additions (`Corrupt` on failure)

`Carcasses` no longer than `MaxCarcasses`; `Processing >= 0`; each carcass `Value` finite and ≥ 0,
`Progress >= 0`, `0 <= DeliveredTick <= Tick`, `Chamber` either −1 or the slot of an active,
Built processing chamber, and no two carcasses share a chamber. `Progress` above the current
`RequiredCutTicks` is not an error (a retuned config); that carcass finishes on the next tick.
`Processing` above the demand is not an error; step 6b returns the surplus to Idle.

**Assumed** — a carcass whose `Progress` is past a retuned `RequiredCutTicks` finishes without
paying out the quarters the new tuning counts as already cut (no `CarcassCut` events for them).
Only a config change between save and load can cause it. Cheap to change: pay the missing quarters
in the finishing tick.

### 8.4 Fixtures

- `save_v1.json` and `save_v2.json` stay forever and now return `Migrated`. Their recorded hash
  constants (`V1FixtureHash`, `V2FixtureHash`) are **re-recorded once**, because the hash now
  folds §10's fields. Each still asserts every older field equal to the decoded fixture, plus an
  empty cutting room and `Processing == 0`.
- New golden `save_v3.json` from an `[Explicit]` maker: the defending bot (`M3Script.DefendingBot()`,
  default config) on seed 1 until a carcass is part-cut (`CutStep(0) >= 1`) with cutters at work, or
  `Assert.Inconclusive` by day 25. `V3FixtureLoads` → `Ok`, its recorded hash, at least one carcass
  with `Progress > 0`, `Processing > 0`, and 100 ticks without exception. **Assumed** — seed 1; take
  the lowest seed that reaches the point if it does not.

---

## 9. View contract

### 9.1 `NestSnapshot` additions

| Field | Source | Notes |
|---|---|---|
| `Processing`, `ProcessingDemand` | `Colony.Processing`, `ProcessingDemand` | Cutter bodies; short when `Processing < ProcessingDemand` |
| `RawFood` | `RawFood` | The total still to cut |
| `CarcassCount` | `Carcasses.Length` | 0–4 |
| `Carcass[4]`: `Value`, `ChamberSlot`, `Step` (0–3), `Step01`, `Cut01`, `Crew`, `DaysLeft` | `Carcasses[i]`, `CutStep`, `CutStep01`, `Cut01`, `CarcassCrew`, `CarcassDaysLeft` | `ChamberSlot −1`: waiting by the room's door. The carcass is drawn in `ChamberSlot`'s pocket at the stage `Step` shows: 0 whole on its back, 1 legs off, 2 head off, 3 shell pried, soft parts going |
| Counter deltas `CarcassesCut`, `CarcassesRotted` | `World.Stats` | Shell plates and rotted carcasses for the midden, with the same baseline reset on open as the other deltas |

Stage progress is extrapolated between snapshots with `SimRunner.Alpha` at `CarcassCrew / StepTicks`
per tick, like the dig's progress.

### 9.2 `NestActivity` changes

- **Remove** the processing jobs (`ProcessingCapacity`, `ProcessingSeconds`, `TryGetProcessing`):
  the snapshot is now the truth. `NestViewTests.DeliveriesKeepTheLastEightAndQueueTheCuttingRoom`
  drops its cutting-room half.
- Keep `ItemDelivered` of a dead insect as the moment it is dragged in.
- `CarcassCut`: one cutter carries pieces from slot `A` to the first store pile with room (3 s);
  on `B = 4` a second carries the shell plates up to the midden.
- `CarcassRotted`: B = 1, the carcass is dragged whole from its place to the midden; B = 2, it is
  dragged straight there from the shaft.
- Midden sources over 2 game days: the existing losses, plus shell plates (`CarcassCut` B = 4) and
  rotted carcasses.

### 9.3 Body allocation (replaces NEST_VIEW.md §2.2's processing row)

| Role | Shown | Cap |
|---|---|---|
| Cutters | `CarcassCrew(i)` at each carcass in a chamber | 4 |
| Floor scrapers | 2 from `Idle`, only in a processing chamber with no carcass, only while `Idle >= 2` | 2 |

### 9.4 State → visual (replaces NEST_VIEW.md §2.3's processing row)

| Subject | Trigger | Visual |
|---|---|---|
| Cutting | a carcass with `ChamberSlot >= 0`, `Crew > 0` | On its back in the chamber at stage `Step`; cutters at the joint; a piece carried out at each step |
| Waiting | `ChamberSlot == −1` | Whole, by the processing chamber's door |
| No hands | `ChamberSlot >= 0`, `Crew == 0` | Untouched, no cutters; label `nestview.processing.nohands` |
| Going off | `DaysLeft < 0.5` | Darkened, sheen dulled; label `nestview.processing.rotting` |
| Rotted | `CarcassRotted` | Dragged whole to the midden |

---

## 10. `StateHash`

Earlier sections keep their order. `ColonyStats` folds `CarcassesCut, CarcassesRotted`, then
`FoodRotted` (double) after `FoodLooted`, so section 10 and the outcome's snapshot carry them. New
section after 17:

18. **Processing**: `Colony.Processing`, `CarcassCount`, then per carcass in index order: `Value,
    Progress, Chamber, DeliveredTick` (low, high).

---

## 11. Tests (`Assets/Tests/EditMode`)

`SimTestKit` gains a hook `World.AddCarcassForTest(float value)` (appends at the current tick) and
`Established()`: `Calm()` with `StartingIdle 30` and `StartingChambers {Brood, Store, Processing}`, so
tier 2 at construction. `M2Defs()` sets `Raw` on the dead insect.

### `ProcessingTests` (new; `Established()`, `UpkeepPerAntPerDay 0`, `BroodUpkeepPerDay 0` unless stated)
- `InsectGoesRaw`: haul a fresh dead insect home → on the delivery tick `Food` unchanged, one
  carcass with `Value` = the delivered value, `ItemDelivered.B == round(value)`,
  `FindsHauled(DeadInsect) == 1`, `FoodGathered` unchanged.
- `CutInFourQuarters`: `AddCarcassForTest(80)` between ticks; on the next tick, T, `Processing 2`;
  `CarcassCut(A = slot, B = 1..4)` on ticks T+449, T+899, T+1349, T+1799; Food +20 at each;
  `CarcassesCut 1`; `Processing 0` and Idle restored on T+1799.
- `OneCutterTakesTwiceAsLong`: `StartingIdle` such that Idle is 1 after nursing → steps every 900 ticks.
- `NursesBeforeCutters`: Idle 1 short of nurse demand + a carcass → no cutters until a nurse's need is met.
- `RotsAfterTwoDays`: Idle 0 → `CarcassRotted(A = 80, B = 1)` on T+7200; `FoodRotted 80`; `RawFood 0`.
- `PartCutRotsRemainder`: one step cut, then cutters removed by hook → rot loses 60; Food kept its 20.
- `RoomFullGoesToMidden`: 4 carcasses + a delivery → `ItemDelivered.B == 0`, `CarcassRotted(B = 2)`.
- `StepOverflowIsLost`: Food = cap − 5, a step of 20 → Food at cap, `StoresFull(A = 15, B = −1)`.
- `TwoChambersCutTwo`: 2 processing chambers, 2 carcasses → `Processing 4`, both progress, each in its own slot.
- `WaitingCarcassKeepsItsChamberOrder`: with 1 chamber the second carcass has `Chamber −1` until the first is done.
- `PopulationAndInNestCountCutters`.
- `StarvationKillsIdleThenCutters`; `RaidKillsIdleThenCutters` (Hot raid via hook).
- `RaidLootsFoodOnly`: a breach with a carcass in the room → `RawFood` unchanged, `Food` falls.
- `TapNeverTakesCutters`: Idle 0, cutters 2, nurses above floor → a tap takes nurses only.
- `CuttingGoesOnInWinter`.
- `NoProcessingChamberRots`: by hook, a carcass with no chamber built → never assigned, rots.

### `DeterminismTests`
- `HashCoversProcessing`: each alone via hooks → hash changes: `Colony.Processing`, a carcass's
  `Value`, `Progress`, `Chamber`, `DeliveredTick`, `CarcassCount`, `Stats.CarcassesCut`,
  `Stats.FoodRotted`.
- `M4_SameSeedIdentical`: default config, `M2Defs()`, `M3Script.DefendingBot()`, 54 000 ticks
  (15 days); two worlds agree on `StateHash` every tick; the run contains `CarcassCut` with `B = 4`.
  **Assumed** — seed 2, the lowest whose bot cuts an insect by day 15; re-pick if a retune loses it.
  (Seed 1 was the guess; implemented, its first insect is delivered on day 15.3 and finished on day
  15.8, just outside the run. `RoundTrip_MidCut` uses the same seed. Cheap: one constant.)
- **Seeds that do not need re-picking.** `M2Script` and the plain `M3Script` only ever dig a brood
  chamber, so they never reach tier 2, never haul a dead insect, and never run step 6b. Processing
  makes no draws. So `M2_SameSeedIdentical` (2024), `M3_SameSeedIdentical` (2026),
  `SaveRoundTripTests` (11, 12) and `M3_DifferentSeedDiffers` behave exactly as now. Only golden
  hash constants move: `V1FixtureHash` and `V2FixtureHash` (§8.4).
- **Seeds that may move.** The `BotYear` bots (seeds 1–5) build a processing chamber, so from their
  first dead insect their runs diverge. Their preconditions (tier 2 by day 15 in 3 of 5 seeds; 6
  spiders and 6 raids) are settled before the first insect or barely touched by it; re-check them.

### `SaveRoundTripTests` (add)
- `RoundTrip_MidCut`: the `M4_SameSeedIdentical` run, saved 100 ticks after the first
  `CarcassCut(B = 1)` → `Ok`, equal hash, 1 000 more ticks with equal hashes and events.
  `RoundTrip_JsonStable` extends to it.

### `SaveMigrationTests` (update)
- `V1FixtureMigrates` (two steps), `V2FixtureLoads` → `V2FixtureMigrates` (`Migrated`, not `Ok`), both
  with re-recorded hashes and an empty cutting room; `V3FixtureLoads` (new); `StepsCount == 2`;
  `NullCarcassesBecomeEmpty`; `TooManyCarcassesIsCorrupt`; `TwoCarcassesOneChamberIsCorrupt`;
  `NegativeProcessingIsCorrupt`.

### `YearModelTests` (add)
- `BotYear_ProcessingKeepsUp` (`[Explicit]`, `[Category("Balance")]`, the same two bots, seeds 1–5):
  for every seed and bot, `CarcassesRotted == 0`, the cutting room never holds more than 2, and
  every carcass delivered before day 39.5 is cut by `YearEnded`. Before merging, run `BotYear` on
  the current build and record each seed's defending-bot workers as winter breaks; after M4, assert
  each within ±3 of its record. That is the "gentle economy" check in one line.

---

## 12. Economy

### 12.1 The arithmetic

**Supply.** Dead insects spawn at 0.25 / 0.4 / 0.35 a day (spring, summer, autumn), about 0.35 and
0.3 a day in summer and autumn after the weather factor, at most two at once. A colony takes them
only from tier 2 (day ~10 for the reasonable player of SIM_M3.md §16.2). The reasonable player
takes about 80% of them: about 5 a year at an average 78 food. The keen player about 6, the casual
about 3 (tier 2 later).

**Service.** One carcass takes 0.5 day with a full crew. Utilisation of one chamber is
ρ ≈ 0.35 × 0.5 ≈ 0.18. The mean wait for the room (deterministic service) is
ρ / (2μ(1 − ρ)) = 0.18 / (2 · 2 · 0.82) ≈ 0.05 day. Food reaches the stores in quarters at 1/8, 1/4,
3/8 and 1/2 day, a mean delay of 0.31 day, so about **0.37 day** in all.

**Food in flight** (Little's law): insect income × mean delay. Reasonable: 0.28 a day × 78 ×
0.37 ≈ **8 food** raw on average across summer and autumn; keen ≈ 10. Stores in the same weeks
hold 140–340 (SIM_M3.md §16.2).

**Laying.** The queen lays fully while the stores hold `QueenFedDays` (2) of upkeep. On day 13 that
is 2 × 36.5 = 73 food against 175 stored. An average 8 food delayed never brings the stores near
it. No effect.

**Labour.** Cutters are busy ρ of the time: 2 × 0.18 ≈ 0.4 of a worker on average, from an idle
pool of 14 or more once tier 2 arrives. The ten haulers who bring a carcass in return to idle on
the same delivery, so a carcass always finds hands. No effect on recruiting.

**Rot.** Needs the room unmanned for about 1.5 days, or four carcasses at once. Neither happens to
a colony with idle workers. **Zero** for every played row below.

**Raids.** Raw is not looted. The reasonable player loses 64 food a year to raids out of stores
around 250; 8 raw of it would have been exposed, so about **2 food** a year is saved.

**Stores at capacity.** Before, an insect delivered into nearly full stores lost the overflow at
once. Now it arrives over half a day, and upkeep frees about 20 food of room in that time, so
slightly less is lost. Small, in the player's favour.

**The year's end.** An insect delivered late on day 29 is cut early on day 30, in winter, and all
of it counts. Nothing is lost at the season edge.

### 12.2 The year, expected rows

Against SIM_M3.md §16.3 (threats on). Feedback loop: none new. Cutting has no loop of its own;
the food it delays is too small to move laying, and laying is the colony's only feedback.

| Player | Insects cut | Raw in flight (mean) | Rotted | Workers as winter breaks, M3 | M4 expected |
|---|---|---|---|---|---|
| Keen, η = 0.8 | ~6 | ~10 | 0 | 116 | **115–116** |
| **Reasonable**, η = 0.6 | ~5 | ~8 | 0 | 70 | **70** |
| Reasonable, defends slowly | ~5 | ~8 | 0 | 69 | **69** |
| Reasonable, never home for a raid | ~5 | ~8 | 0 | 68 | **68** |
| Reasonable, ignores rain | ~4 | ~7 | 0 | 53 | **53** |
| Reasonable, ignores Defend | ~4 | ~7 | 0 | 60 | **60** |
| Casual, η = 0.45 | ~3 | ~5 | 0 | 29 | **29** |
| Never builds (tier 1 all year) | 0 | 0 | 0 | 45 (calm) | unchanged |
| Neglect | 0 | — | — | dead day ~16 | unchanged |
| *Stress: Established, starving, Idle 0 for a day and a half* | — | — | **the whole carcass** | — | — |

The calm-year table (SIM_M2.md §14) moves the same way: no row by more than one worker.

**No `SimConfig` retune.** The gentle economy holds: a well-played colony loses nothing it would
notice. The step's teeth are for the colony that has run out of hands, which is the one GAME.md
already describes as failing.

### 12.3 If processing should bite harder

The lever is `CutWorkerSeconds`. At **720** (a day per carcass with 2 cutters): mean delay ≈ 0.7 day,
≈ 15 food in flight, utilisation 0.35; still under one worker at the headline for every row above.
Two insects hauled the same day now finish a day apart, and a third within a day and a half can
rot — so a second processing chamber earns its slot for a keen player. At **1 440** (two days) the
service time equals the rot deadline and every queued carcass rots: do not go above about 1 000
without raising `CarcassRotDays` with it.

---

## 13. Known degenerate strategies and failure modes

- **A second processing chamber is nearly useless at these numbers.** One room is idle 80% of the
  time. It costs 60 food and a slot for a little speed on a rare second carcass. Honest, and cheap
  to fix if wanted (§12.3).
- **The step is invisible to someone who never opens the nest.** They see "Into the cutting room.
  Food to come" and the HUD's "To cut" row tick down. If playtests show players confused that a
  dead insect "did nothing", the HUD row is the fix, not the rule.
- **Food in flight does not count toward the winter bar.** A dead insect hauled on the last day of
  autumn reads as missing from winter's food until it is cut. Cheap (§14).
- **Rot as a death-spiral accelerant.** A starving Established colony with a carcass in the room
  still has food coming: the cutters are killed after idle, and each step feeds the stores. It
  rots only if the nest is empty of idle and cutters for a day and a half. Converges, not runs away.
- **Tier drop with a carcass in the room.** A colony that falls back to Young keeps its processing
  chamber, so cutting carries on. Only marking new insects is lost.
- **A raid breach during cutting.** Cutters fight and die after idle, so a breach can stall a cut;
  the room refills from the returning haulers. At worst a carcass waits, and two days is long.
- **Saves from before M4 mid-haul.** A dead insect carried when the save was made now arrives raw.
  Intended.

---

## 14. Markers

**Assumed** — dead insects only go raw; seeds, leaves and sugar do not (§1). Medium cost to change.

**Assumed** — a carcass keeps its delivered value; the nest's clock is a hard 2-day deadline (§2.1,
§2.4). Cheap.

**Assumed** — 2 cutters per processing chamber, 360 worker-seconds a carcass, four quarter steps
(§2.3). Cheap: data.

**Assumed** — food from a cut step that does not fit is lost, not held (§2.3). Cheap.

**Assumed** — cutters are killed after idle and before nurses; taps and trails never take them
(§3.2, §3.3). Cheap.

**Assumed** — raids loot `Food` only; carcasses are safe (§3.4). Cheap.

**Assumed** — the winter bar and the "store winter's need" goals count `Food` only, not raw food
still to cut. Cheap to change: one term, `Food + RawFood`, in the gauge and the goal checks.

**Assumed** — the summer dead-insect goal is met on delivery, as today, not when the cutting is
done: the haul is the player's act; the cutting is the colony's. Cheap: one check in the goals
tracker.
