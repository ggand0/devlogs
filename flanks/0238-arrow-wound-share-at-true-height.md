# 0238: the arrow wound share, measured per kind against 0.2.1

Written by Claude Opus 5.5 (the measurement and the 0.8 commit), finished by Claude Fable 5.1 (the knights, 0.83). Branch `fix/archer-rebalance` on main e3fa6be, 2026-10-12. Handoff 112 opened this; handoff 113 closes it.

## The issue

Since the soldiers are 1.70 m tall (PR #25), archers kill up to 1.9 times faster than in 0.2.1. The cause (devlog 0211, step 7): an arrow strikes a man whose axis lies within 0.35 m of its path and whose body spans the arrow's height; the body got taller, the arrows did not change.

## The fix

Gota's parked commit (805ae53, cherry-picked as 136f5c1): one roll per strike decides whether the arrow wounds or glances off. A glancing arrow is heard and does nothing. The share was 0.52 there, measured as too low (29 dead against the 146 target at 36 s). This branch measured the share per kind and set it to 0.83.

## How it was measured

The archery scene (`FL_TEST_ARCHERY`): two units of 500 bowmen fire on a 500-man unit that stands still 110 m away. `FL_ARCHERY_TARGET` picks that unit's kind; `archer` is new (bowmen in the middle, who shoot back). The new tally in the log (`[archery-tally]`, every 120 ticks from the fixed tick, so runs compare tick for tick) counts arrows loosed and ended and, by the victim's team and kind, the men struck, wounded and killed. The same tally was patched into 66044d1 (0.2.1's rules; `tmp/runs/archer-rebalance/old-src`, binary `bin/flanks-old`; its fingerprints equal `tmp/runs/scripts/gates/ss-main-arch.hash`, so the tally changes nothing in the sim) and into main e3fa6be (`bin-main/flanks-main0`). The script `tmp/runs/archer-rebalance/arch.sh <binary> <name> <kind> [ENV]` runs one scene; the logs are in `runs/`.

### Strikes per arrow

Of the arrows that ended their flight by tick 1080 (36 s), the share that struck a man:

| Target | 0.2.1 (1.0 to 1.1 m men) | Main (1.70 m men) | Ratio |
|---|---|---|---|
| Men-at-arms | 2,094 of 3,046 (0.69) | 2,479 of 3,094 (0.80) | 1.17 |
| Spearmen | 2,097 of 3,005 (0.70) | 2,481 of 3,020 (0.82) | 1.18 |
| Knights | 2,175 of 3,005 (0.72) | 2,468 of 3,016 (0.82) | 1.13 |
| Bowmen (tick 240) | 496 of 1,164 (0.43) | 582 of 1,182 (0.49) | 1.16 |

So the taller body takes 13 to 18 percent more strikes per arrow, not 1.9 times. Knights were 1.1 m tall before and grew least.

### Why the dead rose more than the strikes

Arrow damage adds up against fixed hit points, so a man dies at a fixed count of strikes for his kind. The strikes fall on the front of the unit: in the 0.2.1 build at 36 s, 1,012 of the 1,353 living men-at-arms were unhurt, and the rest carried the 2,094 strikes between them (a throwaway log of the hit point spread, `bin/flanks-hpo` and `bin-hp-branch/flanks-hpb`, not committed). When those men take 17 percent more strikes, far more of them cross the count that kills: 282 dead against 147 at 36 s. A share that returns the wounds per arrow to the old count (1/1.17, about 0.855) still overshoots the dead by 13 percent (451 against 400 at 52 s), because the wounds then fall on fewer, taller men.

### The dead per share

Men-at-arms at 52 s (tick 1560), of 500:

| Share | Dead |
|---|---|
| 0.2.1 | 400 |
| 1.0 (main) | 498 |
| 0.855 | 451 |
| 0.83 | 421 |
| 0.80 | 392 |
| 0.75 | 347 |
| 0.52 (the parked number) | 29 at 36 s, main's 279 |

Spearmen over two minutes (tick 2520, 3000, 3600): 0.2.1 has 84, 227, 408 dead; main 26 dead already at 52 s and all 500 by two minutes; 0.80 gives 100, 218, 390; 0.83 gives 103, 237, 414.

Knights over two minutes (tick 3000, 3480, 3600): 0.2.1 has 13, 38, 53 dead; main 47, 93, 125; 0.80 gives 15, 30, 39; 0.83 gives 18, 37, 44; 0.88 gives 23, 55, 71.

Bowmen (they shoot back, so the exchange is over in 32 s): at 16 s 0.2.1 has 376 dead, main 460, 0.80 402, 0.855 411. All 500 dead by 32 s on every build.

### The number

0.83 is the one share within about a tenth of 0.2.1 for every kind: men-at-arms 5 to 10 percent over, spearmen within 4 percent, knights 17 percent under at two minutes and 3 percent under at 116 s (37 against 38; the knight counts are small and move by ten men between samples). 0.80 matched men-at-arms and spearmen to the man but left knights a quarter short; 0.88 matched knights and overshot the rest. A share per kind would fit each better; it is one more table, and the reading that plate knights are wounded by more of the arrows that hit them would need an explanation the measurement does not give.

## Checks

Fingerprints against main e3fa6be (`work/scripts/gate.sh`, prefixes `ar-main`, `ar-branch`, `ar-branch-w100` in `tmp/runs/scripts/gates/`): the direction scene (18), the wide pile (28) and the two-on-one pile (28) equal; the archery scene with `FL_WOUND=1` equal (18), so the tally itself changes nothing; the archery scene at the share differs, as it must. Clippy clean with `-D warnings`, 44 tests pass. A 200k AI battle (`FL_AUTOSTART=1 FL_DEPLOY=0`) ran 75 s with no panic or error at the 0.8 share.

## Figure

`work/notes/vis/046-arrow-wound-share.png` (script `work/scripts/viz/arrow_wound_share_046.py`): dead over time per kind for 0.2.1, main and the branch. Drawn from the 0.83 runs (`w83`, `w83-125`).

## Commits

136f5c1 the parked share (Gota's commit), 2d9b6d4 the tally and `FL_ARCHERY_TARGET=archer`, 1815883 the share at 0.8, 4c601b2 its docs, c91cd6f the share at 0.83.

## Implementation (added 2026-10-12 on Gota's ask)

All of it is in `update_arrows` (`src/arrows.rs`) and the `missile` constants (`src/unit_types.rs`). Nothing else in the sim changed.

**The flight and the strike, unchanged.** Every tick an arrow falls under gravity and moves one step. The step, from where the arrow was to where it is now, is tested against the men near it in the spatial grid. The first man along the step whose axis lies within `HIT_RADIUS` 0.35 m of the step and whose body spans the arrow's height there (his centre plus and minus `half_height` 0.85 m, 5 cm of margin on top) is struck. The arrow is removed from the pool and a `Hit` record is queued; the hits are applied in order after the sweep, so one arrow can kill and the next arrow sees the corpse. Dying men (`META_DYING`) are skipped.

**The roll, new.** When the hit is queued it carries `wound: hash01(seed ^ 0x6A09_E667) < wound_share()`. The seed is the one the damage jitter already uses (the tick and the arrow's pool index), with a different salt, so the roll is deterministic (the fingerprints hold) and independent of the jitter. `wound_share()` reads `missile::WOUND_SHARE` 0.83 once, `FL_WOUND=<share>` overrides it (clamped 0 to 1).

**The apply.** For each hit: the sector the arrow comes from against the man's facing (front, left, rear), the shield counts in front and on the left, the armour everywhere; the factor is `missile::ATTACK` 6 minus armour and shield, clamped to plus or minus 12; the damage is `BASE_DMG` 20 times 1.28 to the factor, times a jitter of 0.85 to 1.15, times the combat scale. The arrow hit sound plays in every case (a shaft that glances off is heard on shield or armour). If the roll said glance, the loop goes to the next hit: no damage, no flash, no kill bookkeeping, no stagger. If it said wound: the hit points drop, the man flashes; at zero he dies (`death_t`, the kill counters, the morale counters of both units); above zero he takes the same stagger roll a melee blow gives, as before.

**Wounds a man takes to die**, at the front, from the numbers above: men-at-arms factor minus 1, 15.6 damage, 90 hit points, about 6; spearmen minus 4, 7.5 damage, 100 hit points, about 14; knights minus 4, 7.5 damage, 160 hit points, about 22; bowmen plus 5, 69 damage, 80 hit points, 2. The share does not change these counts; it changes how many of the strikes count as wounds.

**The tally** (`ArrowStats`): `loosed_total` counted at the drain, `struck`, `wounded` and `killed` as per-team, per-kind arrays counted at the apply. `log_tally` prints them from `update_arrows` on every tick divisible by 120 when `FL_TEST_ARCHERY` is set (`tally_log()`, read once). `spawn_archery_test` (`src/regiments.rs`) takes `FL_ARCHERY_TARGET=archer`.

**The fixes, in order.** Gota's roll per strike (136f5c1) at 0.52; the tally to measure instead of deriving (2d9b6d4); the share at 0.8 from the first measurement (1815883, 4c601b2); knights measured over two minutes and the share at 0.83 (c91cd6f). Not touched: the flight, the 3 m scatter, the strike radius, the strike band, the shot cadence, the projectile spawn.

## Kills per arrow, against 0.2.1 and M2TW (Gota's question, 2026-10-12)

The share of 0.83 is per strike, not per arrow: it decides whether an arrow that hits a man wounds him. The number a player feels is kills per arrow fired, which the tally gives directly (kills over arrows that ended their flight, on a body or on the ground):

| Target | 0.2.1 | This branch |
|---|---|---|
| Men-at-arms (shield, mail), 52 s | 9.0 percent (400 of 4,441) | 9.5 percent (421 of 4,442) |
| Spearmen (big shield, mail), 2 min | 4.1 percent (408 of 10,072) | 4.1 percent (414 of 10,074) |
| Knights (plate), 2 min | 0.5 percent (53 of 9,971) | 0.4 percent (44 of 9,980) |
| Bowmen (no armour, no shield), 16 s | 17.1 percent (376 of 2,195) | 18.1 percent (404 of 2,235) |

Per 100 arrows against a standing block of men-at-arms: about 78 hit a man, 65 wound, 9 or 10 men die. M2TW's measured rate (Withwnar's tests, devlog 0060) is about 10 percent of arrows killing against unarmoured peasants, so the branch sits in the same ballpark, with the armour spread the 0.2.1 calibration chose (devlog 0064). Gota's reading, 2026-10-12: the same ballpark as M2TW; documented here and in the `WOUND_SHARE` comment. The bowmen rate is higher than M2TW's unarmoured anchor because a bowmen block is the only unarmoured target in the game and the exchange is short (both sides shoot, 500 men dead in 32 s).

## 2026-10-12: merged

PR #29 "Fix archers killing too fast since the soldiers became life-size", merged with a merge commit, main 58f90d3. Six commits kept. Final checks before the merge: the full diff against main read, clippy clean with `-D warnings`, 44 tests, the PR body and the commit messages text-checked. Open after it: whether knights get their own share (0.88 would match them alone), and M2TW's one-roll shape without hit points for arrows as a later feel decision.
