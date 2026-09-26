# 0034 — Phase B: charge impact — knockback, stagger, reflection (2026-07-14)

Owner signed off on the phase-A + facing state (sees ~100-man
depletion gaps front vs rear in play) and green-lit phase B
(docs/plans/002-formation-combat-v2.md). Commit d517823. Stagger visual
stays physics-only for now (plan open question 3 unanswered — the
knockback displacement itself is visible).

## Mechanics

- **Knockback**: a charge-flagged hit displaces the victim along the
  blow by `CHARGE_KNOCKBACK(0.9) × m_a/(m_a+m_v)` (heavy into light
  ~0.55 m), applied straight to position in the serial damage apply;
  next tick's separation resolves the pile — same pipe as all overlap,
  no new solver.
- **Stagger**: the same hit cancels the victim's swing (state Recover
  + SWING_STAGGERED, spare bit 5), locks steering AND yaw for
  STAGGER_TICKS(30): the charge's second blow finds the same back the
  first one hit. On expiry the man gets SWING_STAGGER_IMMUNE (bit 6),
  one free pass against the next stagger, consumed by it or wiped by
  his own next swing — chain-charging cannot stunlock a man who never
  gets to act.
- **Bracing is directional**: a wall hit FRONTALLY takes 0.25×
  knockback and never staggers; a braced spearwall additionally
  REFLECTS the charge bonus (attacker takes `dmg × (1.13^cb − 1)`, the
  damage his momentum would have added, in the same apply pass).
  Charged in the flank/rear, walls are bodies like any others — and
  the phase-A spearwall charge-nullification is now frontal-only too.
- **Masses to the EDU scale**: knight 2.0→1.5, spear 1.2→1.0,
  man-at-arms 1.0→0.9 (militia 0.8 and cavalry ~3.5 slot in later).
  Drives both separation shove and knockback ratios.

## FL_TEST_CHARGE (new acceptance)

Two lanes of 500 blue spears on HOLD facing the chargers; west lane in
SPEARWALL, east lane normal order; 500 orange heavies attack each.
Measured:

- WALL lane: dz max −4.7 m, never breaks in the window, kills chargers
  1.6× faster (221 heavy dead vs 137) — reflection pays.
- OPEN lane: bowled back −8.6 m sustained (visible dent), breaks at
  ~58% casualties t≈41, flees (dz −31/−59/−88 = the rout).
- No stunlock: open-lane spears still killed 137 heavies; staggered
  headcount stays single-digit at any sampled instant.

Bands (from these measurements): wall dz ≤ 5 m, open dz ≥ 8 m before
break, wall-lane charger deaths ≥ 1.4× open lane, open-lane charger
deaths > 100.

## Regression sweep — everything in band

SURROUND line 118 (band 115–127), pocket 0 by t=40. ROUT breaks at
495 alive (~50% band), fled 475. DIR: rear/front per-hit 49.3/18.4,
rear bucket dominant. FORM 6/6 OK. FRONT 200k: engaged step 7.71 ms
(pre-B 8.05, budget +0.5 — actually cheaper), drift 2.33 m, nn_min
0.67–0.69 (mass rescale caused no overlap regression).

## Round 2: the spear becomes physical, the stagger becomes visible

Owner rulings after reviewing phase B (now saved as project memory
cascade-design-principle-simulate: NO stance flags/tallies — simulate
the physical event, let effects emerge):

- **Spearwall stance rules DELETED (a4687c7).** The nullify/reflect
  conditionals were "a stupid circle around a unit — BS". Replaced by
  the spear as a physical line: each braced spearwall soldier's point
  covers a segment along HIS OWN facing (0.9 m — inside it the spear
  is useless — to 2.4 m reach, 0.7 m wide). A body crossing it at
  charge speed is IMPALED: damage event from the spearman through the
  ordinary directional pipeline, the charger's own charge_bonus as
  attack points against him (momentum is the force), stagger as the
  stop (no knockback — the momentum went into the point). Detection
  rides the existing fused scan via a tick-start yaw snapshot
  (neighbor yaw is unreadable in the parallel loop — chunked &mut);
  a global per-tick gate (does the enemy field a spearwall at all?)
  makes wall-less battles run the IDENTICAL scan — necessary because
  march speed exceeds the charge threshold, so without it every mover
  in a 200k battle paid the wider scan. Emergent: gap-runners (~1/3
  of frontage + every hole from a dead/turned/staggered wielder) land
  their FULL charge bonus on the wielder; slow walk-ups are never
  impaled; flank/rear ignores the wall; a bleeding wall stops
  stopping charges. FL_TEST_CHARGE reproduces the stance version's
  numbers organically (6 chargers stopped dead at the fence at
  contact, 21 arrival-wave deaths vs 12 open lane, 1.58x charger
  deaths overall, wall holds −4.9 m).
- **Stagger pose (f044b74)** — owner reversed "later": invisible
  stagger made the mechanic unverifiable by eye ("minecraft stagger
  BS"). anim2.w now carries remaining stun (1→0); the shader pitches
  the whole body backward from the feet (−0.42 rad × stagger², 22 Hz
  reel, −0.10 knee sink), same rotation form as the death topple.
  Shader compiled clean (zero "failed to process shader").
- **PERF GATE STILL PENDING**: the 200k FRONT re-measure ran under
  load average 14 (the unchanged BASELINE binary swung 8.8→7.2 ms
  between consecutive runs — machine contention, not signal). The
  hazard is provably inert without enemy spearwalls (global gate), so
  no structural regression is expected; re-measure on a quiet box
  before the next milestone commit. Also noted by owner: 200k camera-
  move fps dips (~40) exist and predate all of this — RENDER-side
  (culling/draw), separate backlog item.

## Round 3: per-hit stagger on the combat factor (d994bb2)

Owner (after approving the pose via FL_DEBUG_STAGGER): every-hit
stagger IS the M2TW behavior, wanted weapon-dependent probability.
Research (TWC stagger thread, Proudnerd guide, Steam mace-stats
thread): M2TW staggers on every hit that connects, is NOT BLOCKED,
and does not kill; a staggering man cannot block (the outnumbered
snowball); vanilla maces have NO hidden knockdown stat — blunt
staggers armored men more only because AP halves armour. So the gate
is "did the defence stop it", which we already compute continuously:
p = clamp(0.65 + 0.05 × factor, 0.05, 1) — even match 0.65, rear
~1.0, elite shieldwall frontal ~0.3, AP bonus emergent. Ordinary hits
stumble 0.5 s (pose auto-reads lighter: render scales by remaining
ticks/30); charge impacts + impalements stay certain at the full 1 s.
Snowball: staggered men lose their defence SKILL (can't parry; shield
passive + armour still count), bounded by the anti-stunlock pass.
FL_DEBUG_STAGGER retained (forces p=1).

Re-measured bands: DIR rear/front per-hit 49.6/22.2 = 2.23x (gate
holds; front avg rose from 18.5 by design — staggered men parry
nothing), ROUT break 500 alive (unchanged), SURROUND line control
115 -> 86 (RE-BASELINED: pressed lines bleed harder under the
snowball), CHARGE wall dz −3.9. BALANCE FLAG for owner play-test:
braced spearwalls now beat equal-count heavies frontally (282 v 222
at t=50, previously losing slowly) — frontal-braced men never stumble
while their stabs stagger the knights. Lever if too strong:
reduced-chance braced stumbles instead of none.

## Round 4: stumble visibility fix + system semantics (6b3300f)

Owner rarely SAW staggers in FL_TEST_CHARGE. Two causes: (a) that
scenario is the stingiest showcase — braced wall spears are frontally
stagger-immune and spear-vs-knight stabs only roll 20%; (b) a real
bug: the pose progress was normalized by the 30-tick impact stun, so
a 15-tick stumble started at half progress and the squared falloff
played it at quarter amplitude (~6°, invisible). Fixed: normalize by
the stumble length — a stumble is the FULL rock for 0.5 s, an impact
holds it ~1 s. Duration is the hierarchy, not amplitude.

Stagger semantics as shipped (owner Q&A, for the record):
- No stagger on a landed hit = the defence handled it (the roll lost
  to skill/shield/armour); the white hit-flash still shows.
- Hits landing DURING a stagger re-roll at raised odds (victim can't
  parry) and REFRESH the timer — several enemies cycling blows keep a
  man down (the M2TW outnumbered snowball). The floor: on recovery he
  carries ONE immunity pass, so he always gets a window to answer;
  no infinite stunlock.
- Stumble odds by matchup: rear ~95-100, knight-vs-open-spear 65,
  light-vs-light 55, knight-vs-knight 50 (symmetric churn),
  spear-stabs-knight 20, anything-vs-braced-wall-frontal 0.
- Factor = attack MINUS surviving directional defence (difference,
  not ratio) — the canon M2TW combat-factor form; the stagger curve
  constants (P0 0.65, 0.05/factor) are OUR calibration of M2TW's
  undocumented block-roll internals, anchored to the owner's chosen
  default. Tuning levers: STAGGER_P0 / STAGGER_P_PER_FACTOR
  (movement.rs), pose constants (unit_instancing.wgsl).

## TODO remaining (stagger presentation polish, owner's call)

The pose landed (f044b74, round 2 above), superseding the earlier
"owner does it later" plan. Still open if wanted: spread the CHARGE
knockback displacement (a fraction on the hit tick, the rest drifting
over the stagger's first ~0.3 s) so the shove reads as a shove rather
than a one-tick pop — the pose now masks most of it, judge in play.

## Round 5: perf verdict + the lag-spike hunt (f5adab9)

- Interleaved A/B (2x each, same load): pre-phase-A 7.1/7.7 ms vs
  current 8.7/9.1 ms engaged mean = +1.45 ms for phases A+B combined.
  Owner ruling: mean accumulation from real mechanics is a non-issue —
  the +0.5/phase budget letter is waived; what he cares about is LAG
  SPIKES. Perf gate CLOSED on those terms.
- Spike hunt (new [spike] attribution log, fires > 14 ms with
  components): 272 spike ticks/130 s at 200k under load ~10/24 cores.
  Verdict: BOTH parallel phases (grid rebuild AND integrate) balloon
  together, spikes occur pre-contact with ZERO damage events, and none
  carry the audit tag — pure core contention between the sim's
  full-width task pool and everything else on the box. Identical on
  the pre-phase-A baseline: predates all mechanics, matches the
  owner's "probably been like this for a while".
- FL_THREADS knob added; measured NEGATIVE at moderate load (14 of 24
  threads at load 10: spikes 272→3029, worst 23→50 ms — throughput
  bound, narrowing the pool hurts more than contention). Kept for
  extreme-load experiments, zero cost unset. Also: yaw snapshot copy
  now gated behind spearwall presence.
- Camera-move fps dips remain RENDER-side (culling/draw), separate
  backlog (GPU culling deferred to ≥300k per 0020-era decision).
- **CORRECTION (owner re-test, near-idle box: browsers/vscode only —
  spikes identical):** the "external workload contention" framing was
  wrong; the measured load was largely MY OWN test runs. The real
  contention is intra-process: Bevy pipelines the render app on
  separate threads CONCURRENT with the sim tick, so at 200k the
  per-frame render prep (instance sync + per-instance CPU culling +
  GPU driver threads) steals cores from the sim's parallel scopes on
  any box. Camera motion grows render prep → frame overruns 33 ms →
  fixed timestep runs multiple catch-up sim ticks in one frame → the
  felt hitch (classic fixed-timestep spiral). OWNER DIRECTIVE: fix on
  a DEDICATED perf branch, not feat/formations (frozen at f5adab9).
  Branch scope: (1) fixed-timestep catch-up clamp (hitch → momentary
  slow-mo; biggest feel win, one focused change), (2) render prep
  cost / GPU culling, (3) verify against PLAYED sessions via the
  [spike] log, never fixed-camera runs. Nothing shipped here fixed
  the spikes — instrumentation + a negative-result knob only.

## State

feat/formations now 2eddde3..d517823 (phase A, facing overhaul 0033,
phase B), all unpushed. TEMP 5v4 sandbox defaults still uncommitted in
units.rs/regiments.rs. Remaining: phase C (rank discipline, gated
FL_RANKFIGHT, re-measure 0025/0026 bands), morale rework (owner does
this himself — do NOT add morale mechanisms), spearwall lateral
tightening 1.05→0.85, stagger pose (open question), levy kind
deferred, archers milestone after.
