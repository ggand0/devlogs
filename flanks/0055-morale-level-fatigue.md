# 0055 — Morale reworked to the M2TW level model + fatigue (2026-07-25)

FULL EVIDENCE RECORD: devlogs/0056-m2tw-morale-fatigue-evidence.md
(all sources, verbatim tables, tier per implemented factor, install
unpack method, extraction paths). This entry is the implementation
log; the summaries below are superseded by 0056 where they overlap.

Branch `feat/morale-fatigue`. Owner direction: the old morale was a
draining tank ("War of the Dots") — factors only ever subtracted, so a
unit flanked once was permanently poorer. In Total War, morale is a
LEVEL decided by current factors; what we computed as the "diff" IS the
morale. Research first, then rework; fatigue added at the same time
(that accumulator is the legitimate home of the WotD-style mechanic).
Run/walk toggle deferred (needs control UI).

## Evidence record (research agent, sources verified)

Tiering: [A] engine-primary (Feral Rome Remastered official formula
docs; M2TWEOP disassembly structs from medieval2.exe), [B] adjacent
titles with published numbers (MTW1 BradyGames official guide; NTW db
table), [C] community observation, [F] unknown/folklore.

- **The model [A]**: M2TWEOP `unit.h` shows per-unit `moraleLevel`,
  `moraleThreshold`, and `moraleEffect effects[34]` ({id, modAmount,
  amount}) — a base stat plus a summed list of concurrent situational
  effects, recomputed continuously. State enum: berserk / impetuous /
  high / firm / shaken / wavering / routing. Timers exist for transient
  shocks (`surpriseCounter`, `chargeTimer`, `waveringTimer`) and rally
  (`routingRallyCounter`, `minRoutDelay`, `isRallied`). RTW's mouseover
  literally listed active factors by name; M2TW hid the text, kept the
  mechanics.
- **Factor values**: M2TW's own numbers are compiled into the exe and
  were NEVER extracted [F]. The MTW1 official-guide table [B] is the
  design template CA carried forward and the only numeric source in
  the family: casualties 10% -2 / 50% -8 / 80% -12; losing combat up
  to -8, winning up to +6; one flank threatened -2, both or rear -6;
  routing friends up to -12, routing enemies up to +8; flanks secure
  +4; no enemy nearby +4; uphill +2; rout at -6 or less, return to
  wavering when the recomputed level recovers; overlapping bands as
  anti-thrash. Discipline weights shock losses [A semantics]. (The
  "Broken -15..-2 …" numbers circulating for M2TW are actually the NTW
  Warscape table — rejected as evidence.)
- **Fatigue [A structure, C rates]**: per-soldier accumulator (4-bit
  quantized), states fresh / warmed up / winded / tired / very tired /
  exhausted. Fighting accumulates fastest, then charging/running;
  walking little (M2TW), idle recovers slowly (minutes scale).
  `hardy`/`very_hardy` = flat rate modifiers; `stat_heat` = armor/heat
  extra rate [A]. Effects: melee and speed penalties, morale penalty,
  exhausted cannot run or charge; only MTW1 published numbers [B]:
  attack -2/-3/-4/-6 by state, morale -3/-6/-8, no run/charge at worst.
- Extras for later [A]: general aura +8/+4 (radius 6 + 7xcommand +
  4xinfluence), fear auras -4..-8 @100m, RTW formation bonuses
  (phalanx +2), ambush surprise -7 for 60 s @80m.

## New model (morale.rs, rewritten)

Every tick, for EVERY regiment (steady or routing):

```
effective = base(kind) + casualties + exchange + flanked + outnumbered
          + contagion + rout_enemies + support + no_enemy + disorder
          + fatigue + wall
```

- base: heavy 14 (elite band; frontal attrition alone can no longer
  reach the rout line), spear 9, light 8 — vanilla M2TW stat scale.
- casualties: MTW1 curve through (10%,-2) (50%,-8) (80%,-12),
  extrapolated. A PLATEAU, not a pump: stop the killing, the penalty
  stops growing. Replaces MORALE_CASUALTY x recent_deaths drain AND the
  depletion multiplier.
- exchange: NEW — winning/losing melee. recent_kills tallied in the
  damage apply pass next to recent_deaths; both EMA-smoothed (~3 s),
  net ratio x intensity (saturates at combined 1.5%/s of strength)
  scaled to -8 losing / +6 winning.
- flanked: the proven 8-probe dominance ring, remapped from a 6/s drain
  to a -6 x flanked01 level term, discipline-weighted.
- outnumbered >3:1: -2, discipline-weighted (owner: soldiers can't
  count a battlefield — stays minor).
- contagion: -3 per routing friendly within 60 m, cap -12,
  discipline-weighted. rout_enemies: +2 each, cap +8 (NEW).
- support: +4 x friends/3 x (1 - flanked01) ("flanks secure").
  no_enemy: +4 when the ring is quiet, unengaged, no enemy near — this
  is also the rally pull. wall: +2 (RTW phalanx analog; replaces the
  0.85 multiplier).
- disorder: kept from the old model (NOT M2TW-evidenced, flagged),
  demoted to 0..-3 while engaged.
- fatigue: 0/0/0/-3/-6/-8 by state (MTW1).

Thresholds: wavering at <= 0 (new `GroupData::wavering` display flag —
trembling banner, WAVERING in the inspect panel), BREAK at <= -6 held
for 1 s (waveringTimer analog; a one-tick spike can't break a
regiment), rally when a routed regiment's recomputed level climbs back
above -5 after the 8 s minRoutDelay — the dice roll is GONE; getting
clear of pursuit is what rallies you (casualty penalty + no_enemy
means badly mauled lights mostly won't, heavies often will). Shatter
below 15% strength unchanged. `morale_resist` became `discipline`
(same values, now applied only to shock terms per the documented
semantic). Display bars normalize [rout line .. base] via
`morale::morale01`.

## Fatigue (fatigue.rs, new)

Per-REGIMENT f32 0..100 (M2TW is per-soldier 4-bit quantized to the
unit; one f32 per regiment is the 200k budget). Rates are free knobs
(M2TW's were never published — ORDERING is the evidenced part):
fighting 0.55/s, charging 0.8/s, moving/fleeing 0.3/s (everything runs
today; the walk/run toggle later gets a walk trickle), idle recovers
0.35/s. Kind mult (stat_heat analog): heavy 1.3, spear 1.1, light 1.0.
Bands at 15/30/50/70/85. Continuous melee: tired ~2 min, exhausted
~3.5 min; very tired -> fresh ~3.5 min standing.

Effects: attack points -2/-3/-4/-6 from winded (damage apply pass, per
the MTW1 attack-only table); speed x0.95/0.88/0.78 from tired (desired-
velocity mult in integrate — fleeing units tire, pursuit catches them);
exhausted: no charge bonus, no charge sprint; morale as above. Inspect
panel shows the state name.

## Consumers touched

movement.rs (fat_atk/fat_nocharge in damage apply, recent_kills tally,
fat_speed + sprint gate in integrate), banners.rs (wavering flag +
morale01 fill), unit_cards.rs (morale01 color), render_units.rs
(stance by wavering), overlay.rs (full signed factor breakdown +
fatigue state), main.rs (FatiguePlugin).

## Verified (Xephyr/llvmpipe scripted runs, second commit)

FOUND AND FIXED in testing: the 22 m flank ring sits INSIDE a 1000-man
block's ~66 m footprint, so the dominance test suppressed every probe
and encirclement read as ~nothing — the old drain model masked this by
time-integrating the tiny reading; the level model exposed it. Fix:
`GroupData::radius` (1.5x RMS distance to centroid, smoothed ~2 s,
update_groups) and the ring probes at `radius + 10 m`. The rout-test
log now prints the full factor breakdown (also enabled under
FL_TEST_FRONT as a line-fight probe).

- FL_TEST_ROUT (1000 lights vs 3x1000 converging): calm morale 12 =
  base 8 + no-enemy 4; casualty curve tracks the MTW1 anchors; flank
  reads 40-60% for the 3-sided envelopment (open rear — correct
  geometry); breaks at -7.9 after the 1 s hold, 63% losses (old model:
  ~30% — the level model holds longer; owner feel-pass lever if too
  stubborn: base_morale, or add the MTW1 army-outnumbered factor,
  2:1 -4 .. 10:1 -12, deliberately NOT added yet since it half-
  contradicts the owner's "soldiers can't count" ruling). On break:
  orders cleared, flees toward its own edge. Post-DEFEAT freeze in the
  log is the game shell stopping the sim, not a morale bug.
- FL_TEST_FRONT (FL_UNITS=10000, 10v10): line fights read flanked 0%
  through the grind (ring fix holds in both geometries), 20-40% only
  when neighbors break and gaps open. Breaks at 60-83% losses,
  cascading through contagion as the line collapses; heavies hold
  longest (reg 0, heavy: never broke at 63% losses, morale -4).
  RALLIES WORK and are legible: reg 4 rallied at -5.0, fought, re-broke
  at -6.4 (hysteresis band + easy re-rout, the M2TW trait); regs
  2/3/10 rallied at -1.5..-1.8 once clear. Shatter floor works; fled
  despawn counter climbs (7 -> 206). Fatigue: heavy reg 0 went 0 -> 84
  (very tired) over the battle, -6 morale late — the intended arc.
  27 morale events over ~4 min, no panic, no NaN.

## Expectations for the feel pass (owner play-test)

- Frontal grinds last longer before first break (break now needs level
  <= -6, typically casualties + losing exchange + disorder together);
  the collapse still cascades through contagion + flanks. If pacing
  needs blood, the levers are base_morale and EXCHANGE_* first.
- Units RECOVER when a threat passes: rout contagion ends when the
  routers leave the radius; a survived flank attempt no longer leaves
  a permanent scar (casualties do).
- Rallies are deterministic and legible: outrun pursuit -> rally.
  Watch for rally/re-rout flapping at the -5/-6 hysteresis; widen
  RALLY_AT if it thrashes.
- Late battle everyone is tired: slower, weaker, waverier. Kind mult
  makes heavies gas first — intended (armor), tunable via fatigue_rate.

## Evidence round 2: primary sources re-verified FIRSTHAND (2026-07-25)

Owner challenged the evidence chain; re-dug the primary sources
directly (not agent summaries). Confirmed verbatim: M2TWEOP unit.h
moraleStruct/moraleEffect[34] (fetched, in scratchpad), moraleStatus
enum berserk0..routing6 (luaEnums.cpp); NEGATIVES personally
established: no morale-effect-id enum anywhere in the EOP repo, no
M2TW factor values in any cheat table (stamina freezes only), RR
never exposed morale/fatigue tables as data (repo searched), and the
MTW wiki itself marks fatigue depletion RATES as "TBD" — the rates
were never documented for ANY game in the lineage. Our rates are
knobs forever unless measured; RR 2.0.4+ displays LIVE effective
morale in its UI (changelog line 489) = a real black-box measurement
path for RTW-lineage values if we ever buy/install RR.

MTW_Morale wiki fetched verbatim (archive): our impl DIVERGES from
the template on contagion: documented = ~-6 per FULL routing unit
saturating at TWO total units (-12), routing units WEIGHTED BY CLASS
vs observer discipline (elites/disciplined count lesser routers as
HALF); routing enemies +4/unit cap 2. Ours was -3/+2 per unit cap
4 with a flat observer multiplier. Also verbatim: "This is the main
factor in how chain routings occur... hammer the end of a flank,
start a rout, and just roll up the enemy line" — CHAIN ROUTS ARE THE
DOCUMENTED CORE MECHANIC of the family, not a tuning artifact; the
documented counterweights are router-class discipline weighting,
the general, and rally. Confirmed extras: charged-in-flank -4 (cav
-6) timed shock, missile fire -2 (+4 fear weapons), uphill +2,
hysteresis semantics exactly as we built (-6 rout, recover above -5).
TODO from this: fix contagion/rout-enemies to the documented curve +
class weighting (after the seed sweep finishes).

## Vanilla install unpacked + leadership term (2026-07-26, 80094c8)

Owner pointed at his Windows M2TW Steam install (/data2/...Flatpak/
.../Medieval II Total War — NOT the Feral DE from devlog 0031; this
one ships medieval2.exe + the OFFICIAL unpacker). Ran
tools/unpacker/unpacker.exe under Proton 8.0 wine (yes | for its 3
prompts, --destination pointed at the session scratchpad so the live
install stays untouched): all 5 packs extracted, 211 text/xml data
files at scratchpad/m2tw_vanilla/. The sship mod install also carries
a full unpacked (modded) data tree for schema reference.

ENGINE-DATA morale values recovered (first real M2TW numbers, tier A,
vanilla descr_area_effects.xml): ae_holy_inspiration +5 morale radius
50 m; ae_cow_carcass -5, radius 10, duration 300, morale_max 16;
(sship adds greek-fire fear -10 radius 5 for 30 s). morale_max 16 =
the engine's morale scale ceiling — our base 8/9/14 sits correctly
under it. These two effects are exactly the named fields in EOP's
moraleStruct (aeCowCarcass / ae_holy_inspiration): data and
disassembly cross-confirm. descr_climates.txt: climate heat 1-4,
comment "zero would mean no armour effects to fatigue at all" —
confirms armour x climate drives fatigue rate (our per-kind
fatigue_rate is the right shape). export_descr_character_traits:
TroopMorale is a real M2TW trait attribute (160 uses, +-1..3).
battle_config.xml holds NO morale/fatigue values (AI/skirmish/ladder
config; sship's is ReallyBadAI's with melee-hit-rate 2.0).

LEADERSHIP TERM implemented (owner: "in m2tw you'd get a general
attached even without a general unit"): every army is led — captain
floor. RTW formula (Feral docs, tier A for RTW): army-wide =
2 + command + influence/2 + TroopMorale, so a bare captain = +2 for
the whole army, aura radius at 0 command ~6 m (skipped until real
generals). Implemented: first heavy regiment per team (else first
alive) becomes the command regiment (GroupData::leader, assigned
once in the morale tick); +2 army-wide while it stands
(LEADER_ALIVE); on break: -8 for 10 s (LEADER_SHOCK, MTW "for a few
seconds") then -2 permanent (LEADER_LOST_PERM); rally restores.
"commander" line in the inspect panel. Verified in FL_TEST_FRONT:
calm heavy reads 20 = 14 + 4 calm + 2 commander; "team 0 LOSES ITS
COMMANDER" fires when the leader heavy breaks. Killing the enemy
command regiment is now a real objective — a 4-point army-wide swing
plus a 10 s window where their whole line is 10 points weaker.

## Cascade sweep + contagion correction (2026-07-25, 7c25427)

FL_SEED added (spawn-jitter offset; sim otherwise deterministic).
6-seed sweep, symmetric FL_TEST_FRONT FL_UNITS=10000, break-side
sequences in event order (B=blue O=orange):

    seed 1  BBBBBBBBBOOOBOOOOOOO   breaks 10/10  rallies 4/1
    seed 2  BBBOBOBBBOOOOBBOBBOO   breaks 11/9   rallies 3/2
    seed 3  BBOBBBBOOOBBOOOOOOB    breaks 9/10   rallies 2/1
    seed 4  OBBOBBBOBBOBOOOOBOBBOOO breaks 11/12 rallies 5/3
    seed 5  BBBBOBOOBOBBBOBBOBBOBOOBBB breaks 17/9 rallies 9/2
    seed 6  BBBBBOBOBBBOBOBBBOOOBBBOBBB breaks 19/8 rallies 11/1

Findings: (1) one-way opening cascades DO happen (seed 1: nine
consecutive blue breaks — the owner's "flank collapse") but mutual
interleaved grinds are equally common (seeds 2, 4); no battle within
the window was decided by the first rout alone, all ran to deep
mutual attrition. (2) Blue breaks FIRST in 5/6 and breaks more
overall — a STRUCTURAL bias: the front-test script pushes a blue
regiment through the line as a salient; "who wins feels random" in a
true mirror is really "structure decides, noise picks the details".
(3) Rallied units re-break routinely (rally counts track break
counts), which is documented behavior.

Contagion corrected to the primary-source curve (see round 2 above):
-6 per weighted routing unit sat at 2 (-12), routers class-weighted
vs observer discipline (heavies half-count routing lights); routing
enemies +4 weighted, cap 2. A/B on seed 1: opening run of consecutive
blue breaks 9 -> 6, orange answers sooner; blue-first structure
unchanged as expected.

## Owner feedback round 1 (2026-07-25)

Fatigue feels fine -> fatigue bar added to unit cards (0f9a005): thin
stamina strip under the morale strip, drains left-anchored, tinted by
band (teal -> amber -> orange -> red-brown). Morale "alright", but the
symmetric scenario feels RNG-decided and one rout chain-collapses the
flank. Assessment: chain routs ARE authentic M2TW; the missing
stabilizer is the GENERAL's aura (+2 + command army-wide, +4/+10
nearby, the engine's anti-cascade anchor) plus army variety from the
deployment phase — in a mirror match with no general, first break is
noise and the contagion + routing-enemy feedback loop does the rest.
No flat defender bonus exists in M2TW field battles; defender edges
are emergent (uphill morale/combat, braced walls, attacker arrives
fatigued — our fatigue already provides that one organically).

## Future (evidenced, not yet built)

Run/walk toggle (+ control button UI) with walk fatigue trickle;
generals + rally aura (+8/+4, radius formula above); timed shock
events (charged-in-flank/rear -4, surprise); uphill +2 (terrain slope
is already queryable); fear auras when monsters/cavalry arrive.

## Tier A + B landed from the measurements (2026-07-26, 5a5e32a)

Applied per docs/plans/morale-from-measurements.md (owner approved both
tiers). A: measured casualty step ladder (>=10% -2, >=25% -4, >=50% -8,
>=80% -12), rout lock -50 decaying over 25 s, commander's own regiment
+8, contagion unchanged (already matched the measured ceiling). B: base
morale rebased to the vanilla EDU scale (heavy 11, light/spear 5),
bands shaken -3 / wavering -7 / rout -11 / rally >-7, shatter floor
15% -> 3%.

MEASURED OUTCOME, FL_TEST_FRONT seed 1, 10v10 at 10k men:
breaks at **81-98% losses** (before: 60-83%), 9 breaks, 8 shatters,
**0 rallies**. Units now fight nearly to annihilation.

This is FAITHFUL to the numbers — in the captured M2TW battle units
routed at 80-99% losses too — but it is a large pacing change and the
rout phase is now brief (they break with so few men left that pursuit
finishes them). Owner feel pass needed.

Diagnosis of the amplifier, for the next round: at break the M2TW
Pikemen had NO positive effects active (list was 28:-50, 20:-12,
19:-4). Our regiments in a frontal line carry support +4 and commander
+2 continuously, so a light sits at 5+4+2 = 11 before negatives and
needs -22 to break. Candidate levers, in order:
1. Gate `support` (secure flanks) on not-losing — it currently applies
   whenever 3 friends are within 60 m, even while the unit is being
   destroyed. The engine's +4s clearly drop off as things go bad.
2. Reduce NEIGHBOR_R (60 m) so support needs genuinely adjacent units.
3. Strengthen the exchange term (currently max -8) — losing badly
   should hurt more than it does.
NOT recommended: touching the measured casualty/threshold numbers.

## Post-review cleanup + two caveats (2026-07-26, c0b047f)

Four-angle review of the branch (reuse / simplification / efficiency /
altitude). Applied: duplicate per-tick fat_nocharge vector removed,
routing-ally and routing-enemy scans fused into one pass, squared
distances in the per-regiment neighbour scans (they run every tick for
every regiment), Local buffer reuse for the two centroid snapshots,
fatigue's four parallel band matches collapsed into one const table,
two dead MoraleFactors fields deleted, cached shaken/wavering flags
replaced by a derived band() helper, FL_SEED read as u32 via env_or.
Clippy clean at --all-targets. PR draft in tmp/drafts/.

CAVEAT 1 — the 6-seed cascade sweep above is WEAKER than presented.
FL_SEED is folded in inside push_unit, but the positional jitter is
computed at the CALL SITES from the un-offset seed (regiments.rs
spawn_regiment, units.rs scenario spawns). So formation geometry was
IDENTICAL across all six seeds; only speed jitter, colour tone and
swing phase varied. The break-sequence clustering still stands as a
observation of one geometry, not of six independent battles. FIXED in 1af8dd2: the offset now applies where a seed is
CONSTRUCTED (spawn_regiment and the scenario spawns) instead of at the
bottom of push_unit, so geometry varies too. Verified across three
seeds: 800/814/804 alive at the same sample point. The sweep table
above predates the fix and should be re-run if its conclusions matter.

CAVEAT 2 — fatigue verification (the run behind ea94f11 / 2a26a0e):
regiment 0 in FL_TEST_FRONT reached fatigue 8 at t=37 s, 27 at 109 s,
55 at 218 s, 74 at 290 s, i.e. ~0.26/s for a regiment both fighting
and manoeuvring (pure-melee rate is 0.20/s). Winded ~2.3 min, tired
~3.5 min, very tired ~4.8 min. Battle behaviour unchanged: 8 breaks,
morale falling 25 -> -1 as the regiment was ground down.

Skipped as too large for a cleanup pass, both worth doing: move
leadership assignment out of the morale tick into the spawn path, and
thread FL_SEED through BattleConfig (see caveat 1).
