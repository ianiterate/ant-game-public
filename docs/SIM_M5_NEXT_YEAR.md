# Simulation spec — next year: the run

Builds on [SIM_M2.md](SIM_M2.md) §6/§10, [SIM_M3.md](SIM_M3.md) §7/§16 and
[SIM_M4_PROCESSING.md](SIM_M4_PROCESSING.md) §8, and replaces only what it says. Copy:
[UI_COPY.md](UI_COPY.md#years).

**In one paragraph.** When winter breaks, the sim records the year's outcome and **holds**. The card
offers *Carry on to year N+1* or *Stay in the garden*. Carrying on starts a new spring with everything
the colony has, one step harder: fewer finds, more threats, a stronger rival and a colder winter, and
one new thing each year up to year 5, where the curve stops. Staying keeps the garden as it is, with
no more cards. A run ends when the colony dies; the headline is the number of years it survived.

**Assumed** — *next year* rather than a second species. It is data, one hold, one save bump and one
small prop. A species is a second economy to balance plus new ant and nest art, and it gives a second
*first* year rather than a reason to go on. Cost to change: low. Nothing here blocks a species later
(a choice at new colony; the run applies to it unchanged); only the order of work changes.

---

## 1. Run state

```csharp
public enum RunMode { Run, Sandbox }
// World: int Year (1-based), int Level, RunMode Mode, bool AwaitingYear,
//        YearOutcome[] Years (one per ended year, oldest first, ≤ SimLimits.MaxYearRecords = 100),
//        ColonyStats RunStats (totals over ended years; PeakPopulation/PeakDay = the best).
// Read API: Year, Level, Mode, AwaitingYear, Years (span), RunStats, LevelRow, YearsSurvived,
//           DayOfYear = Day − (Year − 1)·YearDays, Season (the effective season, §3.3).
```

`Stats` becomes **this year's** totals, reset at each roll. `Outcome` holds the latest ended year.

## 2. The year's end and the choice

**End** (ColonySystem step 9, replaces SIM_M2.md §6.2's trigger): on the tick a day starts with
`Clock.Day == Year · YearDays` and the colony alive, record `Outcome` (Survived), append it to `Years`
and raise `YearEnded` as today. Then, in `Run`, set `AwaitingYear = true`. In `Sandbox`, roll the year
at once (§2.1, same level) and do not hold.

**The hold.** While `AwaitingYear`, `World.Tick` clears events, runs only `CommandSystem` and returns:
no clock, no systems, no draws. Every other command is rejected (`YearEnding`). Because the choice is
a command, replays and saves reproduce it, and a reload at the hold reopens the outcome card.

**Commands** (rejected `NotYearEnd` unless `AwaitingYear`):

- `BeginYear`: `Level = Min(Year + 1, YearLevels.Length)`, then roll (§2.1).
- `StayInGarden`: `Mode = Sandbox`, `Level` unchanged, then roll. Later year ends append a record and
  roll on with a toast, no card. There is no way back into the run.

### 2.1 What carries over and what resets

| Carries over, untouched | Changes at the roll |
|---|---|
| Workers in every job, brood pipeline, food, carcasses being cut | `Year++`; `Level` per the command |
| Chambers and any dig under way; tier (it is derived, with its margin) | Thatch wears: `Shelter = Shelter / 2` (integer) |
| Trails (they decay as always; a late-autumn trail is long gone) | Rival: `Strength = RivalStartStrength · RivalScale` (it winters too) |
| Finds on the ground, claimed or not (winter leaves almost none) | `Stats` rolls into `RunStats`, then resets |
| Weather; a raid in progress (winter raids, from year 2) | Both random streams reseeded (§4); every pending threshold redrawn |
| The player ant, alive or down (its respawn timer runs as usual) | Threat progress accumulators set to 0 |

No starting finds are placed; the empty first half-day is part of the **lean spring** (§5). The Game
layer puts the player ant at the nest entrance on `YearBegan` (presentation only). **Assumed** —
thatch halves, the rival resets to a scaled start, no restock. Cheap: three lines in the roll.

**Assumed** (implementation) — the per-kind `FindsHauled` counts are the year's, like `Stats`: they
reset at the roll, so the year's "Finds" line and the spring cube goal count this year only. Spawn
progress carries over (only thresholds are redrawn). The player ant is moved to the nest on
`YearBegan` in a run only; in the sandbox the year rolls on under the player with a toast. Cost to
change: a line each.

**Roll order** (deterministic): Year/Level/Mode → `RunStats += Stats`, `Stats = default` → shelter →
rival → reseed → spawn thresholds in `ItemKind` order (main) → threat thresholds in `ThreatKind` order
(hazard), progress 0 → raise `YearBegan(A = Year, B = Level)`. It runs in step 3 of the command's
tick; the rest of that tick runs normally.

**Assumed** (implementation) — a held tick applies its commands first, as the hold's only step; when
one of them rolls the year, that tick then runs steps 1–10 in full (clock, weather, draft sampling,
…) without applying the commands again. So the roll comes before that tick's clock and weather
steps rather than between them. Deterministic and saved the same way; cost to change: the order of
four lines in `World.Tick`.

## 3. Difficulty per year

### 3.1 The level table

`SimConfig.YearLevels: YearLevel[5]`. Row 1 is the identity: year 1 is bit-for-bit today's game.

```csharp
[Serializable] public struct YearLevel {
    public float FindRate;          // × every find's spawn rate
    public float ThreatRate;        // × SpidersPerNight, BirdsPerDay, RaidsPerDay, WinterRaidsPerDay
    public float RivalScale;        // × RivalStartStrength and RivalCapacity
    public float WinterUpkeepAdd;   // + WinterUpkeepFactor
    public float WinterRaidsPerDay; // raids a day in winter, before ThreatRate (RaidsPerDay[Winter] stays 0)
    public int   EarlyWinterDays;   // autumn's last days that are winter (§3.3)
    public int   SpiderMax;         // spiders at once (≤ SimLimits.MaxSpiders)
}
```

| Level (year) | 1 | 2 | 3 | 4 | **5 and on (cap)** |
|---|---|---|---|---|---|
| `FindRate` | 1 | 0.92 | 0.85 | 0.80 | 0.75 |
| `ThreatRate` | 1 | 1.25 | 1.5 | 1.75 | 2.0 |
| `RivalScale` (start / ceiling) | 1 (30/100) | 1.15 (35/115) | 1.3 (39/130) | 1.9 (57/190) | 2.4 (72/240) |
| Winter upkeep (`1.25 + Add`) | 1.25 | 1.30 | 1.35 | 1.40 | 1.45 |
| `WinterRaidsPerDay` | 0 | 0.1 | 0.1 | 0.1 | 0.1 |
| `EarlyWinterDays` | 0 | 0 | 0 | 2 | 2 |
| `SpiderMax` | 2 | 2 | 2 | 2 | 3 |
| **New this year** | — | Raids in winter | Fallen berries | Winter two days early | Three spiders |

*Earlier* threats: the calm first day is year 1's only (`Day != 0` is unchanged), and the rival starts
each year stronger, so spring raids are bigger. **Assumed** — every number, and the cap at year 5.
Cheap: data.

### 3.2 The new find: fallen berry

`ItemKind.Berry` (appended; `ItemKinds.Count` 6 → 7): 6 ants, tier 1, 60 food, radius 1.6 u (the
mesh's half-width), spoils in a day, not `Raw`; 0.15 a day in summer and autumn, at most 1, 40–110 u out. `ItemDef` gains `int FromLevel`
(0 = always; berries 3) and the spawner skips a kind below it. About +8% summer and autumn food; tier 1
so it helps a shrinking colony most. One small prop; the name is `world-builder`'s. **Assumed** —
the find and its numbers. Cheap, except removing it later (a migration of the per-kind arrays).

### 3.3 Early winter (rather than a longer year)

`World.Season` is the **effective** season: Winter when `Clock.Season == Autumn` and the day in season
`>= DaysPerSeason − EarlyWinterDays`, else `Clock.Season`. Every sim rule that reads a season (spawn,
laying, upkeep, weather row, threat rates) moves to it; `SeasonChanged` fires on it, so the winter
toasts come on day 28; `WinterFoodNeed` × `(DaysPerSeason + EarlyWinterDays)`. The year stays 40 days.
**Assumed** — winter grows into autumn rather than lengthening the year. A longer year costs 1–2 days:
a stored year start in the clock and the save, and every `YearDays` reader changed.

## 4. Determinism

Unchanged in kind: one build target, float maths, two streams. At each roll both streams restart from
the year's seed. Year 1 is the colony's seed, so year 1 does not change:

```csharp
int YearSeed(int year) => unchecked(Seed + (year - 1) * 1_000_003);   // K = 1 000 003, prime
_rng = new Rng(YearSeed(Year)); _hazardRng = Rng.ForHazards(YearSeed(Year));
```

A large K stops colony *s*'s year 2 replaying colony *s + 1*'s year 1 (K = 1 would). Reseeding makes a
year's draws a function of (seed, year), not of earlier years' draw counts: a reload at the hold cannot
reroll next spring, and a test can start "year 3" from a hooked world. `StateHash` folds `Year`,
`Level`, `Mode`, `AwaitingYear`, `RunStats`, `Years`. **Assumed** — K and the reseed. Cheap until it
ships; afterwards every year-2+ hash moves.

**Assumed** (forced by keeping the golden fixtures) — rather than re-recording the hash constants,
`StateHash` folds the run in a last section only once the world has left year 1's defaults (year 1,
level 1, `Run`, not holding, no records, the berry's per-kind slots all 0); the per-kind loops of the
earlier sections stop at the six first-year kinds. Every year-1 hash, the v1–v3 fixtures' included,
is unchanged, and two worlds that differ in any run field still hash apart. Cost to change: none
until a constant is re-recorded.

## 5. Balance: years 2–5

**The fight decides it.** A raid meets nearly the whole colony: the recall (SIM_M3.md §2) brings the
foragers home before the column arrives, and the defending bot is home too, so the defenders fight at
×1.5 (`RallyFactor`). Under the square law with both sides breaking at half, a raid of `R` breaks in
only when `R > √1.5 · D ≈ 1.22 · D`, `D` the workers in the nest. Below that line it is repulsed at a
cost of about `R² / (4 · D)` workers; above it, the nest loses half its workers and its food. The line
is sharp, so a raid is never a coin flip unless the colony sits right on it.

**Why the first table never ended a run.** Measured (2026-10-10), the defending bot met raids of
21–41 against 22–111 defenders in every year to 7 and was breached at most once per seed. Three
reasons, in order of weight:

- *The rival never reached its ceiling.* Every repulsed raid takes at least half its raiders off the
  rival, and about ten raids a year at ×2 `ThreatRate` held it at 100–120 against a ceiling of 160.
  Raising `ThreatRate` makes this worse, not better: more raids drain the rival faster, so each raid
  is smaller. Raid size is the lever; raid count is not.
- *A shrinking colony drops a tier.* Under 25 workers the raid takes 0.15 of the rival, not 0.25, so
  the colony that most needed to be breached met raids 40% smaller. At a 160 ceiling that is at most
  24 raiders, below the 1.22 · 20 a starved winter nest still musters.
- *Winter size is set by storage, not income.* The bot ends every year with no food left; its
  winter colony is what its store chambers can feed. Lowering `FindRate` barely moves it (the bot at
  1.6× income ends year 1 with 37–56, against 36–64), so the planned `FindRate` cut did little.

Winter raids do loot, during the fight and for 30 s after a breach. No mechanic is missing; the
numbers were too low.

**The retune: a much stronger rival in years 4 and 5.** `RivalScale` 1.45 → **1.9** and 1.6 → **2.4**
(ceilings 190 and 240); nothing else changes, so years 1–3 are as before. A breach barely weakens the
rival (it loses 3–15 raiders), while a repulse costs it half the raid. So once a colony is breached,
the rival climbs to its ceiling and keeps breaking in: the runaway. A colony stays clear only while
its nest holds more than `R / 1.22`. At the cap that is about 37 workers against a tier-2 raid with the
rival held at 180, and 50 against one at the full ceiling. A tier-3 colony (75 or more) is never
breached, since 0.35 · 240 = 84 < 1.22 · 75.

**Measured.** `YearModelTests.BotRun`: the defending bot with `ColonyKeeper`, carrying on every year.

| Workers as winter breaks | Y1 | Y2 | Y3 | Y4 | Dies |
|---|---|---|---|---|---|
| Seed 1 | 43 | 47 | 44 | 10 | Year 5, day 8 (spring) |
| Seed 2 | 46 | 41 | 29 | 17 | Year 5, day 28 (autumn) |
| Seed 3 | 46 | 40 | 44 | 8 | Year 5, day 35 (winter) |
| Seed 4 | 36 | 40 | 32 | 19 | Year 5, day 29 (autumn) |
| Seed 5 | 64 | 33 | 40 | 14 | Year 5, day 36 (winter) |

Every seed survives four years. Three of the five are first breached by year 4's last winter raid
(rival 138–157, 35–39 raiders against 20–28); the other two fall to year 5's summer raids, 39–45 against
30–35. From the first breach the colony is dead within the year. Raid timing is chaotic: the same
table with years 2–3 at 1.2 and 1.45 instead of 1.15 and 1.3 moved one seed's death into year 6. Read single seeds
as ±1 year. The first table, for comparison: 7 years survived on every seed, ending years 4–7 on
8–34 workers.

**Assumed** — the keen player holds past year 5. No keen bot exists; this is the arithmetic above.
Year 1 ends at about 115 for keen play against the bot's 43. If that keeps the keen colony's winter
nest above about 50 (tier 2) at the cap, every raid is repulsed at 8–12 workers. A keen player who
lets the winter nest fall under about 37 is breached like the bot. Cost to change: data. If playtests
show keen runs ending too, lower year 5's `RivalScale` toward 2.2. That table measured deaths in years
5–6 (5, 5, 5, 4, 4 survived).

**Feedback loops.** The rival resets each spring and, against a colony that repulses, converges at
about 0.7 of its ceiling; it runs to its ceiling against one that is breached. The colony converges
for any play while its winter nest stays above the breach line, and diverges below it. That runaway is
how a run ends, on purpose. The **lean spring** is also intended: year 1 starts small and grows,
later years start big with empty stores and an empty garden, so the headline falls from year 1 to 2
for everyone. *Stay in the garden* stays at level 1 and is unaffected (`BotRun_SandboxHolds`).

## 6. Save v4

```csharp
public const int CurrentVersion = 4;   // MinLoadableVersion stays 1
public int Year, Level, Mode;          // 0/0/0 in a v3 save → migration
public bool AwaitingYear;
public StatsSave RunStats;
public OutcomeSave[] Years;            // OutcomeSave gains: public int Year, Level;
// Per-kind arrays (SpawnProgress, SpawnThreshold, FindsHauled) grow to ItemKinds.Count = 7.
```

Only the level's index is saved; the row comes from config, so a retune applies at load as any retune
does (SIM_M2.md §10.4).

**Migration v3 → v4** (`V3ToV4`, chained after v1 → v2 → v3):

- Per-kind arrays padded to 7 with 0 (the berry's threshold is drawn at a roll; below level 3 it is
  never read). `AwaitingYear = false`.
- Still in year 1 (`!Outcome.Recorded`): `Year 1, Level 1, Run`, `Years` empty, `RunStats` zero.
- Past its year end (`Recorded && Survived`, i.e. in today's sandbox): `Sandbox, Level 1,
  Year = Day / YearDays + 1`, `Years = [Outcome]`, `RunStats = Outcome.Stats`, `Stats −= Outcome.Stats`
  per field (peak kept). It plays on exactly as before.
- Dead: `Run`, `Year = EndDay / YearDays + 1`, `Years = [Outcome]`.

A migration step does not see the config, so the sandbox branch saves `Year = 0` and `World.TryLoad`
sets it from the clock (`Day / YearDays + 1`); a v3 record is always year 1's, so the others are set
in the step. A v4 save never carries `Year = 0` otherwise.

**Assumed** — a v3 sandbox colony stays in the sandbox and is not offered the run. Cheap: one branch.

Validation (`Corrupt`): `Year >= 1`; `1 <= Level <= YearLevels.Length` (a higher saved level is
clamped, not an error); `Mode` in range; `Years.Length <= MaxYearRecords`; `AwaitingYear` only if
`Mode == Run` and `Day == Year · YearDays`.

## 7. Tests (`Assets/Tests/EditMode`)

- **`YearTests`** (new): `YearEndHolds` (100 ticks: clock, hash, events unchanged); `BeginYearRolls`
  (Year 2, Level 2, `YearBegan`, stats rolled, shelter 3 → 1, rival 34.5, streams equal
  `new Rng(Seed + K)` after the redraws); `StayFreezesLevel` (day 80: a record, no hold);
  `CommandsRejectedWhileHolding`; `BeginYearRejectedOutsideHold`; `LevelCapsAt5`; `LevelOneIsIdentity`
  (a year with the table against one without: identical hashes); `EarlyWinter` (level 4: day 28 is
  Winter, no spawn or laying, winter upkeep, need × 12); `WinterRaidsFromLevel2`; `BerriesFromLevel3`;
  `SpiderMaxByLevel`; `ThreatsOnDay40`; `DeathEndsRun` (year 2: two records, no hold).
- **`DeterminismTests`**: `Run_SameSeedIdentical` (through the hold, `BeginYear`, 20 000 more ticks);
  `YearSeedsDiffer`. Existing seeds stay; hash constants re-recorded once (§4's fields).
- **`SaveRoundTripTests`**: at the hold; mid-year-2 during a winter raid; `Years`, `RunStats` exact.
- **`SaveMigrationTests`**: `save_v1/v2/v3.json` → `Migrated`, Year 1, Run; a synthetic v3 sandbox save
  → Sandbox, stats split; `StepsCount == 3`; golden `save_v4.json` (`[Explicit]` maker: defending bot,
  seed 1, early year 2).
- **`YearModelTests.BotRun`** (`[Explicit]`, `Balance`): defending bot + `ColonyKeeper` carry on every
  year, seeds 1–5, until death or year 7; per-year workers logged. Assert: every seed dies in year
  5 ± 1 (survives 3–5 years); measured 4 on all five (§5). `BotRun_SandboxHolds`: stay after
  year 1; every seed alive at the end of year 4.
- **Game layer**: goal tracker resets per year; the card's two actions; spring card on `YearBegan`;
  reload at the hold reopens the outcome card.

## 8. Known degenerate strategies and failure modes

- **The lean spring reads as failure** (§5): year 2's count is lower than year 1's even for a better
  player. Hence *Last spring* on the card and years survived as the run's headline. If it still reads
  as punishment, the cheapest fix is restocking the starting finds at each roll.
- **Staying small to keep raids small.** Under 75 workers a colony stays tier 2 and meets 0.25 of the
  rival, not 0.35. At the cap that only helps above the breach line (§5, about 37–50 in the nest), and
  under 25 (tier 1, 0.15) the full-ceiling raid of 36 still breaks a nest that small. Weak.
- **The last winter raid decides the run.** It meets the year's smallest colony and a rival that has
  regrown all winter, so most runs turn on one fight in days 35–39. Its loot hardly matters (the
  stores are nearly empty by then); the half of the nest it kills does. Intended, but it makes the end
  feel sudden: the card names the breach.
- **Long runs are long.** A year is about four hours; four years is about 16. Shorter later years
  would be a clock change.
- **Keen play may never end**: a colony that keeps its winter nest above the breach line holds at the
  cap indefinitely (**Assumed**, §5). Intended (the cap is livable for the best play only). If every
  run should end, let `RivalScale` keep rising past year 5. `ThreatRate` would not do it: more raids
  drain the rival faster (§5).
- **The bot is the only measurement.** Raid timing is chaotic, there is no keen bot, and spiders are
  costed in lives, not in the player's time (about 11 defences a year at 2×). `BotRun` is the real
  check; `RivalScale` at levels 4–5 is what to retune.
