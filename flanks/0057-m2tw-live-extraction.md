# 0057 — M2TW live morale extraction: FIRST REAL VALUES (2026-07-26)

Companion to 0056 (research record) and 0055 (implementation). This is
the first time this project — or, as far as the public record goes,
anyone — has read M2TW's morale effect table out of the running engine.
Tier [A+]: measured from the owner's own game, not inferred.

## How (all of it reproducible, nothing installed in the game)

M2TWEOP was abandoned: its GUI (the only thing that publishes the
injection handshake) crashes under Proton, and a hand-written handshake
replacement did not take either. The winning route needs no injection
at all — read the game's memory from Linux:

- `m2tw_probe.py` (backed up in `tmp/m2tw-extraction/`) is inserted by
  a Steam launch shim INTO the pressure-vessel container, after the
  `_v2-entry-point --verb=... --` boundary. Outside that boundary the
  container's PID namespace hides the game entirely (first failure).
- It launches Proton as its own child and sets `PR_SET_CHILD_SUBREAPER`,
  so the daemonised game stays a descendant. That is what makes
  `/proc/<pid>/mem` legal under the DEFAULT `kernel.yama.ptrace_scope=1`
  — **no sudo, no sysctl, no system change** (owner refused, correctly).
  Sampler runs on a THREAD, not a fork: Yama checks the tracing process.
- The published EOP offsets DO NOT apply to this exe: the documented
  `gameDataAll` (0x02C74F90) holds the value 5, not a pointer (the
  owner's kingdoms.exe differs from kingdoms.exe.orig — patched build).
  So the probe SEARCHES: sweeps the exe's .data section, treats every
  aligned word as a candidate pointer, keeps those whose target matches
  a battleDataS field signature, then applies a **dynamic** test — the
  live battle is the struct whose seconds-elapsed float is TICKING.
  Static flags (inBattle/state) can move in a patched exe; a running
  clock cannot lie. Found it at 0x01B63C44 via pointer 0x01B70A74.
- Parse chain verified against a synthetic memory image before the run
  (scratchpad/test_probe_logic.py): signature accept/reject, army/unit
  walk, name/fatigue/level parse, and the 34-slot effect array decode.

## The extracted table (New England vs New France, 2583 samples, 46-230 s)

Every distinct `(id, modAmount, amount)` triple observed:

| id | modAmount | amount | seen on | reading |
|---|---|---|---|---|
| 3 | 8 | **+8** | ONLY the Demi Lancers (general's bodyguard) | general's own unit — matches the RTW documented `+8 bodyguard at all times` EXACTLY |
| 4 | 4 | +4 | all foot | steady/positional (+4 class) |
| 6 | 4 | +4 | all units, most samples (1681) | army-wide standing bonus (general alive) |
| 8 | 4 | +4 | all units | positional (+4 class) |
| 9 | 1/2/3 | +1/+2/+3 | all units, drifts up and down | scaled positive, small — "winning" style term |
| 18 | 4/10 | **-2 / -5** | cavalry only (Lancers, Cuirassers) | scaled negative |
| 20 | 4/8/16 | **-2 / -4 / -8** | the melee foot (Pikemen) | **casualties**: escalated -2 (t=69) -> -4 (t=96) -> -8 (t=155) as the unit bled — the MTW curve (10% -2, 50% -8) confirmed live in M2TW |

Structural findings:

- **`modAmount` = 2 x |amount| for negatives, = |amount| for positives.**
  Consistent across every sample. The engine stores a magnitude and a
  signed applied value; negatives carry twice the magnitude.
- Effects are **transient list entries**, not a fixed table: effectCount
  ran 1..5 and entries appear/disappear per tick as conditions change —
  exactly the recompute-per-tick level model 0055 implements.
- `moraleLevel` is the STATE enum, as EOP says: only `high`(2) and
  `firm`(3) occurred; the battle never produced a rout.
- `moraleThreshold` fluctuates per unit (6..20) rather than being a
  constant rout line — needs its own investigation.
- Rally counter sat at -1 (no rout), waveringTimer at 50 throughout.

## What this validates in OUR model (0055)

- The +8 general's-unit value we took from the RTW docs is CORRECT in
  M2TW — measured. Our leadership term (+2 army-wide captain floor,
  RTW formula) rests on the same documented family; id 6 (+4 on
  everyone, most common effect) is consistent with an army-wide
  leadership/steady bonus being real and always-on.
- Casualty penalties escalate -2 -> -4 -> -8 as a unit bleeds, which is
  the MTW curve we implemented (10% -2 / 50% -8 / 80% -12). Our
  `casualty_penalty()` piecewise curve is the right SHAPE, measured.
- Positive terms cluster at +4 and negatives at -2/-4/-8: the whole
  system works in small integer points on a ~16-point scale
  (`morale_max 16` from the unpacked data, devlog 0056). Our base
  morale 8/9/14 and modifier magnitudes sit correctly on that scale.

## COVERAGE GAP found in the capture (owner caught it: "units were
## already routing")

The capture holds ONLY army 0 — the player's seven units (2x Longbowmen,
2x Pikemen, 2x New World Cuirassers, 1x Demi Lancers). `playerArmyNum`
read 1, so `playerArmies[]` exposed the local army alone and the ENEMY
army was never sampled. That is why no rout appears in the data even
though units routed on screen: the routers were the enemy's, and our own
seven never broke (they were winning).

FIXED in the probe (not yet re-run): enumerate `battleDataS.sides[8]`
(0x9C, stride 0x18AC) -> `battleSide.armies[64]` (0x58, stride 0x60) ->
`battleSideArmy.stack` (+0x04) = armyStruct*, deduped, with the old
playerArmies path kept as a supplement. That is the complete army list
for BOTH sides, so the next capture gets enemy units, routs, rally
counters and contagion effect ids. Parse-chain test still green.

## Does the AMERICAS campaign change the numbers?

Mechanics: NO. The morale effect table, its ids and amounts, the state
machine and the thresholds are compiled into kingdoms.exe — a campaign
or mod cannot alter them. Americas is stock CA Kingdoms content (no
third-party stat overrides; SSHIP is a separate folder we are not
using), so what we measured is vanilla-Kingdoms engine behaviour.

What IS campaign/mod dependent: per-unit BASE morale (EDU `stat_mental`)
and unit rosters — i.e. the level each unit starts from, not the
modifiers applied to it. So the extracted effect table transfers; the
unit stat rows do not (and we already have vanilla EDU rows from the
0056 unpack).

One open caveat: kingdoms.exe and medieval2.exe are separate builds, so
strictly the table is measured for the KINGDOMS engine. Same-family
values are near-certain, but a vanilla Grand Campaign capture would be
needed to prove it byte-for-byte.

## CAPTURE 2 (owner made the enemy much stronger): the rout data

13950 samples, 870 s, 9 units (2 Longbowmen, 6 Pikemen, 1 Demi Lancers),
mauled hard (Pikemen 120 -> 1) through the FULL ladder: high, firm,
shaken, wavering, routing. Still army 0 only — the sides[] walk did not
add the enemy army (see gap above; the enemy's general died and its
units routed on screen, invisible to us). Everything below is therefore
measured on the PLAYER's units, which is sufficient for the factor
values.

### id 20 = CASUALTIES — measured, decisive

Correlating the id-20 amount against actual soldier loss
(soldiers/soldiersMax, the new columns):

| amount | casualty fraction where it appears |
|---|---|
| (absent) | 0 – 14 % |
| **-2** | from **10.8 %** |
| **-4** | from **25.8 %** |
| **-8** | from **50.8 %** |
| **-12** | from **80.8 %** |

The MTW official table (10 % -2, 50 % -8, 80 % -12) is CONFIRMED live in
M2TW, and the previously unknown intermediate step is recovered:
**25 % -> -4**. So the engine's casualty ladder is
`>=10 % -2, >=25 % -4, >=50 % -8, >=80 % -12`. Our `casualty_penalty()`
piecewise curve (0055) has the right shape and can now be made exact.

### Morale state bands (base morale + sum of effect amounts)

| state | min | median | max |
|---|---|---|---|
| high | -2 | **12** | 29 |
| firm | -11 | **3** | 13 |
| shaken | -13 | **-3** | 13 |
| wavering | -19 | **-7** | 2 |
| routing (excluding the rout lock) | -24 | **-11** | 7 |

Bands OVERLAP heavily — the documented anti-thrash hysteresis, now
measured. A unit oscillated high/firm/shaken repeatedly for 800 s
before finally breaking. Centres land near MTW's published bands
(Steady 2..14, Uncertain -5..5, Wavering -14..-5, rout <= -6).

### id 28 = the ROUT LOCK, amount **-50**

Present in 3540 of 3607 routing samples and in NO other state. When a
unit breaks, the engine injects a -50 effect that pins it deep below
every threshold — that is why routers keep running instead of instantly
recovering, and why rally needs the level to climb back a long way.
This is a mechanism we did NOT have: our 0055 model lets a routed
regiment's level drift back up almost immediately.

### The rout transition, verbatim (Pikemen unit 4)

    t=814.7 firm->shaken   soldiers= 54  eff = 20:-8 | 18:-2 | 9:+3
    t=858.3 shaken->ROUT   soldiers=  1  eff = 28:-50 | 20:-12 | 19:-4
                           waveringTimer 50 -> -102
                           rallyCounter  -1 -> -3941
                           routingCounter 0 -> 1

### Other ids (partly decoded)

| id | amounts | reading |
|---|---|---|
| 3 | +8 | general's own bodyguard unit (matches RTW docs exactly) |
| 4, 8 | +4 | positional/steady positives |
| 6 | +4 | army-wide standing bonus, most common effect |
| 9 | +1..+4 | scaled positive, drifts with the fight |
| 11 | +4 | rare, appeared only while shaken |
| 18 | -2 / -5 | frequent negative; two-step scale |
| 19 | -2..-8 | strongly tied to how many friends are routing (0 routers: only -2; 4+ routers: -4..-8) |
| 21 | -4 | rare, routing/wavering only |
| 24 | -4 | occasional |
| 25 | -1..-3 | small, mostly while routing |
| 26 | -4..-12 | **contagion candidate**: never appears with <2 routers, reaches -12 with 3 |
| 28 | **-50** | rout lock (above) |

ids 19 and 26 both scale with the number of routing friendlies — the
chain-rout driver the owner asked about, now visible as engine data:
between them they reach -8..-12 while a flank is collapsing, on top of
casualties, which is exactly how a line unzips.

## Not yet extracted (needs targeted battles, owner time — NOT to be
## demanded)

- **Fatigue rates**: this battle never tired anyone (fatigue 0..1 the
  whole time — mostly missile/idle). A marching/charging battle would
  give the accumulation curve nobody has ever published.
- **Rout/rally**: no unit broke, so no threshold crossing, no rally
  counter behaviour, no contagion effect ids.
- **Semantic id decoding**: to name ids we need controlled scenarios
  (flank one unit, kill a general, park allies at measured distance)
  and diff which ids appear. The machinery is now in place to do that
  cheaply — each battle costs one launch.

## Probe evolution (what changed between the two captures)

Capture 1 columns: t, army, unit, name, fatigue, level, state,
threshold, waveringTimer, rallyCounter, effectCount, effects.
Capture 2 added, and these are what cracked the decoding:
`soldiers`, `soldiersMax` (the casualty correlation — without them id 20
would still be a guess), `routingCounter`, `isRallied`, `forceRout`,
`inShock`, `moraleBonus`, `minMorale`, `maxMorale`, `minRoutDelay`, plus
the sides[] army walk (which did not yield the enemy — still open).

Method note for the id-20 proof: bucket every sample by its id-20
amount, compute `1 - soldiers/soldiersMax` for each, and read off the
minimum casualty fraction per bucket. The buckets do not overlap at
their lower edges (10.8 / 25.8 / 50.8 / 80.8 %), which is what makes the
ladder unambiguous rather than correlational.

Note on `moraleThreshold` (+0x08): NOT the numeric morale. Tested
`base + sum(effects) == threshold` across 1845 samples: 22 matches,
1823 mismatches. It sat at 8 for most units while the effect sum ranged
widely, and drifted 6..20 for others. The engine appears not to store
the summed level at all (state is derived per tick); our reconstruction
uses `EDU base morale + sum(effect amounts)`, which is what produced the
clean state bands above.

## FATIGUE: the first real rate anchor in the whole lineage

Fatigue accumulation rates were never published for MTW, RTW, M2TW or
their guides — devlog 0056 records that even the MTW wiki marks them
"TBD". The captures give the first measurement, by reading the engine's
own per-unit fatigue counter (unit+0x2D4, a 0..15 scale quantised into
the six display states):

| capture | battle length | fatigue reached |
|---|---|---|
| battle 1 | 184 s | 0 -> **1** |
| battle 2 | 855 s of hard fighting | 0 -> **2..5** |

So ~14 minutes of heavy melee gets a unit to roughly warmed up /
winded. M2TW tires men SLOWLY, and nothing approached exhausted in a
quarter-hour battle. Our rates reached exhausted in ~2.5 min of melee —
about 4x too fast, which the owner's feel pass independently flagged.
Retuned in `ea94f11` to fight 0.14 / charge 0.20 / move 0.08 / recover
0.12 per second: Tired at ~6 min of continuous melee, Exhausted at
~10 min, Very Tired -> Fresh in ~10 min standing. The activity ordering
(fight > charge > move, idle recovers) is unchanged — that part was
always the evidenced bit.

## Artifacts

All extraction data lives in the REPO (gitignored `tmp/`, same
convention as the handoff files), NOT in a volatile /tmp:

- `tmp/m2tw-extraction/probe-battle1-20260726-0330.csv` (2583 rows,
  184 s) and `.log` — the first successful capture, effect table
- `tmp/m2tw-extraction/probe-battle2-routs-0418.csv` (13950 rows,
  855 s) and `.log` — the rout capture: casualty ladder, state bands,
  rout lock, contagion scaling, fatigue anchor
- `tmp/m2tw-extraction/m2tw_probe.py` (the working probe)
- Live copies: game folder `probe.csv` / `probe.log`; shim at
  `/data2/SteamLibraryFlatpak/eop_launch.sh`
- Cleanup when done: clear the Steam launch options, delete the shim +
  probe, and remove the EOP files copied into `mods/americas` (stock CA
  data otherwise untouched; `Uninstall_EOP.bat` ships with them).
