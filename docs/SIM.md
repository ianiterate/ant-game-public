# Simulation spec — M1 vertical slice

Implementation spec for `Assets/Scripts/Sim`. It builds on the M0 core
(`Rng`, `Ids`, `SimConfig`, `SimClock`, `SimCommand`/`SimInput`, `SimEvent`, `World`)
and replaces none of it. [GAME.md](GAME.md) is what the game is; this file is how the
simulation does it. Run `/open-questions` for everything marked below.

Units throughout: distance in **u** (1 u = 1 cm), time in **s**, rates per second.
`dt` = `SimConfig.TickSeconds` (0.1 s). Everything is 2D on the ground plane
(`Vector2`; the Game layer maps `(x, y)` ↔ world `(x, z)`).

---

## 0. Ground rules

- One namespace, `AntGame.Sim`, as M0 already uses.
- Fixed-capacity slot arrays, allocated once in the `World` constructor. No allocation
  per tick. Iterate slots in index order, always. No `Dictionary`, no LINQ.
- Slot reuse: lowest free index first. A slot's `Gen` starts at 0 and is incremented
  **when the slot is taken**, so the first occupant has `Gen = 1` and `default` ids
  stay "none". A stale id (gen mismatch) never resolves.
- Terminal states (`Delivered`, `Gone`, `Done`, `Aborted`) stay visible for the rest
  of the tick that set them, and the slot is freed at the start of the owning system's
  step on the next tick.
- The only `Rng` draws in M1 are the two per ant spawn (§5.3). Nothing else is random.
- `Sim/AssemblyInfo.cs`: `[assembly: InternalsVisibleTo("AntGame.Tests.EditMode")]`.
  Test hooks below are `internal`.

### New file `Core/SimLimits.cs`

Engineering capacities and world-scale facts, not balance (so not in `SimConfig`):

```csharp
public static class SimLimits
{
    public const int MaxAnts = 256;
    public const int MaxItems = 64;
    public const int MaxTrails = 16;
    public const int MaxTasks = 16;
    public const float AntHalfLength = 0.5f;   // a worker is ~1 u long
    public const float MinPointSpacing = 0.01f; // closer points are merged
}
```

`MaxTrailPoints` is derived in the `World` constructor:
`Mathf.CeilToInt(cfg.MaxTrailLength / cfg.TrailSampleDistance) + 2`.

### `SimConfig` additions

Existing fields and defaults are unchanged. Add:

| Field | Default | Meaning |
|---|---|---|
| `StartTimeOfDay01` | `0.25f` | Time of day at tick 0 (0.25 = dawn). |
| `TrailMaxStrength` | `1f` | Upper bound on any trail's strength. |
| `ReachRadius` | `3f` | Player reach beyond an item's `Radius` (u), for Mark and for counting toward a party. |
| `DepositSeconds` | `1f` | How long haulers stand at the nest after delivery before rejoining Idle. |
| `AntLaneHalfWidth` | `0.6f` | Max constant sideways offset of an ant from the trail centre line (u). |
| `WobbleAmplitude` | `0.25f` | Sideways sinusoidal sway (u). |
| `WobbleWavelength` | `6f` | Distance along the trail per sway cycle (u). |
| `StartingBrood` | `4` | Brood count at start. Inert in M1. |

`Assets/Settings/SimConfig.asset` already exists; Unity fills missing fields from the
initialisers on load. Verify in the inspector after the change compiles.

**Assumed** — the game starts at dawn (`StartTimeOfDay01 = 0.25`), so the first 180 s
are daylight and the M1 loop never runs at night speed. Without it the slice opens at
midnight in the dark at 0.6× speed. Cheap to change: one config value.

### `SimClock` change

Time of day is offset by `StartTimeOfDay01`; `Tick` still starts at 0.

```csharp
public static SimClock Start(SimConfig cfg)   // new; World uses this. Tick = 0, derived fields filled.
public static SimClock Start()                // M0 literals: Tick 0, Day 0, TimeOfDay01 0, Spring, IsNight = false
public void Advance(SimConfig cfg)            // Tick++; then Derive(cfg)
private void Derive(SimConfig cfg)
{
    double seconds = Tick * (double)cfg.TickSeconds
                   + cfg.StartTimeOfDay01 * (double)cfg.DayLengthSeconds;
    double days = seconds / cfg.DayLengthSeconds;
    // Day, TimeOfDay01, Season, IsNight exactly as M0 computes them
}
```

`Start()` is **not** equivalent to `Start(cfg)` with `StartTimeOfDay01 = 0`: it keeps the M0
literals, so `IsNight` is `false` at tick 0, whereas `Start(cfg)` derives `IsNight = true` at
midnight. They agree from the first `Advance` on. `World` always uses `Start(cfg)`.

`ClockTests.Config()` sets `StartTimeOfDay01 = 0f` (its assertions assume midnight).

---

## 1. Items

Files: `Sim/Items/ItemKind.cs`, `ItemDef.cs`, `WorldItem.cs`, `ItemSystem.cs`.
(The plan named this folder `World/`; `Items/` avoids a folder named like the `World` class.)

```csharp
public enum ItemKind { None = 0, SugarCube, Seed, Leaf, DeadInsect, Pinecone }
public static class ItemKinds { public const int Count = 6; } // keep in step with the enum

[Serializable]
public struct ItemDef
{
    public ItemKind Kind;
    public int AntsRequired;   // party size including the player, >= 1
    public int TierRequired;   // colony tier needed to mark it, >= 1
    public float FoodValue;    // food added on delivery when fresh
    public float SpoilSeconds; // 0 = never spoils
    public float Radius;       // u; ants stop and stand at this distance from the centre
}

public enum ItemState { None = 0, Lying, Claimed, Hauling, Delivered, Gone }

public struct WorldItem
{
    public ItemId Id;
    public bool Active;
    public ItemKind Kind;
    public ItemState State;
    public Vector2 Pos;      // current centre; follows the trail while Hauling
    public float S, PrevS;   // distance from the nest along Trail; meaningful while Hauling
    public float Age;        // s since spawn, for spoilage
    public TaskId Task;      // set while Claimed/Hauling
    public TrailId Trail;    // set while Claimed/Hauling
}
```

**Where defs live.** Game layer: `Game/Items/ItemDefAsset.cs` (ScriptableObject holding one
`[SerializeField] private ItemDef def`, same pattern as `SimConfigAsset`) and
`Game/Items/ItemDatabase.cs` (ScriptableObject holding `ItemDefAsset[]`, with
`ItemDef[] ToDefs()` returning an array indexed by `(int)ItemKind`, length
`ItemKinds.Count`, unset kinds left `default`). M1 ships one asset,
`Assets/Settings/Items/SugarCube.asset`:

| Kind | AntsRequired | TierRequired | FoodValue | SpoilSeconds | Radius |
|---|---|---|---|---|---|
| SugarCube | 6 | 1 | 60 | 0 | 1.2 |

The other kinds in GAME.md's table are M2 content; no assets in M1.

**Assumed** — sugar cube `Radius = 1.2 u` (a real 1.6 cm cube; about one and a half ant
lengths across). The cube mesh must match. Cheap to change: data plus the mesh.

### State machine

| From | To | When | Event |
|---|---|---|---|
| — | Lying | `World.SpawnItem` | — |
| Lying | Claimed | a player trail to it completes (§3.4) | `TaskCreated` |
| Claimed | Lying | its task aborts (§4.4) | `TaskAborted` |
| Claimed | Hauling | party complete (§4.3) | `ItemPickedUp` |
| Hauling | Delivered | `S` reaches 0 (§1.2) | `ItemDelivered`, `TaskComplete` |
| Lying / Claimed | Gone | `SpoilSeconds > 0 && Age >= SpoilSeconds` | `ItemSpoiled` |

`Hauling → Gone` does not exist in M1 (theft is M3).

**Assumed** — a hauled item keeps ageing but cannot spoil away mid-haul; its delivered
value still scales down. Not exercised by the sugar cube. Cheap to change.

### 1.1 Spawning (setup API)

```csharp
public ItemId SpawnItem(ItemKind kind, Vector2 pos)
```

Takes the lowest free item slot; `State = Lying`, `Pos = pos`, `Age = 0`, `S = PrevS = 0`.
Throws `ArgumentException` if `kind` is `None`, outside `1..ItemKinds.Count − 1`, or has no
def (`defs[(int)kind].AntsRequired == 0`). Returns
`default` if all slots are taken. Called between ticks only (scene setup and tests).

The Game layer calls it once per `WorldItemView` at startup in a stable order: sorted
by `pos.x`, then `pos.y`. (M2 moves spawning inside the simulation.)

### 1.2 `ItemSystem.Step` (tick step 6)

For each item slot in index order:

1. If `State` is `Delivered` or `Gone`: free the slot (`Active = false`), continue.
2. `Age += dt`.
3. If `State` is `Lying` or `Claimed`, `SpoilSeconds > 0` and `Age >= SpoilSeconds`:
   `State = Gone`, raise `ItemSpoiled(A = item index, P = Pos)`. Continue.
4. If `State == Hauling`:
   - `PrevS = S`; `d = WalkSpeed · HaulSpeedFactor · night · dt`, where
     `night = Clock.IsNight ? NightSpeedFactor : 1`; `S = max(0, S − d)`.
   - `Pos = trail.Sample(S)`.
   - For every active ant (index order) with `Carrying == Id`: `ant.S = S`; count them as `n`.
   - `trail.Reinforce(n · d, cfg)` (§3.2).
   - If `S <= 0`: **deliver**:
     - `spoil01 = SpoilSeconds > 0 ? Clamp01(Age / SpoilSeconds) : 0`;
       `value = FoodValue · (1 − spoil01)`; `Colony.Food += value`.
     - `State = Delivered`, `Pos = NestPos`.
     - Each carrier: `Phase = Depositing`, `Timer = DepositSeconds`, `Carrying = default`.
     - Task: `Phase = Done`.
     - Raise `ItemDelivered(A = item index, B = (int)Mathf.Round(value), P = NestPos)`,
       then `TaskComplete(A = task index, B = item index)`.

---

## 2. Colony

Files: `Sim/Colony/ColonyState.cs`, `ColonySystem.cs`.

```csharp
public struct ColonyState
{
    public int Idle;      // workers in the nest, available to recruit
    public int Nursing;   // workers in the nest tending brood; never recruited in M1
    public int Brood;     // inert in M1
    public double Food;   // >= 0; double, because float upkeep sums drift ~2% near 1000 food
    public int Tier;      // derived, stored so a change can raise an event
}
```

`World.AntsOutside` = number of active ant records (derived, not stored).
`Population = Idle + Nursing + AntsOutside`. Brood is not population.

Start: `Idle = StartingIdle (8)`, `Nursing = StartingNursing (4)`, `Brood = StartingBrood (4)`,
`Food = StartingFood (20)`, `Tier = TierFor(Population)` (no event at start).

`TierFor(p)`: `tier = 1`; for each `t` in `TierThresholds` (ascending): if `p >= t`, `tier++`.
With `{25, 75}`: <25 → 1, 25–74 → 2, ≥75 → 3. The chamber condition in GAME.md is M2.

Workers move only between `Idle` and ant records: spawn does `Idle--`, return does `Idle++`.

### 2.1 `ColonySystem.Step` (tick step 7)

1. `workers = Idle + Nursing + AntsOutside`.
2. `before = Food`;
   `Food = Math.Max(0.0, Food − (double)UpkeepPerAntPerDay · workers · dt / DayLengthSeconds)`
   (all in `double`).
3. If `before > 0 && Food == 0`: raise `FoodRanOut`.
4. `t = TierFor(Population)`; if `t != Tier`: `Tier = t`, raise `TierChanged(A = t)`.

With defaults: 12 workers × 0.5 = 6 food/day = 0.01667 food/s.

**Assumed** — at `Food = 0` in M1 nothing else happens: food clamps at zero, one
`FoodRanOut` event fires for the HUD, no ant dies. With no income the 20 starting food
lasts 3.3 days (20 min). M2 decides the real consequence. Cheap to change.

**Assumed** — upkeep is charged on workers only (idle, nursing and outside); brood costs
nothing in M1. Cheap to change.

**Assumed** — in M1 nursing is a fixed count and recruitment draws only from `Idle`.
Cheap to change until M2 brood work starts.

**Undecided** — how workers move between Idle and Nursing, and whether a big recruitment
can pull nurses (GAME.md: "over-recruiting for a haul starves the brood"). Blocks M2 brood
and queen, and the HUD's nursing warning. Not needed for M1.

---

## 3. Trails

Files: `Sim/Trails/Trail.cs`, `TrailDraft.cs`, `TrailSystem.cs`. Drafting lives in
`Core/CommandSystem.cs` (§6.2).

### 3.1 `Trail`

One preallocated instance per slot (class, so the point buffers are reused):

```csharp
public sealed class Trail
{
    public TrailId Id;
    public bool Active;
    public readonly Vector2[] Points;  // length MaxTrailPoints; Points[0] = nest end
    public readonly float[] Cum;       // Cum[0] = 0, Cum[i] = Cum[i-1] + |Points[i] - Points[i-1]|
    public int Count;                  // >= 2 while Active
    public float Length => Cum[Count - 1];
    public float Strength;             // dimensionless, in [TrailFloor, TrailMaxStrength] while referenced
    public bool PlayerLaid;

    public void Set(ReadOnlySpan<Vector2> nestToItem);   // copies points, merges near-duplicates, rebuilds Cum
    public void Sample(float s, out Vector2 pos, out Vector2 tangent);
    public Vector2 Sample(float s);
    public void Reinforce(float distance, SimConfig cfg);
}
```

`S` is distance from the nest end: `S = 0` at the nest, `S = Length` at the item centre.

**`Set(nestToItem)`**: throws `ArgumentException` if given fewer than 2 points or more than
the buffer holds (`MaxTrailPoints`). Copies points in order, dropping any point closer than
`MinPointSpacing` to the previous kept point — except the last (item-end) point, which always
survives: if it is too close, it replaces the previous kept point. If that previous point is
the nest point, the item end cannot replace it, so fewer than 2 points remain. Rebuilds `Cum`.
Throws `ArgumentException` if fewer than 2 points remain.

**`Sample(s)`**: clamp `s` to `[0, Length]`. Binary-search `Cum[0..Count-1]` for the segment
`i` with `Cum[i] <= s < Cum[i+1]`; if `s >= Length` use the last segment
(`i = Count − 2`). `t = (s − Cum[i]) / (Cum[i+1] − Cum[i])`;
`pos = Lerp(Points[i], Points[i+1], t)`; `tangent = normalized(Points[i+1] − Points[i])`
(points toward the item). Segments are never shorter than `MinPointSpacing`, so no divide
by zero.

### 3.2 Strength

Each tick, in `TrailSystem.Step` (tick step 3), for each active trail in index order:

1. `Strength *= decayPerTick`, where `decayPerTick = Mathf.Exp(−TrailDecayPerSecond · TickSeconds)`
   is computed once in the `World` constructor (0.995012 with defaults).
2. `users` = active tasks with `Trail == Id` and `Phase` in {`Recruiting`, `Hauling`}
   plus active ants with `Trail == Id`. Recomputed by scan every tick; not stored.
3. If `Strength < TrailFloor`:
   - `users == 0`: free the slot, raise `TrailFaded(A = trail index)`.
   - otherwise: `Strength = TrailFloor`.

**Reinforcement** — `Reinforce(distance, cfg)`:
`Strength = Min(TrailMaxStrength, Strength + TrailReinforcePerPass · distance / Length)`.
One full traversal by one ant adds `TrailReinforcePerPass` (0.1). Called by Outbound ants
(each tick, distance = their step) and by hauled items (distance × carriers). Ants going
home empty and ants standing still do not reinforce.

Net per second: `dStrength/dt = −0.05 · Strength + 0.1 · (ant-distance walked per second) / Length`.
This is linear with proportional decay, so it **converges**:
`Strength* = 0.1 · n · v / (0.05 · L)` for `n` ants walking at `v`, capped at
`TrailMaxStrength`. Recruitment is capped by `AntsRequired`, so it cannot run away.
From 1.0 with nobody on it, a trail falls below the floor in 783 ticks (78.3 s).

**Assumed** — reinforcement is proportional to distance walked toward food or carrying
food, and strength is capped at 1. Medium cost to change: retunes recruitment timing.

**Assumed** — a trail that a live task or any ant still uses never fades; it holds at the
floor. Consequence: a stale task keeps recruiting at `0.5 × 0.02 = 0.01` ants/s
(one ant per 100 s) until its item is hauled. Cheap to change (M2 can add task expiry).

### 3.3 `TrailDraft` (the trail the player is laying)

```csharp
public struct TrailDraft
{
    public bool Active;
    public ItemId Item;
    public Vector2[] Points;  // length MaxTrailPoints, allocated once; Points[0] = item centre
    public int Count;
    public float Length;      // sum of segment lengths so far
}
```

The draft is recorded item → nest (the order the player walks). There is at most one.

**Append(p)**: if `|p − Points[Count−1]| < TrailSampleDistance`, ignore. Otherwise append,
`Length += distance`. If `Length > MaxTrailLength` (or `Count` would exceed
`MaxTrailPoints − 1`, keeping one slot for the nest point): **abandon** with `TooLong`.

**Abandon(reason)**: `Active = false`, raise `TrailAbandoned(A = (int)reason, P = PlayerPos)`.
The item is untouched (still `Lying`).

```csharp
public enum TrailAbandonReason { None = 0, Cancelled, TooLong, TooShort, ItemUnavailable, NoFreeSlot }
```

### 3.4 Player-laid trail lifecycle

1. **MarkTrailStart** (§6.2) validates and opens the draft with `Points[0] = item.Pos`.
   Raises `TrailMarkStarted(A = item index, P = item.Pos)`.
2. **Every tick while the draft is active**, after the tick's commands:
   `Append(input.PlayerPos)`, then if `|PlayerPos − NestPos| <= NestRadius`: **Complete**.
   The simulation samples the player itself; the Game layer does not need to send
   `MarkTrailPoint` or `MarkTrailEnd` in M1.
3. **Complete**:
   1. Resolve the item; if it no longer exists or `State != Lying`: abandon `ItemUnavailable`.
   2. If `|NestPos − last point| > MinPointSpacing`: append `NestPos` (ignoring the sample
      spacing rule, but still subject to the `TooLong` check).
   3. If `Length <= def.Radius + AntHalfLength`: abandon `TooShort`.
   4. If no free trail slot or no free task slot: abandon `NoFreeSlot`.
   5. **Single player trail**: if another active task has `PlayerLaid` and is `Recruiting`,
      abort it with `Replaced` (§4.4). If it is `Hauling`, leave it to finish. Either way
      clear `PlayerLaid` on the old trail; it then decays like any other.
   6. Take a trail slot: `Set(points reversed)` (nest first), `Strength = PlayerTrailStrength`,
      `PlayerLaid = true`.
   7. Take a task slot (§4.1). Item: `State = Claimed`, `Task`, `Trail` set.
   8. Draft `Active = false`. Raise `TrailLaid(A = trail index, B = task index, P = item.Pos)`,
      then `TaskCreated(A = task index, B = item index)`.
4. **Removal**: a trail is only ever removed by fading (§3.2). After delivery it is
   unreferenced once the haulers have deposited, and fades from wherever its strength is.

**Assumed** — one player trail at a time means: completing a new one aborts the previous
one's task if it is still recruiting (its ants turn home, its item is free again); a haul
already under way finishes. Cheap to change.

**Assumed** — a `Claimed` item cannot be re-marked (the command is rejected); a weak
trail cannot be refreshed by re-marking in M1. Cheap to change.

---

## 4. Tasks

Files: `Sim/Tasks/TaskKind.cs` (`TaskKind`, `TaskPhase`), `ColonyTask.cs`, `TaskSystem.cs`.
(`ColonyTask`, not `Task`, to avoid `System.Threading.Tasks.Task`.)

```csharp
public enum TaskKind { None = 0, Haul }
public enum TaskPhase { None = 0, Recruiting, Hauling, Done, Aborted }

public struct ColonyTask
{
    public TaskId Id;
    public bool Active;
    public TaskKind Kind;
    public TaskPhase Phase;
    public ItemId Item;
    public TrailId Trail;
    public bool PlayerLaid;
    public float RecruitAccum;  // in [0, 1]
    public int Taps;            // TapAnt batches used
    public int Haulers;         // next hauler slot; only increases
    // Derived each tick by scan, not folded into the hash:
    public int Assigned;        // ants with this task in Outbound, AtTarget or Hauling
    public int AtTarget;        // ants with this task in AtTarget
}
```

### 4.1 Creation

Only by trail completion (§3.4). `Kind = Haul`, `Phase = Recruiting`, `PlayerLaid = true`,
`RecruitAccum = 0`, `Taps = 0`, `Haulers = 0`.

### 4.2 `TaskSystem.Step` (tick step 4)

1. Free every task slot whose `Phase` is `Done` or `Aborted`.
2. For every active task, recount `Assigned` and `AtTarget` by scanning ants.
3. For each active task in index order, skipping any whose `Phase` is already `Done` or
   `Aborted` (set earlier this tick, e.g. by a `Replaced` abort in step 2):
   - Resolve the item. If it fails (stale id) or `State == Gone`: abort `ItemGone`; continue.
   - If `Phase == Recruiting`:
     - `playerHere = |PlayerPos − item.Pos| <= def.Radius + ReachRadius`.
     - `atTarget = AtTarget + (playerHere ? 1 : 0)`.
     - If `AtTarget >= 1 && atTarget >= def.AntsRequired`: **start haul** (§4.3); continue.
     - If `Assigned >= def.AntsRequired`: `RecruitAccum = 0`; continue.
     - `RecruitAccum += RecruitPerSecond · trail.Strength · dt`.
     - While `RecruitAccum >= 1 && Assigned < def.AntsRequired && Colony.Idle > 0` and an
       ant slot is free: spawn an ant (§5.3), `Assigned++`, `RecruitAccum −= 1`.
     - `RecruitAccum = Min(RecruitAccum, 1)` (no banking while Idle is empty).

Recruitment rate is `RecruitPerSecond × Strength` ants/s: 0.5/s on a fresh player trail.

**Assumed** — recruitment is a deterministic accumulator, not a random draw: ants leave
at evenly spaced moments. Cheap to change (one `rng.Chance` per tick instead).

**Assumed** — the recruitment cap is `AntsRequired` NPC ants, whether or not the player
helps. If the player is at the item the haul starts one ant early and the last recruit
joins the party on the way home. Cheap to change.

**Assumed** — the player counts toward the party only for starting the haul, and only
while within `def.Radius + ReachRadius` (4.2 u for the sugar cube). Once lifted, a haul
never stalls, whoever walks away. At least one NPC ant must be at the item (a lone
player cannot haul; one-ant items are `PickUp`, M2). Cheap to change.

### 4.3 Start haul

`Phase = Hauling`. Item: `State = Hauling`, `S = PrevS = trail.Length`. For each ant in index
order with `Task == Id && Phase == AtTarget`: `Phase = Hauling`, `Carrying = item.Id`,
`S = PrevS = item.S`, `Slot = Haulers++`. Raise `ItemPickedUp(A = item index, B = task index, P = item.Pos)`.
Recruitment stops.

### 4.4 Abort(reason)

```csharp
public enum TaskAbortReason { None = 0, ItemGone, Replaced }
```

`Phase = Aborted`. Every ant with `Task == Id` in `Outbound`, `AtTarget` or `Hauling`:
`Phase = GoingHome`, `Task = default`, `Carrying = default` (keeps `Trail` and `S`). If the
item still resolves and is `Claimed`: `State = Lying`, `Task = Trail = default`.
Raise `TaskAborted(A = task index, B = (int)reason)`.

### 4.5 Taps

`TapAnt` (§6.2) spawns `n = Min(TapBatch, Colony.Idle, def.AntsRequired − Assigned, free ant slots)`
ants immediately on the task's trail (`Assigned` counted by scan at that moment), then
`Taps++`.

**Assumed** — in M1 a tap only works from the nest (`|PlayerPos − NestPos| <= NestRadius`)
and only sends ants down an existing recruiting task's trail. Cheap to change.

**Undecided** — "tap a nearby ant" outside the nest (GAME.md core loop step 4): what it
does to an ant already on another job. Blocks nothing in M1; blocks multi-trail play in M2.

---

## 5. Ants

Files: `Sim/Ants/AntPhase.cs`, `AntState.cs`, `AntSystem.cs`.

```csharp
public enum AntPhase { None = 0, Outbound, AtTarget, Hauling, Depositing, GoingHome }

public struct AntState
{
    public AntId Id;
    public bool Active;
    public AntPhase Phase;
    public TaskId Task;      // default once the task is gone
    public TrailId Trail;    // always set while Active
    public float S, PrevS;   // distance from the nest along Trail
    public ItemId Carrying;  // set while Hauling
    public float Lane;       // constant sideways offset, u
    public float WobblePhase;// radians
    public float Timer;      // Depositing countdown, s
    public int Slot;         // position in the hauling ring
}
```

### 5.1 Movement — `AntSystem.Step` (tick step 5)

`night = Clock.IsNight ? NightSpeedFactor : 1`; `v = WalkSpeed · night · dt`
(0.3 u/tick by day, 0.18 at night).

For each active ant in index order, first `PrevS = S`, then by phase:

- **Outbound**: `S += v`; `trail.Reinforce(v, cfg)`. Resolve the task and item.
  - Task no longer active (defensive): `Phase = GoingHome`.
  - Else if task `Phase == Hauling` and `S >= item.S − def.Radius`: join the party:
    `Phase = Hauling`, `Carrying = item.Id`, `S = PrevS = item.S`, `Slot = task.Haulers++`.
  - Else if `S >= trail.Length − def.Radius`: `S = trail.Length − def.Radius`, `Phase = AtTarget`.
- **AtTarget**: nothing.
- **Hauling**: nothing; `ItemSystem` moves carriers (step 6).
- **Depositing**: `Timer −= dt`; if `Timer <= 0`: return to nest (below).
- **GoingHome**: `S −= v`; if `S <= 0`: `S = 0`, return to nest.

**Return to nest**: `Active = false`, `Colony.Idle++`, raise `AntReturned(A = ant index)`.

An outbound ant always meets a hauled item before delivery (the ant starts at `S = 0`,
the item ends there), so no recruited ant is ever left behind.

### 5.2 Position (read API, deterministic)

```csharp
public Vector2 AntPosition(int index, float alpha = 1f)
```

`s = Lerp(PrevS, S, alpha)`. `lateral(s) = Lane + WobbleAmplitude · Sin(2π · s / WobbleWavelength + WobblePhase)`.
`OnTrail(s, off)`: `trail.Sample(s, out p, out t)`; `n = (−t.y, t.x)`; `p + n · off`.

| Phase | Position |
|---|---|
| Outbound, AtTarget, GoingHome | `OnTrail(s, lateral(s))` |
| Hauling | `c = ItemPosition(item, alpha)`; tangent `t` at the item's lerped `S`; `n = (−t.y, t.x)`; `θ = 2π · (Slot + 0.5) / def.AntsRequired`; `r = def.Radius + AntHalfLength`; `c + (t · cos θ + n · sin θ) · r` |
| Depositing | `OnTrail(0, Lane)` |

`ItemPosition(int index, float alpha)`: `Lying`/`Claimed` → `Pos`; `Hauling` →
`trail.Sample(Lerp(PrevS, S, alpha))`; `Delivered` → `NestPos`.

Sideways offset is bounded by `AntLaneHalfWidth + WobbleAmplitude` = 0.85 u.

### 5.3 Spawn

Lowest free ant slot; `Gen++`; `Active = true`; `Phase = Outbound`; `Task`, `Trail` from the
task; `S = PrevS = 0`; `Carrying = default`; then, **in this order**:
`Lane = rng.Range(−AntLaneHalfWidth, AntLaneHalfWidth)`,
`WobblePhase = rng.Range(0f, 2π)`; `Timer = 0`, `Slot = 0`. `Colony.Idle−−`.
Raise `AntLeftNest(A = ant index, B = task index, P = NestPos)`.

A spawned ant moves in the same tick (spawns happen in steps 2 and 4, movement in step 5).

**Assumed** — ants spread sideways by a random constant lane plus a sinusoidal sway, both
fixed at spawn. Presentation only; cheap to change, but it changes `StateHash`.

**Assumed** — haulers stand 1 s at the nest entrance (`Depositing`) before rejoining Idle,
so the view has a beat for "food goes in". Cheap to change.

---

## 6. World

### 6.1 Constructor and read API

```csharp
public World(SimConfig config, ItemDef[] itemDefs, Vector2 nestPos, int seed)
public World(SimConfig config, int seed) : this(config, new ItemDef[ItemKinds.Count], Vector2.zero, seed)
```

Validates `itemDefs.Length == ItemKinds.Count` and, for every def with `AntsRequired > 0`,
`(int)def.Kind == index`; throws `ArgumentException` otherwise. Copies the array.
Allocates every slot array, the trail buffers and the draft buffer.
`_clock = SimClock.Start(config)`. Colony as §2.

Public reads (no allocation):

```csharp
ReadOnlySpan<AntState> Ants        // all MaxAnts slots; check Active
ReadOnlySpan<WorldItem> Items      // all MaxItems slots
ReadOnlySpan<ColonyTask> Tasks     // all MaxTasks slots
IReadOnlyList<Trail> Trails        // all MaxTrails slots
ColonyState Colony
int AntsOutside, Population
TrailDraft Draft                   // view draws the trail being laid
Vector2 NestPos
ItemDef GetDef(ItemKind kind)
bool TryGetItem(ItemId id, out WorldItem item)
Vector2 AntPosition(int index, float alpha = 1f)
Vector2 ItemPosition(int index, float alpha = 1f)
ItemId SpawnItem(ItemKind kind, Vector2 pos)
int StateHash()
```

Systems (`CommandSystem`, `TrailSystem`, `TaskSystem`, `AntSystem`, `ItemSystem`,
`ColonySystem`) are `internal static class`es with a `Step(World …)` method. `World` gives them
internal access to its state: `AntState[] AntSlots`, `WorldItem[] ItemSlots`,
`ColonyTask[] TaskSlots`, `Trail[] TrailSlots`, `ref TrailDraft DraftRef`,
`Vector2[] ScratchPoints` (a `MaxTrailPoints` buffer for reversing the draft without
allocating), `ref ColonyState ColonyRef`, `MaxTrailPoints`, `TrailDecayPerTick`,
`TryResolveItem/Task/Trail(id)`, `FreeTrailSlot()`, `FreeTaskSlot()`.

Internal test hooks: `ref ColonyState ColonyRef`, `ref WorldItem ItemRef(int index)`,
`ref AntState AntRef(int index)`, `Trail TrailAt(int index)`,
`TrailId AddTrailForTest(ReadOnlySpan<Vector2> nestToItem, float strength)`.

### 6.2 Commands — `Core/CommandSystem.cs` (tick step 2)

`SimCommand` and `SimInput` keep their M0 shape. Commands are applied **in list order**;
the world never modifies `input.Commands` (the Game layer clears it after the tick). A
rejected command changes no state and raises `CommandRejected(A = (int)kind, B = (int)reason)`.

```csharp
public enum RejectReason
{
    None = 0, NoSuchItem, ItemNotAvailable, OutOfReach, TierTooLow, DraftActive, NoDraft,
    NotAtNest, NoSuchTask, TaskNotRecruiting, TapLimit, NothingToSend, NotImplemented,
}
```

| Command | Arguments | Validation, in order (first failure wins) | Effect |
|---|---|---|---|
| `MarkTrailStart` | `A` = item index, `B` = item gen | draft active → `DraftActive`; id does not resolve → `NoSuchItem`; `State != Lying` → `ItemNotAvailable`; `|PlayerPos − item.Pos| > def.Radius + ReachRadius` → `OutOfReach`; `def.TierRequired > Colony.Tier` → `TierTooLow` | open draft (§3.4) |
| `MarkTrailPoint` | `P` | no draft → `NoDraft` | `Append(P)` |
| `MarkTrailEnd` | — | no draft → `NoDraft`; `|PlayerPos − NestPos| > NestRadius` → `NotAtNest` | Complete (§3.4) |
| `MarkTrailCancel` | — | no draft → `NoDraft` | abandon `Cancelled` |
| `TapAnt` | `A` = task index and `B` = gen, or `A = −1` for the active task with `PlayerLaid` set **and** whose trail is still `PlayerLaid` (a replaced haul keeps its task flag but its trail loses it) | not within `NestRadius` of the nest → `NotAtNest`; no such active task → `NoSuchTask`; `Phase != Recruiting` → `TaskNotRecruiting`; `Taps >= MaxTapsPerTask` → `TapLimit`; `n == 0` → `NothingToSend` | spawn `n` (§4.5) |
| `PickUp`, `Deposit`, `StartBuild` | — | always → `NotImplemented` | none (M2) |

After the command list: the draft's per-tick sampling and auto-complete (§3.4 step 2).

`MarkTrailPoint` and `MarkTrailEnd` exist for tests and scripted trails; the Game layer's
Mark button sends `MarkTrailCancel` when `Draft.Active`, otherwise `MarkTrailStart` for the
interaction probe's item.

**Assumed** — marking an item above the colony's tier is refused when you press Mark
(`TierTooLow`), not after ants arrive. Cheap to change.

### 6.3 Tick order

```csharp
public void Tick(in SimInput input)
{
    Events.Clear();
    StepClock();                                  // 1  (M0, unchanged)
    CommandSystem.Step(this, in input, ref _rng); // 2  commands, then draft sampling
    TrailSystem.Step(this);                       // 3  decay, floor, fade
    TaskSystem.Step(this, in input, ref _rng);    // 4  free, recount, abort, haul start, recruit
    AntSystem.Step(this);                         // 5  movement, arrival, join, deposit, return
    ItemSystem.Step(this);                        // 6  free, age, spoil, haul movement, delivery
    ColonySystem.Step(this);                      // 7  upkeep, tier
    // Weather (M3) goes between 1 and 2; Threats (M3) after 7.
    // Events stay in the queue for the Game layer to drain.
}
```

Consequences engineers rely on: recruitment sees this tick's decayed strength; a spawned
ant moves the tick it spawns; an ant arriving in step 5 counts toward the party in the next
tick's step 4; a delivery in step 6 frees the item and task slots next tick.

### 6.4 Events

Existing kinds keep their values. Payloads:

| Kind | A | B | P | Raised in |
|---|---|---|---|---|
| `DayStarted`, `NightStarted`, `SeasonChanged` | as M0 | | | 1 |
| `AntLeftNest` | ant index | task index | nest | 2, 4 |
| `AntReturned` | ant index | | | 5 |
| `TrailLaid` | trail index | task index | item pos | 2 |
| `TrailAbandoned` | `TrailAbandonReason` | | player pos | 2 |
| `TrailFaded` | trail index | | | 3 |
| `ItemPickedUp` | item index | task index | item pos | 4 |
| `ItemDelivered` | item index | food added (rounded) | nest | 6 |
| `TaskCreated` | task index | item index | | 2 |
| `TaskComplete` | task index | item index | | 6 |

New kinds, **appended after `ThreatSpawned`**:

| Kind | A | B | P | Raised in |
|---|---|---|---|---|
| `TrailMarkStarted` | item index | | item pos | 2 |
| `TaskAborted` | task index | `TaskAbortReason` | | 2, 4 |
| `ItemSpoiled` | item index | | item pos | 6 |
| `CommandRejected` | `CommandKind` | `RejectReason` | | 2 |
| `FoodRanOut` | | | | 7 |
| `TierChanged` | new tier | | | 7 |

`AntsLost`, `WeatherChanged`, `ThreatSpawned` are not raised in M1.

### 6.5 `StateHash`

Keep M0's fold (Seed, Tick low/high, Day, Rng low/high) first, unchanged, then append.
`F(x)` is `h = h * 31 + x` (unchecked); floats fold as `BitConverter.SingleToInt32Bits(f)`;
doubles fold as two ints, low then high 32 bits of `BitConverter.DoubleToInt64Bits(d)`; bools as 0/1; enums as `int`; an id folds `Index` then `Gen`.

1. **Colony**: `Idle, Nursing, Brood, Food` (double: low, high), `Tier`.
2. **Draft**: `Active`; if active: `Item, Count, Length`, then each point's `x, y`.
3. **Items**, for every slot `0..MaxItems−1`: `Gen`; if `Active`: `Kind, State, Pos.x, Pos.y, S, PrevS, Age, Task, Trail`.
4. **Trails**, every slot: `Gen`; if `Active`: `Count, Strength, PlayerLaid`, then each point's `x, y`.
   (`Cum` is derived from points; not folded.)
5. **Tasks**, every slot: `Gen`; if `Active`: `Kind, Phase, Item, Trail, PlayerLaid, RecruitAccum, Taps, Haulers`.
   (`Assigned`, `AtTarget` are derived; not folded.)
6. **Ants**, every slot: `Gen`; if `Active`: `Phase, Task, Trail, S, PrevS, Carrying, Lane, WobblePhase, Timer, Slot`.

Inactive slots fold their `Gen` because it decides the next id handed out.

### 6.6 Game-layer contract (for the M1 Game work, not Sim)

- `SimRunner` builds `new World(cfg, itemDatabase.ToDefs(), nestPos, seed)`, then calls
  `SpawnItem` for each `WorldItemView` in the order of §1.1 and stores the returned id on it.
- `PlayerPos` = the player's world `(x, z)` every frame.
- Mark → `MarkTrailCancel` if `World.Draft.Active`, else `MarkTrailStart(id.Index, id.Gen)`.
  Tap → `TapAnt(−1)`.
- Views interpolate with `AntPosition(i, Alpha)` / `ItemPosition(i, Alpha)`.

There is no save in M1. M2's save must serialize exactly the state folded in §6.5 plus
`Rng.State`, under a version field.

---

## 7. Tests (`Assets/Tests/EditMode`)

### `SimTestKit.cs` (shared helpers)

- `Defs(int antsRequired = 6, int tier = 1, float food = 60, float spoil = 0, float radius = 1.2f)` →
  `ItemDef[]` with only `SugarCube` set.
- `NewWorld(int seed = 1, SimConfig cfg = null, ItemDef[] defs = null)` → nest at `(0, 0)`.
- `Step(World w, ref SimInput input, List<SimEvent> log)` → one tick, append events to `log`,
  clear commands.
- `LayStraightTrail(World w, ref SimInput input, ItemId item, List<SimEvent> log)` → put the
  player at `item.Pos + (−2, 0)`, send `MarkTrailStart`, then move `PlayerPos` 0.5 u/tick
  toward the nest until `TrailLaid`; returns the tick it was laid.
- Standard geometry: cube at `(40, 0)`, so a straight trail has `Length = 40`.

### `ClockTests` (update)

- `Config()` sets `StartTimeOfDay01 = 0f`; existing tests unchanged.
- `StartsAtDawnByDefault`: default config world, after 1 tick `IsNight == false` and
  `TimeOfDay01 ≈ 0.25 + 1/3600` (within 1e-5).
- `FirstNightStartsAfter180Seconds`: default config; `NightStarted` is raised on tick 1800,
  not before.

### `ItemTests`

- `SpawnItem_IsLyingAtPosition`: state `Lying`, `Pos` as given, `Gen == 1`.
- `SpawnItem_WithoutDefThrows`: `SpawnItem(Seed, …)` with `Defs()` throws `ArgumentException`.
- `Constructor_RejectsMisindexedDefs`: a def at index 1 with `Kind = Leaf` throws.
- `SlotReuse_BumpsGen_StaleIdFails`: spoil an item (spoil = 1 s) away, tick once more, spawn
  again: same `Index`, `Gen == 2`, `TryGetItem(oldId)` is false.
- `Spoil_GoesGoneOnceAndFreesNextTick`: spoil = 10 s; `Gone` on tick 100 (±1), exactly one
  `ItemSpoiled`; the next tick the slot is inactive.
- `SugarCube_NeverSpoils`: 20 000 ticks, still `Lying`, no `ItemSpoiled`.

### `ColonyUpkeepTests`

- `StartingCounts`: Idle 8, Nursing 4, Brood 4, Food 20, Tier 1, Population 12.
- `UpkeepOverOneDay`: 3600 ticks no input → `Food == 14` within 1e-3.
- `UpkeepCountsAntsOutside`: `AntsRequired = 12` (so no haul starts), lay a trail; all 8 idle
  ants go out; `Food == 14` within 1e-3 at tick 3600.
- `FoodClampsAtZero_RaisesFoodRanOutOnce`: `StartingFood = 1`, 2 days → `Food == 0`,
  exactly one `FoodRanOut`.
- `TierFromPopulation`: `StartingIdle` 20/21/71 with 4 nursing → Tier 1/2/3; no `TierChanged`
  at construction.
- `NursingIsNeverRecruited`: `StartingIdle = 0`; lay a trail; 600 ticks → no `AntLeftNest`,
  `Nursing == 4`.

### `TrailTests`

- `CumulativeLengths`: points `(0,0) (3,4) (3,10)` → `Cum = [0, 5, 11]`, `Length = 11`.
- `SampleInteriorAndClamped`: `Sample(5) = (3,4)`, `Sample(8) = (3,7)` with tangent `(0,1)`;
  `Sample(−1) = (0,0)`; `Sample(99) = (3,10)`.
- `DecayIsExponential`: unreferenced trail at 1.0, 100 ticks → `exp(−0.5)` within 1e-4.
- `UnreferencedTrailFades`: removed with one `TrailFaded` on tick 783 (±1).
- `ReferencedTrailHoldsAtFloor`: trail with a live task, 2000 ticks, no ants (Idle 0) →
  still active, `Strength == TrailFloor`.
- `ReinforceAddsPerPassAndCaps`: `Length 20`, `Strength 0.5`, `Reinforce(10, cfg)` → 0.55;
  `Reinforce(1000, cfg)` → 1.0.
- `Draft_SpacingAndOrientation`: `LayStraightTrail` → `Points[0] == NestPos`,
  last point `== item.Pos`, all interior gaps ≥ `TrailSampleDistance`, `Length` = 40 ± 1e-3,
  `PlayerLaid`, and on the `TrailLaid` tick `Strength == PlayerTrailStrength · decayPerTick` within
  1e-6 (the trail is laid in step 2 and decayed in step 3 of the same tick).
- `Draft_TooLongAbandons`: walk 401 u away from the nest → `TrailAbandoned(TooLong)`, no trail,
  item `Lying`.
- `Draft_CancelLeavesItemLying`: `MarkTrailCancel` → `TrailAbandoned(Cancelled)`, item `Lying`.
- `SecondPlayerTrailReplacesRecruitingOne`: lay A, wait until 2 ants are out, lay B → task A
  `Aborted` with `Replaced`, A's ants `GoingHome`, item A `Lying`, trail A `PlayerLaid == false`,
  task B `Recruiting`.
- `SecondPlayerTrailLeavesHaulAlone`: lay A, run until A is `Hauling`, lay B → A still
  delivers (`ItemDelivered` for A).

### `RecruitmentTests`

- `FirstRecruitAfter2Point2Seconds`: first `AntLeftNest` on tick `T + 21` (±1), where `T` is
  the `TrailLaid` tick.
- `RateScalesWithStrength`: `PlayerTrailStrength = 0.5` → first recruit on `T + 44` (±1).
- `CapAtAntsRequired`: player away from the item; run to delivery → exactly 6 `AntLeftNest`
  for the task; Idle never below 2.
- `IdleLimitsRecruits`: `StartingIdle = 3` → exactly 3 recruits; after 3000 ticks the task is
  still `Recruiting`, item `Claimed`; `RecruitAccum <= 1` on every tick.
- `TapSendsBatchFromNest`: player at nest, `TapAnt(−1)` → 3 `AntLeftNest` that tick,
  `Idle == 5`, `Taps == 1`.
- `TapRespectsCapAndLimit`: two taps → 6 assigned; a third → `CommandRejected(NothingToSend)`.
  With `AntsRequired = 20`, `StartingIdle = 20`: the fourth tap → `TapLimit`.
- `TapAwayFromNestRejected`: player 10 u from the nest → `CommandRejected(NotAtNest)`, no spawn.
- `PlayerCountsAsOne`: player held within 4.2 u of the cube → `ItemPickedUp` on the tick after
  the 5th ant arrives, with exactly 5 ants `Hauling`; the 6th later joins with `Slot == 5`
  before delivery.
- `PlayerAwayNeedsSix`: player at nest → `ItemPickedUp` only after 6 ants are `AtTarget`.
- `PlayerAloneCannotHaul`: `AntsRequired = 1`, player at the item, `StartingIdle = 0` → no
  `ItemPickedUp` in 600 ticks.
- `TierGate`: `TierRequired = 2` → `MarkTrailStart` rejected `TierTooLow`, no draft.

### `HaulTests`

- `HaulSpeedDayAndNight`: item `S` falls 0.18 u per tick by day (±1e-4); with
  `StartTimeOfDay01 = 0` (night) 0.108 u per tick.
- `DeliveryAddsFoodOnce`: run the loop with the player at the nest; exactly one `ItemDelivered`
  (B = 60) and one `TaskComplete`; `Food == 20 + 60 − upkeep(ticks)` within 1e-3; item
  `Delivered` that tick and inactive the next; task `Done` then freed.
- `HaulersDepositThenReturn`: after delivery the 6 carriers are `Depositing`; 10 ticks later all
  are gone, 6 `AntReturned`, `Idle == 8`, `AntsOutside == 0`.
- `SpoiledValueScales`: `spoil = 100 s`; haul delivered at item `Age = A` → food added
  `60 · (1 − A/100)` within 1e-3.
- `ItemGoneAbortsTask`: `spoil = 15 s`; the standard trail is laid at ~8 s and recruits leave
  from ~10 s, so ants are outbound when it spoils → `TaskAborted(ItemGone)`, every ant returns, `Idle == 8`.
- `FinishedTrailFades`: after delivery and deposit the trail fades and `TrailFaded` is raised
  within 1000 ticks; slot inactive afterwards.
- `AntsStayNearTrail`: for every outbound ant on every tick, distance from
  `trail.Sample(S)` ≤ 0.85 + 1e-4.

### `CommandTests`

- `MarkStartRejections`: out of reach → `OutOfReach`; stale gen → `NoSuchItem`; `Claimed` item →
  `ItemNotAvailable`; during a draft → `DraftActive`. No state change in each case.
- `PointEndCancelWithoutDraft`: each → `NoDraft`.
- `MarkEndAwayFromNest`: draft active, player 20 u from the nest → `NotAtNest`, draft still active.
- `MarkTrailPointAppends`: draft active, `MarkTrailPoint((30,5))` → `Draft.Count` +1.
- `NotImplementedCommandsAreInert`: `PickUp`, `Deposit`, `StartBuild` each raise
  `CommandRejected(NotImplemented)`; `StateHash` equals a control world ticked identically
  without them.
- `CommandsApplyInListOrder`: `[MarkTrailStart, MarkTrailCancel]` in one tick → draft inactive,
  `TrailMarkStarted` precedes `TrailAbandoned` in the event list.
- `WorldDoesNotClearCommands`: the list has the same count after `Tick` as before.

### `DeterminismTests`

Script (identical for every world): every 1200 ticks, at cycle tick 0, `SpawnItem(SugarCube,
(40, cycle % 2 == 0 ? 15 : −15))`, player placed 2 u from it, `MarkTrailStart`; the player then
walks to the nest at 0.5 u/tick along a path bent by `y += 3·sin(x/5)`; at cycle tick 200,
`TapAnt(−1)`; the player stays at the nest for the rest of the cycle.

- `SameSeed_IdenticalHashAndPositions`: two worlds seed 2024, 20 000 ticks. On every tick:
  equal `StateHash()`, equal `AntsOutside`, and for every active ant `AntPosition(i)` equal
  bit for bit. At the end: at least 14 `ItemDelivered` events (the run is not idle).
- `DifferentSeed_Differs`: seed 2025 under the same script → end hashes differ and the first
  spawned ant's `Lane` differs.
- `HashCoversState`: on a mid-haul world, changing (via test hooks) Food, one ant's `S`, one
  trail's `Strength`, one item's `S`, each alone, changes `StateHash()`.
- The existing `ClockTests.TwoWorldsWithSameSeedShareStateHash` stays.

---

## 8. Worked M1 timeline (default numbers)

Inputs outside the simulation:

**Assumed** — reference layout: nest at the patch's (0, 0), sugar cube 50 u away in a
straight line; the player's walk home is 58 u. Player walk speed 5 u/s (Game-layer
`PlayerMotionAsset`, not a sim value). Cheap to change; it is scene and controller data.

Sim inputs: `L = 58`, outbound distance `L − 1.2 = 56.8`, `v = 3 u/s`, haul `1.8 u/s`,
recruit `0.5 · Strength` per s, decay 0.05/s. All in daylight (night starts at 180 s).
Times below come from stepping the rules above at 0.1 s.

| t (s) | What happens | Arithmetic |
|---|---|---|
| 0 | Player at nest. 06:00 game time. | |
| 0–45 | Explore, find the cube. | budget, not simulated |
| 45 | Mark at the cube (`TrailMarkStarted`). | |
| 57 | Player crosses the 4 u nest ring: `TrailLaid`, `TaskCreated`. | 54 u / 5 u/s ≈ 11 s |
| 59, 62, 64, 67, 70, 73 | Six recruits leave (`AntLeftNest`). Strength 0.90 → 0.62. | ∫0.5·S dt reaches 1, 2, … 6 |
| 78 → 92 | They arrive at the cube, 2–3 s apart. | 56.8 / 3 = 18.9 s each |
| 92 | Sixth ant arrives; `ItemPickedUp`. Strength 0.44. | |
| 92 → 124 | Party hauls the cube home. | 58 / 1.8 = 32.2 s |
| **124** | `ItemDelivered` (+60), `TaskComplete`. | |
| 125 | Haulers rejoin Idle (`AntReturned` × 6). | `DepositSeconds` 1 |
| ~185 | Trail fades (`TrailFaded`). | from 0.39: ln(0.39/0.02)/0.05 ≈ 59 s |

Loop from Mark to delivery: **79 s**. Total with 45 s of exploring: **2 min 04 s**, 56 s
under the 3-minute target. Variants: player walks back and stands at the cube −3 s; one
tap at the nest on arrival −9 s; both −12 s. Food at delivery:
`20 − 12 · 0.5 · 124/360 + 60 = 77.9`. Idle never drops below 2.

The target is met up to roughly 100 s of exploring (the slack), or a 100 u trail
(loop ≈ 125 s with a 20 s walk home). If it ran at night (speeds × 0.6) the same loop would
take 114 s instead of 79 s.

**No existing `SimConfig` default changed.** New fields are listed in §0.

---

## 9. Known degenerate strategies and failure modes

- **Tapping makes the trail optional for small finds.** Two taps at the nest send all six
  ants at once; trail strength then only matters for how fast the trail fades. Acceptable in
  M1 (tap is part of the loop); if trails should matter, lower `TapBatch` or
  `MaxTapsPerTask` to 1.
- **Stale tasks never die.** A trail to an unhaulable item (idle pool too small) holds at the
  floor and trickles one recruit per 100 s forever. Harmless with one cube; M2 needs task
  expiry or a give-up rule.
- **Task index order is recruitment priority.** With several recruiting tasks, a lower slot
  index fills first each tick. Irrelevant with one player trail; M2 needs a fair order.
- **Player-at-item is always worth it** but only by one ant-arrival (~3 s); not exploitable.
- **Replacing your own trail cancels it.** Marking a second item is the only way to call
  ants back. Intended for M1; a dedicated recall is a later design.
