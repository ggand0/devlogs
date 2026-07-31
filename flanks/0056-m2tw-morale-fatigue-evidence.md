# 0056 — M2TW morale/fatigue evidence record (2026-07-25..26)

Companion to devlog 0055 (implementation). This is the full research
record: two online research passes (agent + hands-on primary-source
verification) and the dig through the owner's actual M2TW install.
Every claim is tiered:

- [A] engine-primary: M2TWEOP disassembly of medieval2.exe; Feral
  Rome Remastered official docs (Feral reimplemented RTW with source
  access and documented hardcoded formulas); actual game data files
  from the owner's vanilla install.
- [B] adjacent titles with published numbers: MTW1 (BradyGames
  official strategy guide + CA developer posts, via the archived
  totalwar.org wiki); NTW's extracted db table (Warscape).
- [C] community observation/testing, qualitative.
- [F] unknown/folklore, explicitly flagged.

## 1. The M2TW morale architecture [A]

M2TWEOP (`M2TWEOP-library/types/unit.h`, fetched raw, copy in the
session scratchpad) reverse-engineered the per-unit morale state:

    struct moraleEffect { int id; int modAmount; int amount; };
    struct moraleEffects { moraleEffect effects[34]; };
    struct moraleStruct {
      unit *thisUnit;
      int moraleLevel;          // 7-state machine, see enum below
      int moraleThreshold;
      int moraleBerserkCounter;
      int surpriseCounter;      // timed shock class exists
      int routingFinishCounter;
      int routingRallyCounter;  // rally is a timed, checked process
      int waveringTimer;
      int chargeTimer;
      moraleEffects effects;    // THE FACTOR LIST: concurrent summed
      moraleEffects *effectsPtr;
      int effectCount;
      int moraleBonus; int bloodBonus;
      int8 inMoraleShock; isRallied; processRally; forceRout;
      int8 lockedMorale;        // EDU lock_morale
      int8 enemyCurse/curse/enemyChant/chantEffectCounter; // RTW auras kept
      int8 routingCounter;
      int aeCowCarcass;         // the two DATA-DRIVEN area effects
      int ae_holy_inspiration;  //   (cross-confirmed in section 4)
      int minMorale; maxMorale; minMoraleSet; maxMoraleSet;
      int forceRallyStatus;
      int minRoutDelay;
    };

State enum (`luaEnums.cpp`): berserk 0, impetuous 1, high 2, firm 3,
shaken 4, wavering 5, routing 6. `unit.cpp:1923`: "moraleLevel use
moraleStatus enum". Rome Remastered's `unit_set_morale` command uses
the IDENTICAL state list — shared lineage confirmed.

Verdict: morale is a LEVEL recomputed from a base stat plus a list of
up to 34 concurrent situational effects (each with id and amount),
with overlapping state bands, timed shocks, and rally as a recoverable
timed process. This confirmed the owner's intuition and drove the
0055 rework from the old drain-accumulator to the recomputed level.

Fatigue [A]: per-soldier `uint16 fatigueCount` + `uint8 baseFatigue`
+ `char fatigueModifier` + `int groundFatigueModifier`, quantized
into a 4-bit state field; per-unit `int fatigue` mirror. States
(display names, [C] universally attested): fresh, warmed up, winded,
tired, very tired, exhausted.

ESTABLISHED NEGATIVES [verified by hand, full-repo grep + GitHub-wide
code search]: the 34 effect ids are named/valued NOWHERE public. EOP
hooks zero morale/fatigue functions (functionsOffsets.cpp checked).
No other public disassembly project has these structs. No cheat table
publishes morale values (Recifense's table has a stamina freeze only,
proving the fatigue counter was located but publishing nothing).
TWC's modern threads on thresholds are Cloudflare-walled and
unarchived.

Bonus recovered for the archers milestone [A]: EOP's byte-faithful
reimplementation of M2TW's ARROW kill chance
(`patchesForGame.cpp onCalcArrowKillChance`): height advantage
attack += clamp(heightDiff x 0.2, -4, +4); dense forest -6; AP halves
armour, gunpowder-AP halves it AGAIN; shield full frontally, halved
from flank arcs (45-degree sectors centered at +-90 degrees), halved
vs gunpowder; conversion to kill chance per mille: a>=-6: 20a+200,
-12<=a<-6: 10a+140, a<-12: 4a+68, clamp 1..990 (attack diff 0 = 20%
per hit).

## 2. Rome Remastered official hardcoded values [A for RTW lineage]

`Battle_and_Campaign_Formulae.md` (Feral GitHub, read in full,
"cannot be modded"):

- General: bodyguard +12 while rallying / +8 otherwise; units within
  influence +10 rallying / +4 otherwise; ALL units army-wide while
  the general lives: 2 + command + influence/2 + TroopMorale.
  Influence radius = 6 + 7 x command + 4 x influence (meters). Only
  the army's own commanding general counts. THE CAPTAIN FLOOR: at 0
  command / 0 influence / no traits the army-wide term is still +2
  and the radius ~6 m — basis of the 0055 leadership term.
- Auras (75 m, non-stacking, don't self-buff): chant +6 friendly /
  -5 enemy (+2 to-hit majority); screech +3 / -5 (-2 enemy to-hit
  majority); eagle (`command` attr) +4.
- Fear (frighten_foot/mounted, 100 m): 1 unit -4, 2 units -6, 3+ -8.
- Hidden +4; ambush reveal -7 for 60 s to enemies within 80 m facing
  away.
- Formations: testudo +4 morale (-10 atk +5 def), schiltrom +3,
  phalanx +2, loose -2 (+2 if elephants near), wedge +10 atk -5 def.
- Warcry: attack interval -40%..-100% over 2.8..7 s charge, 40 s.
- Experience to-hit: 9+(attExp-defExp) lookup
  [-6,-5,-4,-4,-3,-3,-2,-2,-1,0,1,2,2,3,3,4,4,5,6].
- Battle difficulty (enemy to-hit): Easy -6 / Normal 0 / Hard +4 /
  Extreme +7.
- Discipline semantics (RR EDU.md): "determines the amount of morale
  lost when morale shocks occur (death of general, flanked, etc)";
  impetuous may charge unordered; berserk = 75% attack-delay cut.
- stat_mental morale UI labels: 1-2 Poor, 8-11 Good, 12+ Excellent;
  base morale caps at 128. M2TW adds optional `lock_morale` 4th token.
- Fatigue knobs: hardy -2 / very_hardy -4 / extremely_hardy -8
  (stack; RR-only third tier), inexhaustible; stat_heat -2..5.
- rout_str_mod 0.25 (routing units count 25% strength in AI eval).

ESTABLISHED NEGATIVE [hand-verified repo grep]: casualty penalties,
flank-threat values, routing-neighbor values, outnumbered penalties,
state thresholds, and fatigue rates/state penalties are in NO RR doc
and NOT in `data_controlled_features.md` (the 2.0.4 hardcode-to-data
file). RR 2.0.4 changelog: UI now displays CURRENT morale including
bonuses — a black-box measurement path for RTW-lineage values.

## 3. The MTW1 official table [B] — the design template

Archived totalwar.org wiki `MTW_Morale` (BradyGames guide + CA dev
posts; fetched verbatim from Wayback) and Puzz3D's thread 75234.
This is the ancestor table CA carried forward; nearly every "RTW/
M2TW morale numbers" forum post is an uncredited copy of it. Where
MTW and RR overlap, values CHANGED (general: MTW +1/star within 50 m
vs RR's formula; ambush -8 vs -7/60 s) — so treat as design shape,
not M2TW fact.

States: Impetuous 10+, Steady 2..14, Uncertain -5..5, Wavering
-14..-5, ROUTING AT -6 OR LESS; bands overlap deliberately ("prevents
thrashing"); a routing unit "runs until conditions shift its morale
back to Wavering" (recover above -5). MP rout point -18.

Negatives: casualties 10% -2 / 50% -8 / 80% -12 (of start-of-battle
size); losing melee up to -8 (infantry losing to cavalry additional
-6, Puzz3D: -14 total); one flank threatened -2, both flanks or
flank+rear -6 (range ~60 m, don't stack); charged in flank/rear -4
(cavalry vs infantry -6); ambushed/charged by hidden -8; under
missile fire -2 (fear-causing weapons additional -4); routing friends
up to -12 AT TWO WEIGHTED UNITS (discipline weighting: elites count
other elites as 1, all others as HALF; disciplined count elite+
disciplined as 1, others half; normal counts all as 1) — verbatim:
"This is the main factor in how chain routings occur, which is why
it is a good idea to hammer the end of a flank, start a rout, and
just roll up the enemy line"; general killed -8 for a few seconds
then -2 permanent plus loss of star bonuses; army outnumbered (and
outclassed) 2:1 -4 up to 10:1 -12 (range ~75 m per one thread);
loose/disordered -2 (charged while disordered -6); skirmisher forced
to flee -6, out of ammo further -6.

Positives: both flanks/flank+rear covered by friendlies +4; no enemy
nearby +4; routing enemies up to +8 at two weighted units; uphill
+2; winning melee up to +6; impetuous charge +4; locally outnumber
3:1 +4; general's own unit +2; within 50 m of general +1/command
star, beyond +1/2 stars; rally +8; valour +2/level; cornered army /
siege defenders +8; difficulty Easy human +4 / Expert AI +4;
morale-off +12.

Combat-result terms update per combat cycle ("can swing greatly from
second to second").

Fatigue (wiki `MTW_Fatigue`, fetched verbatim): 5 states; Fresh no
effect; Tired -2 attack; Quite Tired -3 attack -3 morale; Exhausted
-4 attack -6 morale; Completely Exhausted -6 attack -8 morale and
CANNOT RUN OR CHARGE. Depletion causes/rates: the wiki itself says
"TBD" — NEVER documented, for any title in the lineage [F].

MISATTRIBUTION WARNING [F]: the "Broken -15..-2 / Shaken -3..2 /
wavering 40 s / broken 600 s" numbers circulating as RTW/M2TW facts
are daniu's extracted NAPOLEON `_kv_morale` db (Warscape). Reject.

M2TW-specific qualitative [C]: base morale globally higher than RTW
(militia hold to ~50% dead frontally; elites near death); non-fire
missile casualties barely move morale out of melee; gunpowder
casualties huge; casualty counts only after the death animation.

## 4. The owner's install: vanilla data findings [A]

Install: /data2/SteamLibraryFlatpak/SteamLibrary/steamapps/common/
Medieval II Total War — WINDOWS M2TW (medieval2.exe 1.52 + official
unpacker + Stainless Steel in mods/). NOT the Feral DE of devlog
0031; that dead end (custom LZSS) is bypassed entirely. Method:
`tools/unpacker/unpacker.exe` under Proton 8.0 wine
(dist/bin/wine, scratch WINEPREFIX, `yes |` through its 3 prompts,
`--source=<packs> --destination=<scratchpad>/m2tw_vanilla` so the
live install is untouched). All 5 packs extracted; 211 text/xml
files.

Recovered engine-DATA morale values (vanilla descr_area_effects.xml
— the only two data-driven morale effects, matching the two named
moraleStruct fields, disassembly and data cross-confirming):

    ae_holy_inspiration   +5 morale   radius 50 m
    ae_cow_carcass        -5 morale   radius 10 m, duration 300 s,
                          morale_max 16
    (sship adds ae_greek_fire_fear -10, radius 5, 30 s)

`morale_max 16` = the engine's working morale ceiling; our base
scale (light 8 / spear 9 / heavy 14) sits correctly under it.

descr_climates.txt: per-climate `heat` 1..4, header comment "zero
would mean no armour effects to fatigue at all" — armour x climate
heat drives fatigue accumulation in-engine; validates our per-kind
fatigue_rate (armour analog) shape.

export_descr_character_traits.txt: `TroopMorale` is a real M2TW
trait attribute (160 occurrences, +-1..3 per trait level) — the
trait term in the leadership formula.

battle_config.xml [negative]: NO morale/fatigue content — AI combat
balancing, skirmish ranges, siege-ladder queueing (sship's copy is
ReallyBadAI's, e.g. melee-hit-rate 2.0). descr_campaign_db.xml is
campaign-side. EDU stat_mental rows match the devlog 0031 mirror
calibration.

## 5. Ground-truth extraction paths (untapped)

1. M2TWEOP on the owner's install (Windows M2TW + Proton makes this
   feasible on this box): Lua-read `moraleStruct.effects[34]`
   (id/modAmount/amount) live during staged custom battles — flank a
   unit, park an ally at measured range, kill a leader — and recover
   the actual M2TW factor table nobody has published. Risks: EOP
   exe-version match (targets Steam 1.52 — present), DLL injection
   under wine.
2. Rome Remastered's live-morale UI (2.0.4+): black-box measure
   RTW-lineage values by staging and reading the displayed number.

## 6. Implementation mapping (what 0055 built vs evidence tier)

    factor                 our value                        tier
    model shape            level = base + summed effects    A (EOP)
    states + hysteresis    waver <=0, rout <=-6 held 1 s,   B (MTW bands)
                           rally >-5 after 8 s              + A (waveringTimer,
                                                              minRoutDelay exist)
    casualty curve         (10%,-2)(50%,-8)(80%,-12) interp B
    losing/winning melee   -8..+6 x intensity, 3 s EMA      B values, ours shape
    flank threat           -6 x flanked01 x discipline      B values, ours ring
    outnumbered local      -2 at >3:1 density               B-ish, owner-demoted
    contagion              -6/weighted router, sat 2 (-12)  B verbatim
      class weighting      drilled half-count lessers       B verbatim
    routing enemies        +4/weighted, sat 2 (+8)          B
    secure flanks          +4 x friends/3 x (1-flanked01)   B value, ours shape
    no enemy near          +4                               B
    wall stance            +2                               A (RTW phalanx)
    leadership             +2 army-wide captain floor;      A (RR formula floor);
                           break -8 10 s then -2 perm       B (MTW shock)
    disorder               0..-3 engaged                    NOT evidenced (kept
                                                            from owner model)
    fatigue states/effects atk -2/-3/-4/-6, morale          B (MTW official)
                           -3/-6/-8, no charge exhausted
    fatigue rates          fight .55 charge .8 move .3      F — undocumented for
                           recover .35, bands 15/30/50/70/85  the entire lineage
    fatigue kind mult      1.3/1.1/1.0                      A shape (heat x
                                                            armour), F values
    base morale            8/9/14                           A scale (morale_max
                                                            16, EDU vanilla)
    radii (60 m neighbor)  60 m                             ~B (flank-threat
                                                            "~60 m" note; rout
                                                            radius unknown [F])

Not yet built (evidenced, queued): real generals (stars, aura
formula, rally +8/+10), timed charged-in-flank shock -4 (cav -6),
missile-fire -2 (fear +4) when archers land, uphill +2, army-wide
outnumbered 2:1..10:1 (owner ruling needed: conflicts with "soldiers
can't count" for the local case), cornered +8.
