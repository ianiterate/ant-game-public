# Simulation spec — M3 weather and threats

Implementation spec for M3 in `Assets/Scripts/Sim`. It builds on [SIM.md](SIM.md) (M1) and
[SIM_M2.md](SIM_M2.md) (M2), which match the code, and replaces only what it says it replaces.
[GAME.md](GAME.md) is what the game is; this file is how the simulation does it. Run
`/open-questions` for everything marked below.

Units as before: distance in **u** (1 u = 1 cm), time in **s**, rates per second unless a name says
per day or per night. `dt` = `TickSeconds` (0.1 s). A day is 360 s; night is the first and last
quarter of it, so a night is 180 s and a day's daylight is 180 s.

M3 adds: weather (§1), the recall that rain and raids share (§2), how an ant outside dies (§3),
spiders (§4) and defending against them (§5), birds (§6), raids from a rival colony (§7), and the
player ant's danger, death and respawn (§8). Threats are generic — spider, bird, rival ants —
because species fidelity is still undecided in GAME.md. Nothing below names a species or
depends on one.

---

## 0. Ground rules and what changes

All earlier ground rules hold (SIM.md §0, SIM_M2.md §0): slot arrays, index order, no allocation
per tick, gen bumped when a slot is taken, terminal states freed next tick by the owning system.

### 0.1 Tick order (replaces SIM_M2.md §0.1)

```csharp
public void Tick(in SimInput input)
{
    Events.Clear();
    StepClock();                                        // 1  clock; DayStartedThisTick
    WeatherSystem.Step(this, ref _hazardRng);           // 2  weather sample (§1.2)          NEW
    CommandSystem.Step(this, in input, ref _rng);       // 3  + MarkThreat, Alarm; PlayerDown rejects
    TrailSystem.Step(this);                             // 4  decay (× rain), floor, fade
    TaskSystem.Step(this, in input, ref _rng);          // 5  recall (§2), hauls, defends (§5), builds
    AntSystem.Step(this);                               // 6  + defenders muster and join fights
    ItemSystem.Step(this);                              // 7  unchanged
    SpawnSystem.Step(this, ref _rng);                   // 8  × weather spawn factor (§1.3)
    ColonySystem.Step(this);                            // 9  unchanged
    ThreatSystem.Step(this, in input, ref _hazardRng);  // 10 schedulers, raid, spiders, bird, player (§9)  NEW
    DayStartedThisTick = false;
}
```

Weather runs before trails and tasks, so a tick that starts rain already washes and recalls.
Threats run after the colony, as SIM.md §6.3 reserved. Consequences engineers rely on:

- A worker killed in step 10 leaves `Population` this tick; the colony's tier, peak and death
  checks see it in the next tick's step 9. A colony killed by a raid raises `ColonyDied` one tick
  after the last worker dies.
- Rain or a raid that begins in step 2 or step 10 recalls ants in step 5 of the same or next tick.
- An ant that reaches the muster point in step 6 counts toward the defend party in the next
  tick's step 5, exactly as an ant arriving at a find does for a haul.

### 0.2 New files

`Sim/Weather/`: `Weather.cs` (`Weather`, `WeatherRow`, `WeatherState`), `WeatherSystem.cs`.
`Sim/Threats/`: `ThreatKind.cs` (`ThreatKind`, `SpiderState`, `RaidPhase`), `Spider.cs`,
`BirdState.cs`, `RaidState.cs` (`RaidState`, `RivalState`), `PlayerState.cs`, `ThreatSystem.cs`
(schedulers, raid, spiders, bird, player), `ThreatQueries.cs` (the read API of §14, as a `World`
partial). `Core/Ids.cs` gains `SpiderId`. The plan's `EventScheduler` is folded into
`ThreatSystem`: three accumulators do not justify a file.

### 0.3 A second random stream

`World` gains a second `Rng`, the **hazard stream**: `_hazardRng = Rng.ForHazards(seed)`, where
`internal static Rng ForHazards(int seed) => new Rng(seed ^ 0x5EED_A2A2)`. Weather and threats draw
only from it; everything M1 and M2 draw (ant lanes, find placement) stays on the main stream.

Why: weather and threats must be tunable without moving every find in the garden. With one
stream, changing a bird rate would change where every later sugar cube lands, and a balance A/B
run with the same seed would differ in ways that have nothing to do with the change.

**Assumed** — two streams, both from the one seed. This departs from the "one seeded Rng"
convention in `unity-conventions`. Cost to change: cheap before implementation; afterwards it
changes every hash and the save's shape.

### 0.4 Switches and calm tests

`SimConfig` gains `WeatherEnabled` and `ThreatsEnabled` (both `true`). Off means the system does
nothing, makes no draws and raises no events: the weather stays Clear, no threat arrives, the
player cannot die. They exist so earlier tests keep earlier rules, and for debugging. They are
not a game mode.

`SimTestKit`: `M1Config` sets both false. `Rich()`, `Fast()` and a new `Calm()` (the default config
with both false) are used by every M2 test, so M2's arithmetic is unchanged. Every M2 test that
builds `new SimConfig {…}` becomes `new CalmConfig {…}` (a test-only subclass setting both false),
the same pattern as `M1Config`. M3 tests use the default config or `Hot()` (§15).

### 0.5 `SimLimits` additions

```csharp
public const int MaxSpiders = 4;            // SimConfig.SpiderMax may not exceed it
public const float MusterMargin = 2f;       // u between a spider's reach and the defenders' muster point
public const float FightRingRadius = 2f;    // u: fighters stand on this ring around the spider (§14.2)
public const float SpiderRingMaxDistance = 90f;  // u: outer edge of the fallback ring for dens (§4.2)
```

Both are geometry, not balance, so they are not in `SimConfig`.

One bird and one raid at a time, so neither has a slot array.

### 0.6 `SimConfig` additions

Existing fields keep their meaning and defaults. Season-indexed arrays are Spring, Summer, Autumn,
Winter (length 4). Tier-indexed arrays have length `TierThresholds.Length + 1`.

| Field | Default | Meaning |
|---|---|---|
| **Weather** | | |
| `WeatherEnabled` | `true` | §0.4. |
| `WeatherSampleSeconds` | `90f` | Weather changes only on multiples of this (a quarter day). |
| `WeatherTransitions` | §1.1 table | `WeatherRow[12]`, row `season · 3 + from`; each row `{Clear, Overcast, Rain}` sums to 1. |
| `WeatherMinDwell` | `{2, 1, 1}` | Samples a state lasts at least, by `Weather`. |
| `WeatherSpawnFactor` | `{1, 0.8f, 0.8f}` | Find spawn rate multiplier, by `Weather`. |
| `RainTrailDecayFactor` | `6f` | Trail decay multiplier while it rains. |
| **Threats, shared** | | |
| `ThreatsEnabled` | `true` | §0.4. |
| `ThreatJitter` | `0.5f` | Each threat interval is the mean × `[1 − j, 1 + j)`. |
| `KillTrailFactor` | `0.7f` | A trail's strength is multiplied by this when a worker is taken on it (§3.3). |
| `ThreatDangerPerSecond` | `0.35f` | Player danger gained per second within reach of a spider or a raid column (§8). |
| `DangerRecoverPerSecond` | `0.2f` | Player danger lost per second out of reach. |
| `PlayerRespawnSeconds` | `10f` | Delay before the player returns from the nest (§8). |
| `ThreatMarkMargin` | `6f` | u beyond a spider's reach from which the player can mark it. |
| **Spiders** | | |
| `SpidersPerNight` | `{0.1f, 0.25f, 0.2f, 0}` | Mean arrivals per night, before the tier factor. |
| `SpiderTierFactor` | `{0.5f, 1, 1.25f}` | Arrival multiplier by colony tier. |
| `SpiderMax` | `2` | Spiders present at once. |
| `SpiderStayDays` | `2f` | A spider leaves this long after arriving, unless driven off first. |
| `SpiderReach` | `4f` | u. Workers and the player within this of a spider are in danger. |
| `SpiderPatrolRadius` | `6f` | u. Radius of the loop it walks around its den. |
| `SpiderSpeed` | `1f` | u/s along the loop. Not slowed at night. |
| `SpiderMoveSpeed` | `3f` | u/s when moving to a new den. |
| `SpiderKillSeconds` | `10f` | One worker taken per this many seconds of workers being within reach. |
| `SpiderNestClearance` | `25f` | u. A den is never closer to the nest than this. |
| `SpiderRelocateSeconds` | `30f` | Seconds with nobody in reach before a spider follows the traffic to the busiest trail. |
| `SpiderDefenders` | `6` | Party size that attacks a spider (the player counts as one, §5.3). |
| `SpiderToughness` | `50f` | Defender-seconds of fighting that drive a spider off. |
| `SpiderFightKillSeconds` | `9f` | One defender lost per this many seconds of fighting; a full party wins first. |
| **Birds** | | |
| `BirdsPerDay` | `{0.25f, 0.5f, 0.4f, 0}` | Mean birds per day, arriving in dry daylight only. |
| `BirdWarningSeconds` | `5f` | The shadow's warning before the strike. |
| `BirdStrikeRadius` | `3f` | u around the shadow at the strike. |
| `BirdMaxKills` | `3` | Workers a strike takes at most. |
| `BirdNestClearance` | `10f` | u. Birds do not target workers this close to the nest. |
| `AlarmReach` | `10f` | u from the shadow within which the player can raise the alarm. |
| **Rival and raids** | | |
| `RivalStartStrength` | `30f` | The rival colony's strength on day 0, in raiders it could field. |
| `RivalGrowthPerDay` | `0.15f` | Logistic growth rate. |
| `RivalCapacity` | `100f` | Logistic ceiling. |
| `RaidsPerDay` | `{0.05f, 0.15f, 0.2f, 0}` | Mean raids per day. |
| `RaidCommitByTier` | `{0.15f, 0.25f, 0.35f}` | Fraction of the rival's strength sent, by *your* tier. |
| `RaidMinRaiders` | `3` | A raid smaller than this is not sent. |
| `RaiderSpeed` | `2f` | u/s, × `NightSpeedFactor` at night. |
| `RaidReach` | `5f` | u around the raid column's head that is dangerous to the player. |
| `RaidKillPerSecond` | `0.02f` | Kills per second per fighter, both sides (§7.4). |
| `RallyFactor` | `1.5f` | Defenders' kill rate multiplier while the player is at the nest. |
| `RaidBreakFraction` | `0.5f` | A side breaks when it has lost this fraction of its starting count. |
| `RaidLootPerRaiderSecond` | `0.15f` | Food each raider at the nest carries off per second. |
| `RaidBreachLootSeconds` | `30f` | How long raiders loot unopposed after a breach. |

**No existing default changes.** §16.4 explains why none is needed.

The constructor validates, throwing `ArgumentException`: `WeatherTransitions.Length == 12`, every
entry ≥ 0, every row sums to 1 within 1e-3; `WeatherMinDwell` and `WeatherSpawnFactor` length 3,
dwell ≥ 1; season arrays length 4; tier arrays length `TierThresholds.Length + 1`;
`1 <= SpiderMax <= MaxSpiders`; `SpiderDefenders >= 1`; `0 < RaidBreakFraction <= 1`;
`NestPos` inside `SpawnArea` (raids enter from its edge).

**Assumed** — every number in this table. All are cheap to change (data), and §16 shows what they
add up to.

---

## 1. Weather

### 1.1 State and transitions

```csharp
public enum Weather { Clear = 0, Overcast, Rain }
[Serializable] public struct WeatherRow { public float Clear, Overcast, Rain; }
public struct WeatherState { public Weather Current, Next; public int Dwell; }
```

`Dwell` is the number of sample intervals `Current` has held, counting the one in progress.
`Next` is already decided: it is the weather for the interval after this one, which is what makes
a forecast possible.

Default transitions (from-state rows; columns Clear, Overcast, Rain):

| Season | from Clear | from Overcast | from Rain |
|---|---|---|---|
| Spring | .75 / .25 / 0 | .35 / .40 / .25 | .20 / .50 / .30 |
| Summer | .85 / .15 / 0 | .50 / .30 / .20 | .30 / .50 / .20 |
| Autumn | .70 / .30 / 0 | .30 / .40 / .30 | .15 / .50 / .35 |
| Winter | .60 / .40 / 0 | .25 / .70 / .05 | .30 / .60 / .10 |

Clear never goes straight to rain, so every rain is preceded by at least one overcast interval
and is forecast at least 90 s ahead. With the minimum dwell `{2, 1, 1}` the long-run shares,
computed from the chain including dwell, are:

| | Clear | Overcast | Rain | Mean rain spell | Rain spells per season |
|---|---|---|---|---|---|
| Spring | 61% | 29% | 10% | 129 s | 2.9 |
| Summer | 78% | 18% | 4% | 112 s | 1.4 |
| Autumn | 52% | 33% | 15% | 138 s | 3.9 |
| Winter | 47% | 50% | 3% | 100 s | 1.0 |

**Assumed** — three states sampled every quarter day, rain always via overcast, a minimum clear
spell of two intervals, wet springs and autumns. Cheap to change: data.

### 1.2 `WeatherSystem.Step` (tick step 2)

`SampleTicks = RoundToInt(WeatherSampleSeconds / TickSeconds)` (900), computed once. If
`!WeatherEnabled`, return. Otherwise, on ticks with `Tick % SampleTicks == 0`:

```
prev = Current
Current = Next
Dwell = Current == prev ? Dwell + 1 : 1
u = hazardRng.NextFloat()                     // always exactly one draw per sample
if Dwell < WeatherMinDwell[Current]: Next = Current
else: row = WeatherTransitions[season · 3 + Current]
      Next = u < row.Clear ? Clear : u < row.Clear + row.Overcast ? Overcast : Rain
if Current != prev: raise WeatherChanged(A = Current, B = prev)
if Next != Current: raise WeatherForecast(A = Next, B = Current)
```

`season` is the clock's season on the sample tick. A world starts with `Current = Next = Clear`,
`Dwell = 1`. The first sample (tick 900) therefore stays clear with `Dwell = 2` and makes the first
real draw; the earliest overcast is tick 1 800 and the earliest rain tick 2 700 (270 s). The
tutorial's first haul (about 2 minutes, SIM.md §8) is always dry.

### 1.3 Effects

| | Clear | Overcast | Rain |
|---|---|---|---|
| Find spawns (SpawnSystem rate ×) | 1 | 0.8 | 0.8 |
| Trail decay (TrailSystem) | × 1 | × 1 | × `RainTrailDecayFactor` |
| Recruiting, taps, workers going out | yes | yes | **recalled** (§2) |
| Hauling parties | move | move | move |
| Spiders hunt, move, arrive | yes | yes | no; they hide (§4.7) |
| Birds | by day | by day | none |
| Raids launched | yes | yes | held until dry |

Night already slows everything outside to 0.6× (SIM.md). Weather does not change speeds.

**Washout.** `RainDecayPerTick = exp(−TrailDecayPerSecond · RainTrailDecayFactor · dt)` = 0.97045,
computed once. An unreferenced trail at 1.0 fades in 131 ticks (13.1 s) in rain against 783 dry.
A trail still used by a task holds at the floor (SIM.md §3.2), so a rain of any length leaves a
recruiting trail at 0.02, which recruits one worker per 100 s. That is the cost of rain, and the
next section is how the player pays it back.

**`TrailWashed`.** The first time in a rain that the player's haul trail or a defence trail is held
at the floor, TrailSystem sets `Trail.Washed = true` and raises `TrailWashed(A = trail index)`, so the
HUD can say "walk it again" at the moment it matters. `Washed` is cleared on a dry tick, by a
re-walk (§1.4, §5.1), and when a trail slot is taken. It is state: hashed and saved (§12, §13).

**Assumed** — rain recalls workers on the way out and leaves carrying parties alone; overcast and
rain cut find spawns by a fifth. Cheap to change.

### 1.4 Walking a trail again (replaces SIM.md §3.4's "a weak trail cannot be refreshed")

`MarkTrailStart` on an item that is `Claimed` is accepted when the item's task is the player's
current haul (the task `TapAnt(−1)` would pick among hauls) and is `Recruiting`. The draft opens with
`Draft.Refresh = true`. The player walks home as usual. On completion (the M1 Complete, before
step 1):

- If the item is still `Claimed` by that task and the task is `Recruiting`:
  `trail.Strength = Max(trail.Strength, PlayerTrailStrength)`; raise
  `TrailRefreshed(A = trail index, B = task index, P = item pos)`. The walked path is discarded:
  the trail keeps its points, so ants on it are undisturbed. No trail or task slot is taken.
- Otherwise abandon with `ItemUnavailable`.

`TooLong` and `TooShort` apply to the walk as for any draft. Any other `Claimed` item still rejects
with `ItemNotAvailable`.

**Assumed** — re-walking renews strength but keeps the old path. Cheap to change; replacing the
path would have to move every ant already on it.

### 1.5 Read API

`WeatherState WeatherNow`; `float SecondsToWeatherSample` = `(SampleTicks − Tick % SampleTicks) · dt`;
`bool Raining`. The HUD shows "rain soon" whenever `Next == Rain && Current != Rain`.

---

## 2. Recall (rain and raids)

`bool RecallActive => (WeatherNow.Current == Rain) || Raid.Active`. While it holds:

- **Recall pass**, first in `TaskSystem.Step` after freeing terminal tasks and before `Recount`:
  every active ant in `Outbound` or `AtTarget` whose task is a `Haul` or `Defend` in `Recruiting`
  gets `Phase = GoingHome`, `Task = default` (keeping `Trail` and `S`, as Abort does).
  `Hauling` and `Fighting` ants are untouched.
- Recruiting tasks do not recruit: `RecruitAccum = 0`, no spawns. Haul and fight starts are not
  blocked, but nobody is left at a target to start one.
- `TapAnt` is rejected with `Recalled`.

When the recall ends the tasks are still `Recruiting` and recruit again at their trail's strength.
After rain that strength is at the floor until the player walks the trail again (§1.4). After a
dry raid it is wherever decay left it.

The nest's defence against a raid is this recall plus the fight in §7.4. It needs no task slot:
the UI presents it as the colony's defend order, but in the simulation it is the raid record.

**Assumed** — the recall is automatic for raids as well as rain. The player's part in a raid is
being at the nest (§7.4). A panel order to call workers home would belong to the open question of
how much strategy game there is (undecided in GAME.md), and this does not pre-empt it.

---

## 3. How a worker outside dies

### 3.1 `ThreatSystem.KillAnt(w, index, ThreatKind cause, bool weakenTrail)`

1. If the ant is `Hauling`: note its item.
2. `Active = false`. No `Idle++`: the worker is gone. `Stats.WorkersKilled++`.
3. If `weakenTrail` (a spider's hunting kill; not a fight, §5.4, and not a bird strike, which
   weakens its trail once, §6.2): the ant's trail, if it resolves, `Strength *= KillTrailFactor`.
   TrailSystem's floor rule applies next tick.
4. If the noted item now has no active carrier: **drop it** (§3.2).

The caller raises `AntsLost(A = count, B = (int)cause, P = position)` once per cause per tick.

### 3.2 A party wiped out drops its find

The item becomes `Lying` at its current `Pos`, `S = PrevS = 0`, `Task = Trail = default`,
`Expires = false`. Its task is aborted with the new reason `PartyLost`. Ants already outbound on
that task turn home through the M1 Abort path. The find keeps its `Age`, so a dead insect keeps
spoiling, and it can be marked again.

A party with at least one carrier left keeps going at full speed: M1's rule that a lifted haul
never stalls still holds while anyone carries it.

**Assumed** — a dropped find never expires (it may still spoil). Cheap to change. Expiry by
`Age` would remove most dropped finds at once, since they are usually old.

### 3.3 Kills weaken trails

Each worker taken by a spider or a bird multiplies its trail's strength by 0.7. A spider sitting
on a busy trail therefore slows that trail's recruiting as well as taking workers. This is the
spider's main cost to the colony's food (§16.2). It is negative feedback: a weaker trail sends fewer
workers past the spider, so the spider takes fewer.

**Assumed** — one factor for every kill on a trail. Cheap to change.

---

## 4. Spiders

### 4.1 Record

```csharp
public enum ThreatKind { None = 0, Spider, Bird, Raid }
public enum SpiderState { None = 0, Patrolling, Moving, Fighting, Gone }

public struct Spider
{
    public SpiderId Id; public bool Active; public SpiderState State;
    public Vector2 Den;            // centre of its loop (or of the loop it is moving to)
    public float Angle;            // radians on the loop
    public int Dir;                // +1 or −1
    public Vector2 Pos, PrevPos;   // where it is; PrevPos for interpolation
    public Vector2 MoveTo;         // while Moving: Den + R·(cos Angle, sin Angle)
    public float Age;              // s since arrival
    public float KillAccum;        // [0, 1+): seconds of exposure / SpiderKillSeconds
    public float Quiet;            // s with nobody in reach
    public float Wounds;           // defender-seconds this fight
    public float FightKillAccum;   // this fight
    public TaskId Defend;          // the Defend task aimed at it, if any
}
```

`World` owns `Spider[MaxSpiders]`. Spiders are lightweight: a few floats, no pathing.

### 4.2 Arrival (scheduler, §9 step 3)

`World` owns `float[] ThreatProgress, ThreatThreshold`, length 4, indexed by `ThreatKind`. No
scheduler advances on day 0, the tutorial day (`Clock.Day == 0`). Thresholds are drawn
`hazardRng.Range(1 − ThreatJitter, 1 + ThreatJitter)` at construction if `ThreatsEnabled` (in the
order Spider, Bird, Raid: the hazard stream's first three draws) and after each use.
`NightSeconds = (NightEnd01 + 1 − NightStart01) · DayLengthSeconds` (180) and
`DaySeconds = DayLengthSeconds − NightSeconds` (180) are derived once.

A scheduler whose rate is 0 this season (winter, by default) neither advances nor fires an arrival
held at its threshold from before, exactly as SpawnSystem does for finds. Progress resumes in
spring where it stopped.

Spiders arrive **at night**, and not in rain:

```
if Clock.IsNight and !Raining:
    progress[Spider] += SpidersPerNight[season] · SpiderTierFactor[Tier − 1] · dt / NightSeconds
    if progress >= threshold:
        if active spiders >= SpiderMax: progress = threshold        // no banking
        else: TryPlaceSpider(); progress −= threshold; threshold = draw
```

**`TryPlaceSpider`** — the den goes on the **busiest trail**: the active trail with the most active
ants whose `Trail` is it (ties: lowest index), if it has at least one ant. Up to
`SpawnPlacementTries` tries, one draw each: `s = L · (0.3 + 0.5 · u)`, `den = trail.Sample(s)`.
Accept if `den` is in `SpawnArea`, at least `SpiderNestClearance` from the nest, and at least
`2 · (SpiderPatrolRadius + SpiderReach)` (20 u) from every other spider's `Den`. If there is no busy
trail or every try fails, fall back to the ring: up to `SpawnPlacementTries` tries of two draws
(`a`, `r²` uniform in `[SpiderNestClearance², SpiderRingMaxDistance²]`), same acceptance. If that fails too, nothing
arrives (the arrival is used up). On success, two more draws: `Dir = u < 0.5 ? +1 : −1`,
`Angle = Range(0, 2π)`. Take the lowest free slot, `Gen++`, `State = Patrolling`,
`Pos = PrevPos = Den + R·(cos, sin)`. Raise `ThreatSpawned(A = Spider, B = slot, P = Den)`.

The den is on the trail, so the loop crosses it twice. The kill disk (radius 4 around a point on a
radius-6 loop) covers the trail 46% of the time.

**Assumed** — spiders come at night and settle where traffic is. Medium cost to change: the
placement rule is what makes spiders matter (§16.2). Random placement would leave most spiders
nowhere near a trail.

### 4.3 Patrol

`Patrolling`, not raining: `PrevPos = Pos`; `Angle += Dir · SpiderSpeed / SpiderPatrolRadius · dt`;
`Pos = Den + SpiderPatrolRadius · (cos Angle, sin Angle)`. One loop takes 377 ticks (37.7 s).

### 4.4 Hunting

`Patrolling` or `Moving`, not raining. Victims are active ants in `Outbound`, `AtTarget`,
`Hauling` or `GoingHome` (the phases out on a trail) within `SpiderReach` of `Pos`, measured at
`AntPosition(i)` (alpha 1) from the per-tick scratch buffer (§9).

```
if any victim in reach:
    Quiet = 0
    KillAccum += dt / SpiderKillSeconds
    if KillAccum >= 1: KillAnt(nearest victim; ties lowest index, Spider); KillAccum −= 1
else:
    Quiet += dt          // KillAccum is kept: exposure adds up across passes
```

So a spider takes one worker per 10 s that workers spend within its reach, however many are
there. That is handling time: a spider cannot eat a party at once. With workers walking a busy
trail through its loop, exposure is about 11% of the time (46% overlap × ~40% chance a worker is
under it × ~60% of the time it is on a trail in use), so about **4 workers a day** while it is
left alone. The 60% depends on §4.5: a trail lives only as long as one haul (about two minutes),
so a spider that stayed where it settled would sit by a dead trail most of its stay. That
figure drives §16; the bot run in §15 is the real check.

### 4.5 Following the traffic

When `Patrolling`, `Quiet >= SpiderRelocateSeconds`, not raining, and `Defend` does not resolve to
a live task: run the busy-trail placement of §4.2 (no ring fallback), additionally requiring the
new den to be at least 20 u from the current one. On success: `State = Moving`, `Den = new den`,
`MoveTo = Den + R·(cos Angle, sin Angle)`, `Quiet = 0`, raise `SpiderMoved(A = slot, P = Den)`.
On failure: `Quiet = 0` (try again in 30 s).

`Moving`: `Pos` steps toward `MoveTo` at `SpiderMoveSpeed`; on arrival `State = Patrolling`. It
hunts on the way (§4.4).

A spider aimed at by a Defend task never moves. The defenders' trail ends at its den.

With `SpiderRelocateSeconds = 30` a spider is on whatever trail the colony is using within about a
minute of the last one going quiet (30 s, plus the walk at 3 u/s). Routing around it does not
work for long; driving it off does. That is deliberate: hauls are short, so a predator that kept
its first den would be a hazard of the trail you had two minutes ago.

**Assumed** — spiders follow the colony's traffic. Cheap to change (one number); the cost of
changing it back is that undefended spiders stop mattering and Defend stops paying (measured: a
bot that switches finds every two minutes met a 120 s spider near traffic only 7–15% of its stay).

### 4.6 Leaving

`Age += dt` every tick in every state. When `Age >= SpiderStayDays · DayLengthSeconds` and
`State != Fighting`: `State = Gone`, raise `SpiderLeft(A = slot, P = Pos)`. The slot is freed at
the start of the next ThreatSystem step. A Defend task aimed at it aborts with `ThreatGone` in the
next TaskSystem step.

### 4.7 Rain

In rain a spider does not move, hunt, relocate or arrive. It is still drawn (hidden under cover)
and still ages. A fight already under way continues.

---

## 5. Defending against a spider

### 5.1 Marking a spider

The player defends the way they haul: press **Mark** near the spider and walk home. A new command
`MarkThreat(A = spider index, B = gen)`, validated in order:

| Check | Reject |
|---|---|
| player down (§8) | `PlayerDown` |
| a draft is active | `DraftActive` |
| the spider does not resolve, or is `Gone` | `NoSuchThreat` |
| `|PlayerPos − spider.Pos| > SpiderReach + ThreatMarkMargin` (10 u) | `OutOfReach` |
| it is `Fighting`, or its `Defend` task is live and not `Recruiting` | `DefendActive` |

If the spider's Defend task is `Recruiting`, the mark is a **re-walk** of the defence trail, as
§1.4 is for a find: `Draft.Refresh = true`, and completion sets that trail's strength to
`Max(Strength, PlayerTrailStrength)`, clears `Washed`, and raises `TrailRefreshed(A = trail,
B = task, P = den)` instead of laying a new trail. Without it a defence washed out by rain would
recruit one worker per 100 s until the spider left.

Effect: open the draft with `Draft.Spider = id`, `Draft.Item = default`, `Points[0] = spider.Den`,
`Count = 1`. Raise `TrailMarkStarted(A = −1 − slot, P = Den)` (a negative `A` means a spider).

Marking from 10 u keeps the player outside the 4 u reach, but the spider walks a loop of radius 6:
standing still that close is a risk the player takes knowingly.

**Complete** (at the nest, the M1 path): if the spider no longer resolves, is `Gone` or is
`Fighting`: abandon `ThreatGone`. Append `NestPos`. If `Length <= DefendStandoff + 1` (13 u):
abandon `TooShort`. No free trail or task slot: `NoFreeSlot`. Take a trail slot (nest first,
`Strength = PlayerTrailStrength`, **`PlayerLaid = false`**) and a task slot:
`Kind = Defend`, `Phase = Recruiting`, `Trail`, `Spider = id`, `PlayerLaid = false`. Set
`spider.Defend` to the task id. Raise `TrailLaid`, `TaskCreated(A = task, B = −1 − spider slot)`.

`DefendStandoff = SpiderPatrolRadius + SpiderReach + MusterMargin` = 12 u, derived.

A defend trail is not the player's one haul trail. It never replaces a haul trail and is never
replaced by one. At most one Defend task aims at a spider.

**Assumed** — a spider is defended by marking it, not by a tap near it or a panel order. This keeps
"the player's movement is the order", and it leaves GAME.md's open question of what a tap
away from the nest does untouched. Cheap to change at the command level; the copy and prompts
follow it.

### 5.2 The Defend task

`TaskKind` gains `Defend`, `TaskPhase` gains `Fighting`, `AntPhase` gains `Fighting` (all
appended). `ColonyTask` gains `SpiderId Spider`. The party size is `SpiderDefenders`, in the role
`def.AntsRequired` plays for a haul.

`TapAnt(−1)` picks the lowest-index `Defend` task in `Recruiting` if there is one, else the
player's haul as before. `MaxTapsPerTask` applies to each task separately.

**Assumed** — a tap at the nest serves a recruiting defence before the haul. Cheap to change.

### 5.3 `TaskSystem`, defend part (after hauls, before builds)

For each active task with `Kind == Defend`, `Phase` in {`Recruiting`, `Fighting`}:

1. Resolve the spider. If it fails or the spider is `Gone`: Abort `ThreatGone`; continue.
2. If `Recruiting`:
   - `musterS = trail.Length − DefendStandoff`; `muster = trail.Sample(musterS)`.
   - `playerHere = PlayerAlive && |PlayerPos − muster| <= ReachRadius`.
   - If `AtTarget >= 1 && AtTarget + playerHere >= SpiderDefenders`: **start the fight** (below).
   - Else recruit exactly as a haul (SIM.md §4.2), capped at `SpiderDefenders`, with nothing while
     `RecallActive`.

**Start the fight:** task `Phase = Fighting`. Spider `State = Fighting`, `Wounds = 0`,
`FightKillAccum = 0`. Every ant on the task in `AtTarget`: `Phase = Fighting`,
`Slot = task.Haulers++`. Raise `DefendStarted(A = task, B = spider slot, P = spider.Pos)`. The spider
stops where it is. The fight happens there, wherever it is on its loop.

`AntSystem`, `Outbound` with a Defend task: `musterS` as above. If the task is `Fighting` and
`S >= musterS`: `Phase = Fighting`, `Slot = task.Haulers++` (late arrivals join). Else if
`S >= musterS`: `S = musterS`, `Phase = AtTarget`.

`Recount` counts `Fighting` as assigned. `TrailSystem`'s users count tasks in `Fighting` too.
`Abort` sends `Fighting` ants home as well.

### 5.4 The fight (ThreatSystem, §9 step 5)

For a spider in `Fighting`, with `n` = active ants in `Fighting` on its Defend task:

```
Wounds += n · dt
if Wounds >= SpiderToughness:                              // driven off
    spider State = Gone; raise SpiderDrivenOff(A = slot, B = task, P = Pos); Stats.SpidersDrivenOff++
    task Phase = Done; each fighter: Phase = GoingHome, Task = default, S = musterS
else:
    FightKillAccum += dt / SpiderFightKillSeconds
    while FightKillAccum >= 1 and n > 0: KillAnt(the fighter with the highest Slot, Spider) — no trail weakening;
                                         n−−; FightKillAccum −= 1
    if n == 0:                                             // defence failed
        task Phase = Aborted (reason DefendersLost; outbound defenders turn home through Abort)
        raise DefendFailed(A = task, B = slot, P = Pos)
        spider State = Patrolling, Wounds = 0, FightKillAccum = 0, Defend = default
```

A spider fought while `Moving` (marked before it reached its new loop) is `Patrolling` after a
failed defence, so its next patrol step puts it on its loop at once: a jump of up to the distance it
had left to walk. The view smooths it. If a task stops resolving mid-fight (a defensive case), the
spider ends the fight the same way.

The player counts toward starting the fight (as toward starting a haul) but adds no wounds. A
player standing within the spider's reach during the fight is in danger like anywhere else.

### 5.5 Fight arithmetic

Exact tick counts at the defaults (float32, computed by stepping the rule above):

| Fighters at the start | Outcome | Ticks | Defenders lost |
|---|---|---|---|
| 8 | driven off | 63 | 0 |
| **6** | driven off | 84 | **0** |
| 5 (5 + player) | driven off | 103 | 1 |
| 4 | driven off | 137 | 1 |
| 3 | driven off | 231 | 2 |
| 2 | all lost | ~180 | 2 |

A full party drives the spider off in 8.4 s and loses nobody: 50 defender-seconds arrive before
the first 9 s kill. Starting short, with the player standing in as the sixth, costs one. A spider
left alone takes about 4 workers a day for its two days. So the price of a defence is six
workers' time and the player's walk, not lives, and it pays whenever a spider has settled on the
colony's traffic.

**Assumed** — a full party wins unhurt. Cheap to change (`SpiderFightKillSeconds`); below 8.4 s a
full party loses one again.

---

## 6. Birds

A bird is an event, not an entity: one `BirdState` at most.

```csharp
public struct BirdState
{
    public bool Active;
    public AntId Target;          // the worker the shadow follows
    public TrailId Trail;         // the target's trail, kept when the target goes
    public Vector2 Pos, PrevPos;  // the shadow
    public long StrikeTick;
    public bool Alarmed;
}
```

### 6.1 Arrival (scheduler, §9 step 3)

By day, not in rain, not on day 0, and only while no bird is active:
`progress[Bird] += BirdsPerDay[season] · dt / DaySeconds` (§4.2: the 180 s of daylight). When
it reaches the threshold: `progress −= threshold`; draw a new threshold; then pick a target among
active ants in `Outbound`, `AtTarget`, `Hauling` or `GoingHome` farther than `BirdNestClearance`
from the nest: `k = hazardRng.Range(0, count)`, the `k`-th in index order. If there are none, the
bird passes unseen (no draw for `k`, no event). Otherwise `Active`, `Target`, `Trail`,
`Pos = PrevPos = AntPosition(target)`, `StrikeTick = Tick + RoundToInt(BirdWarningSeconds / dt)`
(50 ticks), `Alarmed = false`. Raise `ThreatSpawned(A = Bird, B = 0, P = Pos)`.

Birds are not scaled by tier: a bird does not care how big the nest is.

### 6.2 Warning and strike (§9 step 6)

Each tick while active: `PrevPos = Pos`; if `Target` still resolves (active, same gen), `Pos` =
its position and `Trail` = its trail. Otherwise the shadow stays where it is.

On `Tick == StrikeTick`:
- Unless `Alarmed`: victims are ants in the four trail phases within `BirdStrikeRadius` of `Pos`,
  nearest first (ties lowest index), at most `BirdMaxKills`. `KillAnt` each, cause `Bird`, without
  the per-kill trail factor (the strike weakens the trail once, below).
- If `Trail` resolves: `Strength *= KillTrailFactor` (whether or not anyone was taken: the trail
  is scattered).
- If the player is alive and within `BirdStrikeRadius` of `Pos`: the player dies (§8), cause `Bird`.
- Raise `AntsLost(A = n, B = Bird, P = Pos)` if `n > 0`, then `BirdStrike(A = n, B = Alarmed ? 1 : 0, P = Pos)`.
  `Active = false`.

### 6.3 The alarm

New command `Alarm` (no arguments; the Game layer sends it on **Interact** while a shadow is up).
Rejected `PlayerDown` if down; `NoAlarmTarget` unless a bird is active and
`|PlayerPos − Bird.Pos| <= AlarmReach`. Effect: `Alarmed = true`, raise
`AlarmRaised(P = Bird.Pos)`. The strike then takes nobody, but still scatters the trail and still
kills a player standing under it.

Expected losses: a strike on a lone worker takes 1–2; on a carrying party, 3. About 1.6 on
average, and about 70% of birds find a target. With the defaults that is roughly 0.3, 0.55 and
0.45 workers a day in spring, summer and autumn.

**Assumed** — birds target a worker, warn for 5 s and take up to three; the alarm is raised with
Interact, which is otherwise unused. Using Tap would decide the open question of what a tap away
from the nest does. Cheap to change.

---

## 7. Raids

### 7.1 The rival colony

```csharp
public struct RivalState { public float Strength; }   // raiders it could field; never seen
```

`Strength` starts at `RivalStartStrength` and grows every tick, in every season:
`Strength += RivalGrowthPerDay · Strength · (1 − Strength / RivalCapacity) · dt / DayLengthSeconds`.
Undisturbed: 30 → 65.8 by day 10 → 99.4 by day 40. A raid that ends takes `RaidersLost` off it
(floor 1).

**Assumed** — the rival is one number with logistic growth. Losses to you weaken it; food it loots
does not strengthen it. Cheap to change. If loot fed its growth, a colony that loses raids would
face ever bigger ones: a runaway, avoided on purpose.

### 7.2 Launch (scheduler, §9 step 3)

Not on day 0. `progress[Raid] += RaidsPerDay[season] · dt / DayLengthSeconds`. At the threshold:
if raining or a raid is active, hold (`progress = threshold`). Otherwise use it up, draw a new
threshold, then:

```
R0 = RoundToInt(Rival.Strength · RaidCommitByTier[Tier − 1])
if R0 < RaidMinRaiders: nothing (no draw)
a = hazardRng.Range(0, 2π)
From = where the ray from NestPos along (cos a, sin a) leaves SpawnArea
Raid = { Active, Phase = Marching, From, S = PrevS = 0, Length = |NestPos − From|,
         StartRaiders = Raiders = R0 }
raise ThreatSpawned(A = Raid, B = R0, P = From)
```

Raid size follows your tier, so a growing colony keeps meeting raids it can feel.

### 7.3 March

`Marching`: `PrevS = S`; `S += RaiderSpeed · night · dt`. The head is at
`From + (NestPos − From)/Length · S`. A patch-edge raid is 90–140 u from the nest: 45–70 s by day.
Raiders crossing trails take nobody. Their only danger on the way is to the player (§8).

When `S >= Length − NestRadius`: **contact**. `Phase = Fighting`, `Defenders0 = InNest`
(`Idle + Nursing + Digging`). Raise `RaidContact(A = Raiders, B = Defenders0, P = NestPos)`. If
`Defenders0 == 0` it is a breach at once (`Looting`). Fighting starts the next tick.

### 7.4 The fight at the nest

Every worker in the nest fights: idle, nursing and digging. Ants outside do not, until they come
home (the recall of §2 brings them). `D` is the live count each tick, so workers who arrive
mid-fight join it. `R = Raiders`. `rally` = `World.PlayerAtNest`: the player is alive and this
tick's `input.PlayerPos` is within `NestRadius` of the nest. `World` keeps that position in
`_lastPlayerPos`, set at the start of every `Tick` from the input. Like `DayStartedThisTick`, it is
input, not state: neither hashed nor saved, and always rewritten before it is read. Per tick, in this order:

```
take = Min(Food, RaidLootPerRaiderSecond · R · dt); Food −= take; Raid.Loot += take; Stats.FoodLooted += take
RaiderKillAccum   += RaidKillPerSecond · (rally ? RallyFactor : 1) · D · dt
DefenderKillAccum += RaidKillPerSecond · R · dt
while RaiderKillAccum >= 1 and R > 0:   R−−, RaidersLost++, accum −= 1
while DefenderKillAccum >= 1 and D > 0: kill one in the nest (idle, else nursing, else a digger:
                                        the starvation order, SIM_M2.md §3.2), DefendersLost++,
                                        Stats.WorkersKilled++, accum −= 1
if RaidersLost >= ceil(StartRaiders · RaidBreakFraction): repulsed
else if DefendersLost >= ceil(Defenders0 · RaidBreakFraction) or InNest == 0: breach
```

Raise `AntsLost(A = defenders lost this tick, B = Raid, P = NestPos)` on ticks with losses. If the
loot empties the stores, raise `FoodRanOut`.

- **Repulsed**: raise `RaidRepulsed(A = DefendersLost, B = round(Loot))`;
  `Stats.RaidsRepulsed++`; `Rival.Strength −= RaidersLost`; `Phase = Over`.
- **Breach**: `Phase = Looting`, `Timer = RaidBreachLootSeconds`, `Stats.RaidsBreached++`, and
  raise `RaidBreached(A = DefendersLost, B = round(Loot))` with the totals so far, so the HUD can
  say so the moment they break in. Each tick: loot as above, no fighting, `Timer −= dt`. At 0:
  raise `RaidLootingEnded(A = DefendersLost, B = round(Loot))` with the final totals;
  `Rival.Strength −= RaidersLost`; `Phase = Over`. An empty nest at contact breaks in the same way.

`Over` is terminal; the record is cleared at the start of the next ThreatSystem step. Digging goes
on during a raid: the diggers fight in it but are not taken off the build.

The queen is never a target, and the brood is not touched. A breach costs workers and food, never
the colony's heart.

### 7.5 Raid arithmetic

This is Lanchester's square law with both sides breaking at half. Loss to the defenders falls as
the nest grows, roughly `0.375 · R0² / D0` when `D0` is much larger than `R0`, which is where spare
workers earn their upkeep. Exact outcomes at the defaults (stepping the rule above, float32, no
workers coming home mid-fight):

| In the nest | Raiders | Player home | Result | Workers lost | Raiders lost | Food looted | Fight |
|---|---|---|---|---|---|---|---|
| 33 | 19 | no | repulsed | 4 | 10 | 35 | 16.2 s |
| 33 | 19 | yes | repulsed | 3 | 10 | 23 | 10.5 s |
| 60 | 15 | no | repulsed | 1 | 8 | 12 | 6.8 s |
| 60 | 15 | yes | repulsed | 1 | 8 | 8 | 4.5 s |
| 15 | 19 | no | **breach** | 8 | 5 | 123 | 24 s + 30 s looting |

Being home saves a worker or two and a third of the loot.

**Assumed** — the fight is the square law above: every worker in the nest fights, both sides break
at half, raiders loot while they fight and for 30 s after a breach, and the player at the nest makes
the defenders fight half again as hard. No threats of any kind on day 0. Cheap to change: data,
except the square law itself, which is what gives a big nest its advantage (medium: §16 would need
redoing). A nest that is too empty when raiders
arrive loses half its workers and, through the fight and the 30 s of looting, about six food per
raider.

---

## 8. The player ant: danger, death, respawn

GAME.md had this as undecided: when the player ant dies, respawn or game over, and whether the
queen's death ends the colony. It blocks M3's tuning, so M3 runs on the smallest rule it needs.

**Assumed** — the player ant dies to threats and comes back. After `PlayerRespawnSeconds` (10 s)
you leave the nest as a fresh worker, and the colony is one worker smaller. Nothing else is lost.
The queen never dies in M3: no threat targets her or the brood. The year, the save and the colony go
on. Cost to change, by what you might decide instead:

- **Game over when the player ant dies** (permadeath): a third outcome beside survived/died
  (`YearOutcome` gains a cause), the save is ended or deleted at death, and every threat's player
  danger must be retuned to be rare and well telegraphed. In particular the bird's instant kill
  (§6.2) would have to go. About a day of sim work and a full balance pass; the colony rules are
  untouched.
- **A different respawn cost** (longer delay, food, nothing): one or two config values. Cheap.
- **The queen can die** (raid breach reaching her, starvation): `ColonyState` gains the queen,
  laying stops at her death, and the colony's death rule changes (SIM_M2.md §6.3: today it is
  "no workers and no brood"). If her death also ends the game, the same third outcome as above.
  Raids would need a "reached the queen" result and their arithmetic redone. Medium: a few days,
  plus the raid table in §7.5 and the year model in §16.

### 8.1 State

```csharp
public struct PlayerState { public bool Down; public float Danger; public float RespawnTimer; }
```

`PlayerAlive => !Down`. The player ant is still not sim-stepped: its position is an input. The
sim only decides whether it is alive.

### 8.2 Danger (§9 step 7)

While alive, the player is **in reach** if any non-`Gone` spider not hiding from rain is within
`SpiderReach` of `PlayerPos`, or a `Marching` raid's head is within `RaidReach`. Then
`Danger += ThreatDangerPerSecond · dt`, otherwise `Danger = Max(0, Danger − DangerRecoverPerSecond · dt)`.
At `Danger >= 1` the player dies, cause the first source found (spiders in index order, then the
raid). A bird strike kills outright (§6.2).

From zero, the player dies on the 29th tick (2.9 s) in reach, and recovers fully in 5 s out of it.

### 8.3 Death and respawn

**Death**: `Down = true`, `Danger = 0`, `RespawnTimer = PlayerRespawnSeconds`,
`Stats.PlayerDeaths++`. An active draft is abandoned with the new reason `PlayerDied`. Raise
`PlayerDied(A = (int)cause, P = PlayerPos)`.

While down: every command is rejected `PlayerDown`; the player never counts toward a party,
muster or rally; danger does not accrue. The Game layer keeps sending `PlayerPos`. The sim ignores
it.

**Respawn** (each tick while down): `RespawnTimer = Max(0, RespawnTimer − dt)`. At 0: if `InNest > 0`
and the colony is not `Dead`, take one worker from the nest (idle, else nursing, else a digger),
`Down = false`, raise `PlayerRespawned(P = NestPos)`. Otherwise wait: the player returns the tick a
worker is in the nest. A dead colony never returns the player; the outcome screen takes over.

The worker taken is not counted in `WorkersKilled`: it became you. `PlayerDeaths` records it.

---

## 9. `ThreatSystem.Step` (tick step 10)

If `!ThreatsEnabled`: return. Otherwise, in this order:

1. **Free** spider slots in `Gone` and a raid in `Over` (`Active = false`).
2. **Positions**: fill `Vector2[] AntPosScratch` (length `MaxAnts`, allocated once) with
   `AntPosition(i)` for every active ant. Kills later in this step deactivate ants; everything
   below skips inactive ones. Nobody moves in step 10.
3. **Schedulers**, in order spider, bird, raid (§4.2, §6.1, §7.2), then rival growth (§7.1).
   These are the only places, with §4.5, that draw from the hazard stream in step 10.
4. **Raid** (§7.3–7.4).
5. **Spiders**, index order: fight (§5.4), else, if not raining, patrol or move (§4.3, §4.5), hunt
   (§4.4), relocation check (§4.5); then age and leaving (§4.6).
6. **Bird** (§6.2).
7. **Player** (§8): respawn if down, else danger and death.

How many draws a tick makes depends only on state: whether a sample, an arrival or a relocation
is due, and how many placement tries are rejected. Rejection depends on float positions, so, like
the rest of the sim, the hazard stream is deterministic on one build target (as GAME.md assumes),
not across platforms.

---

## 10. Commands

`CommandKind` appends `MarkThreat`, `Alarm`. `RejectReason` appends `Recalled`, `PlayerDown`,
`NoSuchThreat`, `DefendActive`, `NoAlarmTarget`. `TaskAbortReason` appends `PartyLost`,
`ThreatGone`, `DefendersLost`. `TrailAbandonReason` appends `PlayerDied`, `ThreatGone`.

| Command | Change |
|---|---|
| every command | rejected `PlayerDown` first while the player is down |
| `MarkTrailStart` | accepts the player's recruiting `Claimed` find as a re-walk (§1.4) |
| `MarkThreat` | new (§5.1) |
| `Alarm` | new (§6.3) |
| `TapAnt` | `Recalled` while `RecallActive` (checked after `NotAtNest`); `−1` prefers a recruiting defence (§5.2) |
| `StartBuild`, `CancelBuild` | unchanged (digging is not recalled) |

`TrailDraft` gains `bool Refresh` and `SpiderId Spider`. `ItemSystem`'s expiry exemption for the
draft's item is unchanged (a spider draft has no item).

---

## 11. Events

Existing kinds now raised:

| Kind | A | B | P | Step |
|---|---|---|---|---|
| `WeatherChanged` | new `Weather` | old | | 2 |
| `ThreatSpawned` | `ThreatKind` | spider slot / 0 for a bird / raiders for a raid | den / shadow / entry point | 10 |
| `AntsLost` | count | `ThreatKind` cause | where | 10 |

`TrailMarkStarted.A` is `−1 − spider slot` for a spider. `TaskCreated.B` is `−1 − spider slot` for a
defence.

New kinds, **appended after `YearEnded`**:

| Kind | A | B | P | Step |
|---|---|---|---|---|
| `WeatherForecast` | coming `Weather` | current | | 2 |
| `TrailRefreshed` | trail index | task index | item pos | 3 |
| `SpiderMoved` | slot | | new den | 10 |
| `SpiderLeft` | slot | | pos | 10 |
| `DefendStarted` | task index | spider slot | spider pos | 5 |
| `SpiderDrivenOff` | spider slot | task index | pos | 10 |
| `DefendFailed` | task index | spider slot | pos | 10 |
| `AlarmRaised` | | | shadow pos | 3 |
| `BirdStrike` | workers taken | 1 if alarmed | shadow pos | 10 |
| `RaidContact` | raiders | defenders | nest | 10 |
| `RaidRepulsed` | workers lost | food looted (rounded) | nest | 10 |
| `RaidBreached` | workers lost so far | food looted so far (rounded) | nest | 10 |
| `RaidLootingEnded` | workers lost | food looted (rounded) | nest | 10 |
| `TrailWashed` | trail index | | | 4 |
| `PlayerDied` | `ThreatKind` cause | | player pos | 10 |
| `PlayerRespawned` | | | nest | 10 |

A raid fight raises at most one `AntsLost` a tick. The 256-event cap is not at risk.

---

## 12. `StateHash`

Earlier sections keep their order, with these in-section additions:

- Section 2 (**Draft**), if active, after the points: `Refresh`, `Spider` (index, gen).
- Section 4 (**Trails**), per active trail, after `PlayerLaid`: `Washed`.
- Section 5 (**Tasks**), per active task, after `ProgressTicks`: `Spider` (index, gen).
- `ColonyStats` gains, folded after `PeakDay` (so both section 10 and the outcome's snapshot carry
  them): `WorkersKilled, PlayerDeaths, SpidersDrivenOff, RaidsRepulsed, RaidsBreached`, then
  `FoodLooted` (double).

New sections after 11:

12. **Hazard stream and weather**: `_hazardRng.State` (low, high), `Current, Next, Dwell`.
13. **Player**: `Down, Danger, RespawnTimer`.
14. **Spiders**, every slot: `Gen`; if active: `State, Den.x, Den.y, Angle, Dir, Pos.x, Pos.y,
    PrevPos.x, PrevPos.y, MoveTo.x, MoveTo.y, Age, KillAccum, Quiet, Wounds, FightKillAccum, Defend`.
15. **Bird**: `Active`; if active: `Target, Trail, Pos.x, Pos.y, PrevPos.x, PrevPos.y, StrikeTick`
    (low, high), `Alarmed`.
16. **Schedulers and rival**: `ThreatProgress[0..3], ThreatThreshold[0..3], Rival.Strength`.
17. **Raid**: `Active`; if active: `Phase, From.x, From.y, S, PrevS, Length, StartRaiders, Raiders,
    Defenders0, DefendersLost, RaidersLost, RaiderKillAccum, DefenderKillAccum, Timer, Loot` (double).

`AntPosScratch` is derived and is neither hashed nor saved.

---

## 13. Saves: version 2

The sim save is at version **1** today (`SaveData.CurrentVersion = 1`, fixture `save_v1.json`). M3
makes it **2**.

### 13.1 Shape

```csharp
public const int CurrentVersion = 2;      // MinLoadableVersion stays 1
public HazardSave Hazards;                 // RngState == 0 means absent: a save from before M3

[Serializable] public struct HazardSave
{
    public long RngState;
    public int Weather, WeatherNext, WeatherDwell;
    public bool PlayerDown; public int PlayerDanger, RespawnTimer;           // float bits
    public int[] ThreatProgress, ThreatThreshold;                            // float bits, length 4
    public int RivalStrength;                                                // float bits
    public SpiderSave[] Spiders;                                             // every slot
    public BirdSave Bird;
    public RaidSave Raid;
}
[Serializable] public struct SpiderSave { public int Gen; public bool Active; public int State, Dir,
    DenX, DenY, Angle, PosX, PosY, PrevPosX, PrevPosY, MoveX, MoveY, Age, KillAccum, Quiet, Wounds,
    FightKillAccum, DefendIndex, DefendGen; }
[Serializable] public struct BirdSave { public bool Active, Alarmed; public int TargetIndex, TargetGen,
    TrailIndex, TrailGen, PosX, PosY, PrevPosX, PrevPosY; public long StrikeTick; }
[Serializable] public struct RaidSave { public bool Active; public int Phase, FromX, FromY, S, PrevS,
    Length, StartRaiders, Raiders, Defenders0, DefendersLost, RaidersLost, RaiderKillAccum,
    DefenderKillAccum, Timer; public long Loot; }
```

Additions to existing DTOs (all default to the right value for a v1 save): `DraftSave` gains
`Refresh, SpiderIndex, SpiderGen`; `TrailSave` gains `Washed`; `TaskSave` gains `SpiderIndex, SpiderGen`; `StatsSave` gains
`WorkersKilled, PlayerDeaths, SpidersDrivenOff, RaidsRepulsed, RaidsBreached` and `long FoodLooted`.
New enum values are stored as ints, like the old ones.

### 13.2 Migration v1 → v2

`SaveMigrations.Steps` gains its first entry, `V1ToV2`, and it is **empty**. The change is additive
(SIM_M2.md §10.4), but the right starting values for the hazard block depend on the config, which a
migration step does not see. So `World.TryLoad` does it: when `Hazards.RngState == 0` it calls
`InitHazards()`, the same code a new colony runs: hazard stream `Rng.ForHazards(Seed)`, weather
Clear/Clear with `Dwell = 1`, the player alive with no danger, no spiders, bird or raid,
`Rival.Strength = RivalStartStrength`, and fresh thresholds drawn from the new hazard stream.

A real v2 save never has `RngState == 0` (an `Rng` state is never zero), so the test is
unambiguous. A save with `RngState == 0` but any active spider, an active bird or an active raid is
`Corrupt`.

Consequence for players: a colony saved in the middle of its year by an M2 build continues with
clear skies and a rival at its starting strength, whatever the day. Raids in that year are smaller
than in a fresh one. That is accepted: the alternative is a migration that guesses the day's rival
strength, and the guess would be wrong under any later retune.

**Assumed** — M2 saves continue with fresh weather and threats. Cheap to change.

### 13.3 Validation additions (`Corrupt` on failure)

`Hazards.Spiders` no longer than `MaxSpiders` (shorter is padded); enum ints in range; `Dwell >= 1`;
`ThreatProgress`/`ThreatThreshold` length 4; active spiders `Gen >= 1`; a spider's `Defend`, a
task's `Spider`, the draft's `Spider` and the bird's `Target`/`Trail` may be stale but never name a
gen above their slot's; an active Defend task's spider must resolve; a `Fighting` spider must have a
resolving Defend task in `Fighting`; `RespawnTimer >= 0`; `Danger` in `[0, 1]`.

---

## 14. Game / UI contract

### 14.1 Read API (no allocation)

```csharp
WeatherState WeatherNow; float SecondsToWeatherSample; bool Raining; bool RecallActive;
ReadOnlySpan<Spider> Spiders;                     // MaxSpiders slots; check Active
Vector2 SpiderPosition(int slot, float alpha);    // Lerp(PrevPos, Pos, alpha)
float SpiderReach, SpiderPatrolRadius;            // config, for the danger ring and the loop
bool SpiderHiding(int slot);                      // raining and not fighting
float SpiderSecondsLeft(int slot);                // until it leaves
Vector2 DefendMuster(int taskIndex);              // where the defenders gather
BirdState Bird; Vector2 BirdShadow(float alpha); float BirdSecondsToStrike; float BirdStrikeRadius;
RaidState Raid; Vector2 RaidHead(float alpha); float RaidSecondsToContact; int InNest; bool PlayerAtNest;
RivalState Rival;
PlayerState Player; bool PlayerAlive; float PlayerDanger; float RespawnSecondsLeft; bool WaitingForWorker;
bool CanRefresh(ItemId item);                     // §1.4 would accept Mark on it
```

### 14.2 What the view shows

- **Weather**: rain and overcast as lighting, particles and wet ground; trails visibly thinning in
  rain (their alpha already follows strength). A HUD weather icon, with "rain soon" whenever
  `Next == Rain` and it is not raining.
- **Sun**: rotates with `Clock.TimeOfDay01` (already derivable; M3 presentation work).
- **Spider**: a body at `SpiderPosition`, faint ground marks along its loop, and a danger ring of
  `SpiderReach` when the player is within about 15 u. In rain it is drawn hidden or tucked in.
- **Defend**: the defend trail like a haul trail; defenders gather at `DefendMuster`. Fighters
  are placed by `AntPosition`, which puts `Fighting` ants on a ring of `SimLimits.FightRingRadius`
  (2 u) around the spider at
  angle `2π (Slot + 0.5) / SpiderDefenders`; the view lerps them in from the muster point.
- **Bird**: a moving shadow at `BirdShadow`, radius `BirdStrikeRadius`, darkening over
  `BirdSecondsToStrike`; a swoop at `BirdStrike`.
- **Raid**: a column of raider bodies along `From → RaidHead` (count `Raiders`, capped for
  drawing); at the nest, a scuffle and a raiders/defenders count. Raiders are drawn from a generic
  rival-ant model, nothing species-specific.
- **Danger**: a screen-edge vignette from `PlayerDanger`. **Death**: the controller stops, the
  camera holds, a short fade; on `PlayerRespawned` the Game layer teleports the player to `P` (the
  nest) and returns control.

### 14.3 Inputs

- **Mark** with the interaction probe on a spider (an `Interactable` on the spider view, reach
  `SpiderReach + ThreatMarkMargin`) sends `MarkThreat(index, gen)`. On a find it sends
  `MarkTrailStart` as before, including on a find that `CanRefresh`.
- **Interact** while `Bird.Active` and within `AlarmReach` of the shadow sends `Alarm`.
- **Tap** at the nest as before. It is refused with `Recalled` in rain or during a raid.

### 14.4 Copy needed (for `world-builder`, not written here)

All generic: "spider", "bird", "raiders", "the rival colony". No species and no biology the sim
does not model (UI_COPY voice rules).

- HUD: `hud.weather.clear`, `hud.weather.overcast`, `hud.weather.rain`, `hud.weather.rain_soon`;
  `hud.raid.marching` ({n} raiders, {s} s to the nest); `hud.raid.fighting` ({n} raiders,
  {d} defenders); `hud.respawn` ({s}); `hud.respawn.waiting` (no worker left in the nest to take
  your place).
- Prompts: `prompt.spider.mark`, `prompt.spider.defended` (defenders called),
  `prompt.spider.fighting`, `prompt.bird.alarm`, `prompt.find.refresh` (walk the trail again),
  `prompt.nest.recalled.rain`, `prompt.nest.recalled.raid`.
- Toasts: `toast.weather.overcast`, `toast.weather.rain`, `toast.weather.rain_soon`,
  `toast.weather.dry` (the rain has stopped; walk your trail again), `toast.trail.refreshed`,
  `toast.spider.arrived`, `toast.spider.moved`, `toast.spider.left`, `toast.spider.driven_off`,
  `toast.defend.started`, `toast.defend.failed`, `toast.ants_lost.spider`, `toast.ants_lost.bird`,
  `toast.bird.shadow`, `toast.bird.alarm`, `toast.bird.missed` (alarmed strike),
  `toast.raid.spotted`, `toast.raid.contact`, `toast.raid.repulsed` ({lost}, {food}),
  `toast.raid.breached` (they are in; {lost} so far), `toast.raid.looting_ended` ({lost}, {food}),
  `toast.trail.washed` (rain has washed your trail; walk it again when it stops), `toast.player.died.spider`, `toast.player.died.bird`,
  `toast.player.died.raid`, `toast.player.respawned`, `toast.task.party_lost`,
  `toast.task.threat_gone`, `toast.trail.player_died`, `toast.trail.threat_gone`.
  `DefendersLost` aborts share `toast.defend.failed`.
- Rejections: `reject.recalled.rain`, `reject.recalled.raid`, `reject.no_such_threat`,
  `reject.defend_active`, `reject.no_alarm_target`. `PlayerDown` is silent.
- Names: `threat.spider`, `threat.bird`, `threat.raiders`, `threat.rival`.
- Outcome screen lines: workers lost to threats, food looted, raids repulsed of raids, spiders
  driven off, times you were lost.

---

## 15. Tests (`Assets/Tests/EditMode`)

`SimTestKit` gains `CalmConfig` and `Calm()` (§0.4), `Hot()` (default config with
`SpidersPerNight ×10`, `BirdsPerDay ×10`, `RaidsPerDay = {1, 1, 1, 0}`, so every threat appears in
a short run), and test hooks on `World`: `SetWeatherForTest(Weather current, Weather next)`,
`int AddSpiderForTest(Vector2 den, float angle, int dir)`, `ref Spider SpiderRef(int)`,
`void StartBirdForTest(AntId target)`, `void LaunchRaidForTest(float angle, int raiders)`,
`ref RaidState RaidRef`, `ref RivalState RivalRef`, `ref PlayerState PlayerRef`,
`ulong HazardRngState`. Hooks act between ticks. Unless stated, a test uses `Quiet()`: `Rich()`'s
food values with weather and threats **on**, every transition row `{1, 0, 0}` (always clear unless a
hook sets it) and every arrival rate 0, so only what the test sets up happens.
Standard geometry is SIM.md's: nest at the origin, cube at `(40, 0)`.

### `WeatherTests`
- `RowsSumToOne`: every default row sums to 1 within 1e-5 and has no negative entry. A config with
  one row summing to 0.9 throws `ArgumentException` at construction.
- `ClearNeverGoesStraightToRain`: 50 seeds × 200 000 ticks: no `WeatherChanged(Rain, Clear)`.
- `SamplesOnlyOnTheQuarterDay`: every `WeatherChanged` and `WeatherForecast` is on a tick divisible
  by 900.
- `DwellHolds`: same runs: every clear spell lasts ≥ 1 800 ticks; every rain ≥ 900.
- `NoRainBeforeTick2700`: 200 seeds: `Current != Rain` for ticks < 2 700.
- `ForecastComesTrue`: after every sample, the next sample's `Current` equals this sample's `Next`.
- `StationaryShare`: `WeatherSampleSeconds = 0.1`, `DaysPerSeason = 100 000` (spring forever),
  200 000 ticks: rain share 0.103 ± 0.01, clear 0.608 ± 0.015.
- `SameSeedSameSkies`: two worlds, seed 7, 100 000 ticks: identical `WeatherChanged` sequences.
- `HazardStreamIsolated`: `WeatherSpawnFactor = {1,1,1}`, `M2Defs()`, no input, 36 000 ticks, weather
  and threats on vs. off: identical `ItemSpawned` sequences (tick and `P`), and identical main
  `RngState` at the end.
- `DisabledIsClearAndDrawsNothing`: `WeatherEnabled = false`: Clear for 100 000 ticks; the hazard
  state after 100 000 ticks equals its state after construction (threats also off).

### `RainTests`
- `OutboundTurnHome`: lay the standard trail; when 3 ants are `Outbound`, set rain. That tick all 3
  are `GoingHome` with `Task` none; the task is still `Recruiting`, `RecruitAccum == 0`; no
  `AntLeftNest` while it rains.
- `AtTargetTurnHome`: `AntsRequired = 12`, wait until 6 are `AtTarget`, set rain → all 6 `GoingHome`.
- `HaulersCarryOn`: rain during a haul: the item is still delivered, exactly one `ItemDelivered`.
- `Washout`: an unreferenced trail at 1.0 in rain fades with `TrailFaded` on tick 131 (±1); dry,
  783 (SIM.md).
- `ReferencedHoldsAtFloor`: a recruiting trail in rain for 300 ticks: `Strength == TrailFloor`.
- `TapRecalled`: player at the nest in rain → `CommandRejected(TapAnt, Recalled)`, no spawn.
- `RecruitingResumesAtTheFloor`: rain 300 ticks, then clear: the next recruit comes ~100 s later
  (between ticks 980 and 1 020 after the rain ends).
- `RewalkRenews`: after rain, `MarkTrailStart` on the claimed cube (accepted), walk home →
  `TrailRefreshed`, the same trail and task ids, `Strength == PlayerTrailStrength · decayPerTick`
  (±1e-6), no new trail slot; the first recruit follows within 22 ticks.
- `RewalkOnlyTheRecruitingPlayerHaul`: a `Hauling` item → `ItemNotAvailable`.
- `OvercastCutsSpawns`: SIM_M2's `MeanRate` with overcast held: 36 000 ticks → 17–23 seeds.

### `SpiderTests`
- `PatrolIsDeterministic`: `AddSpiderForTest((50, 0), 0, +1)`; after 100 ticks
  `Angle == 100 · 0.1 / 6` (±1e-4) and `Pos == den + 6 · (cos, sin)` (±1e-4).
- `LoopTakes377Ticks`: after 377 ticks `Pos` is within 0.05 u of the start.
- `TakesOnePer10SecondsOfExposure`: `SpiderSpeed 0`, an `AtTarget` worker held within reach (cube at
  the den, `AntsRequired 12`): first `AntsLost(1, Spider)` on tick T + 100 (±1), the next at +200.
- `ExposureAccumulates`: two exposures of 60 ticks separated by 300 quiet ticks → the kill comes
  40 ticks into the second.
- `KillWeakensTrail`: on the kill tick the victim's trail strength is 0.7 × the same trail's
  strength in a control world without the spider (the two agree until step 10 of that tick) (±1e-5).
- `HidesInRain`: rain for 600 ticks with a worker in reach: no kill, `Pos` unchanged.
- `ArrivesAtNightOnly`: `Hot()`: every `ThreatSpawned(Spider)` is on a tick with `IsNight`; none on day 0.
- `DenOnTheBusiestTrail`: two trails with 4 and 1 ants: the den lies on the first (within 1e-3 of
  `Sample(s)` for some `s` in `[0.3 L, 0.8 L]`) and ≥ 25 u from the nest.
- `AtMostSpiderMax`: `Hot()`, 5 days: never more than 2 active.
- `LeavesAfterTwoDays`: `SpiderLeft` on placement tick + 7 200 (±1); slot inactive the next tick.
- `MovesWhenQuiet`: no ants near its loop for 300 ticks and a busy trail elsewhere →
  `SpiderMoved`, `State == Moving`, then `Patrolling` on arrival.
- `NeverMovesWhileDefended`: same, with a Defend task aimed at it → no `SpiderMoved`.
- `MarkThreatMakesADefence`: mark from 8 u, walk home → `TrailLaid`, a `Defend` task `Recruiting`,
  trail `PlayerLaid == false`; an existing player haul is still `Recruiting` (not replaced).
- `MarkThreatRejections`: from 11 u → `OutOfReach`; a second mark → `DefendActive`; stale gen →
  `NoSuchThreat`; during a draft → `DraftActive`.
- `SixDriveItOff`: a fight starting with 6 → `SpiderDrivenOff` 84 ticks (±1) after
  `DefendStarted`, no `AntsLost`; all 6 `GoingHome`; task `Done`; `Stats.SpidersDrivenOff == 1`.
- `TwoAreLost`: a fight starting with 2 (`SpiderDefenders = 2`): `DefendFailed` 180 ticks (±1)
  after the start; 2 lost; task `Aborted(DefendersLost)`; spider `Patrolling`, `Wounds == 0`.
- `RemarkRewalksADefence`: a defence trail washed by rain (`TrailWashed` raised); `MarkThreat` on
  its spider is accepted and completes with `TrailRefreshed`, the same task, strength back to
  `PlayerTrailStrength · decayPerTick`, `Washed == false`.
- `PlayerCountsAtMuster`: player within 3 u of the muster point → the fight starts with 5 ants.
- `SpiderGoneAbortsDefence`: the spider leaves while defenders walk out → `TaskAborted(ThreatGone)`,
  every defender returns.

### `BirdTests`
- `DaylightOnly`: `Hot()`, 3 days with ants always out: every `ThreatSpawned(Bird)` is by day,
  none in rain, none on day 0.
- `NoTargetNoBird`: no ants outside → no bird events; `ThreatProgress[Bird]` still resets.
- `FiveSecondWarning`: `BirdStrike` exactly 50 ticks after `ThreatSpawned(Bird)`.
- `TakesNearestThree`: strike on a 6-ant hauling party → `AntsLost(3, Bird)`; 3 still `Hauling`;
  the item still delivered.
- `WipedPartyDropsTheFind`: a seed party of 2 struck → both lost, `TaskAborted(PartyLost)`, the seed
  `Lying` at the strike point with `Expires == false`, and markable again.
- `AlarmSavesThem`: `Alarm` from 8 u during the warning → `AlarmRaised`, `BirdStrike(0, 1)`, the
  trail still × 0.7.
- `AlarmOutOfReach`: from 12 u → `CommandRejected(Alarm, NoAlarmTarget)`.
- `PlayerUnderTheStrikeDies`: player within 3 u at the strike → `PlayerDied(Bird)`.

### `RaidTests`
- `MarchesFromTheEdge`: `LaunchRaidForTest(0, 15)` with the default area `(−100, −100, 200, 200)`:
  `ThreatSpawned(Raid, 15, P = (100, 0))`; `RaidContact` 480 ticks (±1) later by day
  (`(100 − 4) / 2` s).
- `RecallsOnLaunch`: outbound ants turn home on the raid's first tick; `TapAnt` → `Recalled`; no
  recruiting until `RaidRepulsed`/`RaidLootingEnded`.
- `Repulse_33v19`: in-nest 33 (via hooks, no ants out), raiders 19, player away →
  `RaidRepulsed(A = 4, B = 35)` 162 ticks (±2) after contact; `Rival.Strength` down by 10.
- `RallyHelps_33v19`: player at the nest → `RaidRepulsed(A = 3, B = 23)`, 105 ticks (±2).
- `Breach_15v19`: → `RaidBreached(A = 8)` at the break-in, then `RaidLootingEnded(A = 8, B = 123 ± 2)`
  300 ticks later; brood and `Colony.Dead` untouched.
- `EmptyNestBreachesAtOnce`: in-nest 0 → `Looting` on the contact tick.
- `NoRaidInRainOrWinter`: rain held → no `ThreatSpawned(Raid)`, `ThreatProgress[Raid]` held at
  threshold; on winter days → none.
- `RivalGrowsLogistically`: rates 0, 36 000 ticks → `Rival.Strength` 65.76 ± 0.05.
- `ColumnEndangersPlayer`: player on the column's path → `PlayerDied(Raid)` after 29 ticks in reach.

### `PlayerDeathTests`
- `DiesAfter29TicksInReach`: stationary spider (`SpiderSpeed 0`), player inside reach →
  `PlayerDied(Spider)` on tick 29 (±1).
- `DangerRecovers`: 20 ticks in, then out → `Danger` falls 0.02 a tick to 0.
- `DeathAbandonsTheDraft`: dying while laying a trail → `TrailAbandoned(PlayerDied)`.
- `WhileDownEverythingIsRefused`: every command → `PlayerDown`; the player at a find does not count
  toward its party.
- `RespawnTakesOneWorker`: `PlayerRespawned(P = NestPos)` 100 ticks (±1) after death; `Idle` one lower
  at that tick; `Population` one lower; `Stats.PlayerDeaths == 1`; `WorkersKilled` unchanged.
- `RespawnWaitsForAWorker`: everyone outside → no respawn until the first `AntReturned`, then
  respawn that tick with that worker taken.
- `DeadColonyNeverRespawns`.
- `QueenUntouched`: after a breach with an empty nest, `Brood` unchanged and the queen still lays
  the next egg on schedule.

### `DeterminismTests` (add)
- `M3_SameSeedIdentical`: `Hot()`, `M2Defs()`, seed 2035 (**Assumed** — the seed is chosen so the run contains every required event kind, including a worker lost to a spider, which seed 2024 no longer does now that a full party wins unhurt; re-pick the seed when a retune empties the event set), 72 000 ticks, `M3Script`: `M2Script` plus,
  when a spider is within 30 u of the player's path, walk to 8 u of it and `MarkThreat`, walk home,
  tap once; `Alarm` whenever a shadow is within 10 u; walk to the nest on `ThreatSpawned(Raid)`; and
  once, on day 2, walk into a spider. Two worlds agree on `StateHash` every tick. The run must
  contain at least one of each: rain, `AntsLost` by each cause, `SpiderDrivenOff`, `RaidRepulsed`
  or `RaidLootingEnded`, `PlayerDied`, `PlayerRespawned`, `TrailRefreshed`.
- `M3_DifferentSeedDiffers`: seed 2025 → different weather sequence.
- `HashCoversM3State`: changing, each alone via hooks: weather `Next`, a spider's `Angle`,
  `KillAccum`, the raid's `S`, the bird's `StrikeTick`, `PlayerDanger`, `Rival.Strength`,
  `ThreatProgress[Raid]` → `StateHash` changes.

### `SaveRoundTripTests` (add)
- `RoundTrip_M3` at four points of the `M3Script` run (seed 11): mid-fight with a spider, a raid
  marching, a bird shadow up, the player down → `Ok`, equal `StateHash`, and 1 000 more ticks with
  equal hashes and identical event sequences.
- `RoundTrip_JsonStable` extends to these points.

### `SaveMigrationTests` (update)
- `V1FixtureLoads` becomes `V1FixtureMigrates`: `save_v1.json` → `Migrated` (not `Ok`); every M2
  field equal to the fixture's decoded values (colony, items, trails, tasks, ants, chambers,
  spawner, stats, outcome); hazards fresh: Clear/Clear/1, no spider, bird or raid, player alive,
  `Rival.Strength == RivalStartStrength`, hazard stream state equal to
  `Rng.ForHazards(seed)` after its three threshold draws; 100 ticks without exception. Its
  recorded `StateHash` constant is re-recorded once, because the hash now covers M3 state.
- `V2FixtureLoads`: golden `save_v2.json` (an `[Explicit]` maker, taken mid-raid with a spider
  present) → `Ok` and its recorded hash.
- `ZeroHazardRngWithSpidersIsCorrupt`.
- `StepsCount`: `SaveMigrations.Steps.Count == CurrentVersion − MinLoadableVersion` (1).

### `YearModelTests`
- `RaidTableHolds`: the five rows of §7.5 reproduced by a world set up with hooks (each a test case).
- `FightTableHolds`: the six rows of §5.5.
- `BotYear` and `BotYear_DefendingPays` (`[Explicit]`, `[Category("Balance")]`, about 143 000 ticks
  per bot per seed): default config, `M2Defs()`, seeds 1–5, two bots that differ **only** in how they
  meet threats.

  Both bots run the same colony policy, `ColonyKeeper`, the build rule of SIM_M2.md §14.1, checked
  whenever the bot is at the nest and no dig is under way, first match wins, in the lowest unlocked
  free slot: a brood chamber while `QueenBlocked` and fewer than 3 are built; a processing chamber
  once `Population >= 16` and none is built; a store chamber once processing and 2 brood chambers
  exist, while `FoodCapacity < 1.1 · WinterFoodNeed`. Both mark finds as `M2Script` does (the
  lowest-index lying find the tier allows, every 1 200 ticks). A bot that never builds processing
  stays at tier 1, where spiders come at half rate and raids are small, and measures nothing.

  - `DefendingBot` = `M3Script` with `ColonyKeeper` and its one deliberate death switched off.
  - `IgnoringBot` = the same routine that never marks a spider, never raises the alarm, never goes
    home for a raid, never re-walks a trail, and marks into rain.

  **Preconditions** (`Assert.Inconclusive` if they fail, because the bots are then not the players
  §16 describes): in at least 3 of 5 seeds each bot reaches tier 2 by day 15; over the five seeds
  each bot meets at least 6 spiders and 6 raids.

  **`BotYear`**: for every seed the defending bot's colony is alive at `YearEnded`, and its
  `WorkersKilled` is between 5% and 30% of its `WorkersRaised`.

  **`BotYear_DefendingPays`**, measured on what defending controls; birds are a tax both bots pay,
  and the alarm is rarely within reach, so bird losses are left out:
  1. *Per spider*: (workers lost to spiders, fights included) ÷ (spiders that arrived), pooled over
     the five seeds. The defending bot's is at most half the ignoring bot's.
  2. *Controllable cost*: lost to spiders + lost to raids + `FoodLooted / 10`, median over the
     seeds. The ignoring bot's is at least 1.5× the defending bot's.

  Log both bots' workers as winter breaks, but assert nothing on them: §16.3 shows the headline is
  food-bound and moves least of all.

---

## 16. Balance: the year with threats

### 16.1 The model

The expected-value year model of SIM_M2.md §14, re-implemented at 10 s resolution, reproduces the
calm year (reasonable player: 88 workers as winter breaks, against SIM_M2's 86). Threats are added
as expected rates derived from the rules above:

- **Rain** costs the share of the season it rains (§1.1) times how well the player handles it: a
  player who waits it out and re-walks the trail loses half the rain's time
  (`income × (1 − 0.5 · rain share)`). One who lays trails into rain and never re-walks loses one
  and a half times it.
- **Spiders** arrive per §4.2 (about 5–6 a year for a colony that reaches tier 2 around day 10,
  at most 2 at once) and, while undefended, follow the traffic (§4.5) and take 4 workers a day
  (§4.4). Each kill weakens the trail (§3.3), and the model charges an undefended spider 35% of the
  income of the hauls it sits on. That figure is an estimate of recruiting slowed by a trail
  repeatedly cut to 0.7×, and it is the model's largest uncertainty. A defence with a full party
  costs no worker (§5.5).
- **Birds**: 1.6 workers a strike, 70% of birds finding a target.
- **Raids** run the exact §7.4 arithmetic against `Population − 6` in the nest (the haulers
  carrying when raiders arrive). They come on about days 13, 20 and 25, from a rival growing 30 → 83.

### 16.2 Reasonable player (η = 0.6, builds, defends)

The SIM_M2 §14.2 player, who also marks a spider within about 2 minutes of it settling on their
trail, raises the alarm for one bird in four, is home for every other raid, and re-walks trails
after rain.

| End of day | Workers | Brood | Food | Tier | B/S/P | Threats that day |
|---|---|---|---|---|---|---|
| 0 | 12 | 8 | 49 | 1 | 1/1/0 | Calm (day 0). |
| 4 | 16 | 20 | 52 | 1 | 2/1/0 | Rare spring spider; birds from day 2. |
| 8 | 24 | 29 | 85 | 1 | 3/1/1 | |
| 10 | 29 | 33 | 138 | **2** | 3/1/1 | Summer: spiders and birds pick up. |
| 13 | 38 | 35 | 175 | 2 | 3/2/1 | Raid: 19 vs 34, repulsed; 2 lost, 22 food looted. |
| 16 | 51 | 35 | 235 | 2 | 3/2/1 | |
| 20 | 67 | 33 | 256 | 2 | 3/3/1 | Raid: 21 vs 64; 2 lost, 21 food. |
| 25 | 89 | 20 | 294 | **3** | 3/4/1 | Raid: 30 vs 85; 2 lost, 21 food. |
| 29 | 101 | 17 | 339 | 3 | 3/4/1 | Shelter 1.7. |
| 33 | 111 | 7 | 53 | 3 | 3/5/1 | Winter: no threats. |
| 35 | 103 | 1 | 0 | 3 | 3/5/1 | Stores run out. |
| **39** | **70** | 0 | 0 | 3 | 3/5/1 | **Year ends with 70.** |

Over the year threats take **16 workers** (spiders 2, birds 8, raids 6) and 64 food. That is 13% of
the 121 workers the colony raises. Against the calm year: the peak falls 132 → 114, Mature comes on
day 22 instead of 20 (so fewer pinecones: shelter 1.7, not 2.4), and the year ends with 70, not 88.
**The reasonable player ends the year alive, 20% down**, most of it to rain and loot (food), not
to deaths.

### 16.3 Other players

| Player | Peak | Lost to threats (spider/bird/raid) | Food looted | Workers as winter breaks | Calm year (SIM_M2) |
|---|---|---|---|---|---|
| Keen, η = 0.8, defends fast, always home for raids | 123 | 11 (0/6/5) | 54 | **116** | 134 |
| **Reasonable**, η = 0.6 | 114 | 16 (2/8/6) | 64 | **70** | 88 |
| Reasonable, defends slowly (~3.5 min) | 109 | 23 | 67 | **69** | |
| Reasonable, never home for a raid | 111 | 20 | 88 | **68** | |
| Reasonable, but ignores rain | 107 | 16 | 64 | **53** | |
| Reasonable, but ignores Defend (no marks, no alarms, never home for a raid) | 87 | 47 (26/11/10) | 92 | **60** | |
| Ignores both | 79 | 47 | 92 | **45** | |
| Casual, η = 0.45 | 77 | 14 | 20 | **29** | 46 |

**Ignoring rain costs 17 workers at year's end**: it is a straight loss of income, and food is what
the year's end is made of. **Ignoring Defend** costs three times the workers dead (47 against 16;
spiders alone 26 against 2), 28 more food looted, a peak 27 lower and Mature nine days later (day
31, as winter begins, against day 22), but only **10 at the headline**.

Why so little at the headline: in SIM_M2's economy the year ends **food-bound**. A colony grows to
whatever its brood chambers allow, then winter starves it back to what its stores can feed
(SIM_M2.md §14.2: the calm reasonable colony starves 46 in winter). A worker killed in summer is
a mouth not fed in winter, so about three quarters of every summer death is refunded as less
starvation. Only food lost moves the headline much, which is why spiders weaken trails, raids
loot, and rain washes trails: those are the threats' teeth. The sandbox after day 40 does not
refund anything: an ignoring colony enters year two with 10 fewer workers, no shelter to speak of, and a
rival at full strength.

Whether threats should hit the headline harder is a question about the headline measure
(GAME.md: workers as winter breaks, itself an assumption), not about threats. If neglect should
show more there, the cheapest lever is to put threat deaths on the outcome screen beside the
workers; the stats already hold them.

### 16.4 Feedback loops

- **Rival**: logistic growth toward 100, minus every raider killed. Repulsing raids holds it near
  70–85 through autumn. It converges; nothing you lose feeds it (§7.1).
- **Raid size against your nest**: raid size grows with your tier, defence with your population,
  and loss falls as `R0² / D0`. A nest that keeps growing converges to 1–3 workers lost per raid.
  A nest that shrinks (starving, a tier-1 colony in a later year) faces raids that breach, lose it
  half its workers and loot its food: **a runaway for a failing colony in the sandbox**. In year one
  it cannot happen before autumn, and an autumn colony at tier 2 still repulses (21 vs 40+).
- **Spider on a trail**: kills weaken the trail, which sends fewer workers past the spider. It
  converges to a starved trail, not to a dead colony, and re-walking the trail restarts it.
- **Rain**: no loop; a fixed tax per season.

### 16.5 `SimConfig` retunes

**None.** No existing default changes. The calm-year numbers in SIM_M2.md §14 stay correct for a
calm world (`ThreatsEnabled = false`). If the user wants the reasonable player's headline back near
86 with threats on, the cheapest lever is find spawn rates about 10% higher (data in
`Assets/Settings/Items`, not `SimConfig`). That is not done here, because it would also lift the keen
player to ~125.

---

## 17. Known degenerate strategies and failure modes

- **Death as a ride home.** Dying costs one worker and returns you to the nest after 10 s. From a
  far find next to a spider, walking into it (3 s) beats walking home (20 s). It costs a worker
  each time, about 3.5 food and a week of brood, so it is weak. The fix if it shows up is a longer
  respawn.
- **Re-walking any time.** A recruiting trail can be walked again whenever you like, not only after
  rain. It is labour for strength, the same trade marking already is. Intended.
- **Birds are mostly a tax.** The alarm needs you within 10 u of a shadow you have 5 s to see. That
  happens on the trail you are walking and at your own hauls, not elsewhere. Intended: it rewards
  staying with your workers.
- **Big nests make raids free.** Past ~60 workers at home a raid costs about one worker and a few
  food. Raid size scales with tier to keep raids felt, but tier stops at 3. Year two in the sandbox
  will feel this.
- **Turtling** (never sending anyone out) beats raids and starves. Not viable.
- **A spider cannot be outwalked.** It follows the colony's traffic within about a minute (§4.5),
  so marking finds in another direction buys only that minute. Defending is the answer, and a full
  party is free in lives (§5.5), so the real price of a spider is six workers' time and the
  player's walk to it. If spiders feel like a chore rather than a threat, raise
  `SpiderRelocateSeconds` (they settle and can be avoided) or `SpiderFightKillSeconds` below 8.4 s
  (a defence costs a worker again).
- **Two short parties do not beat one full one.** A party waits until it is full before it attacks,
  so trickling defenders in is not possible; starting a worker short with the player standing in
  costs one.
- **A defend trail that wiggles near its end** can put the muster point inside the spider's
  ground: the standoff is measured along the trail, not in a straight line. Rare, because players
  walk straight home. If it shows up, measure the muster point by straight-line distance from the den.
- **A spider on the nest's doorstep** cannot happen (25 u clearance). Trails shorter than 25 u are
  therefore spider-safe, and players may learn to farm near finds first. That is mild and leaves
  the near ring as the safe, poor ground.
- **Tap prefers a defence.** While a defence recruits, Tap cannot hurry the haul. Intended
  priority; rarely matters, since a defence fills in seconds.
- **A raid during a long haul** recalls nothing that is carrying, so the party can arrive into
  the fight and join it. That is fine, but the HUD's defender count will jump.
- **The model's spider cost is an estimate** (35% of the income of an undefended spider's trail).
  The `BotYear` run is the real check, and the first thing to look at if spiders feel weak or brutal.
