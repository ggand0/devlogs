# 0070 — M2TW battle audio evidence: fight, death, celebrate, retreat (2026-07-31)

Owner direction: audio is the immersion carrier for this game; good
audio buys forgiveness for low-poly graphics. Next audio milestones:
combat ambience, soldier death, celebration on enemy rout, retreat
ambience. All mined from the SSHIP unpacked configs
(`mods/sship/data/descr_sounds_*.txt`, vanilla-derived) in the
owner's M2TW install, same source as devlog 0069.

## The M2TW pattern, once

Every battle sound is one of two shapes. STATE BANKS (unit_idle,
unit_march, unit_run, unit_charge, unit_fighting, unit_celebrate,
unit_retreat): one 3d positional clip per unit in the state, clip
picked by unit size (`lod` thresholds), retriggered or crossfaded
while the state lasts. PER-SOLDIER EVENTS (soldier_voice vocals,
weapon_hit): fired per soldier per event with a probability gate,
positional, tiny mindist. Foreground detail comes from the individual
layer; the group sheet is glue. We already replicated this shape for
charges (devlog 0069); the same recipe covers everything below.

## Combat ambience (melee)

- Group sheet: `unit_fighting` bank — per FIGHTING unit, lod 3/40/80
  (Group_Fight_Small/Medium/Large), `volume -40 fadein 2 fadeout 2
  effect_level .25`. A true crossfading loop bed per engaged unit,
  very quiet; the fight reads mostly through the layers below.
- Weapon hits: `weapon_hit` bank, BY MATERIAL — sword has `hit metal
  (-20)`, `hit wood (0)`, `hit flesh (0)`, `hit leather (0)`, plus
  `death_hit flesh (volume 0, priority 180, 10-clip Death_Hits pool)`
  played on KILLING blows. Defaults `mindist 3 volume -25
  probability 1 probradius 7`. Every swing connect makes a sound; the
  material comes from what it hit.
- Per-soldier combat vocals, DURING melee: `Individual_Attack_Grunt`
  (p .4, -20, the attacker's effort grunt), `Individual_Attack_Scream`
  (p .25, -15), `Individual_Grunt`/`Individual_Groan` (p .25, -20,
  the victim), `Individual_Battle_Scream` (p .2, -10). This human
  layer over the steel is most of what "M2TW melee" sounds like.
- Ours today: three camera-global beds + clang/shield/damage one-shots
  budgeted from hit stats. Missing: the human grunt/scream layer
  (biggest gap), death-blow hits as their own louder pool, and
  per-regiment positional fight sheets (owner's spare
  `misc_combat_ambience0` clip is a candidate).

## Soldier death

- `Individual_Death`: probability 1 (default), `volume 0` (FULL),
  `priority 130 mindist 1.5`, 8+ clip pool. EVERY dying man screams,
  positionally; distance culls what you hear, not a rate limiter.
- `death_hit flesh` (weapons bank): the meaty kill impact, volume 0,
  priority 180, separate 10-clip pool from ordinary hits.
- `Individual_Fall_Grunt` (p 1, -20) and `Individual_Fall_Scream`
  (looped, full volume, for falling deaths e.g. off walls).
- Ours today: one death scream per 0.5-1 s globally (cooldown), no
  kill-impact distinction. The M2TW read: budget screams from actual
  deaths/tick near the camera (the clang accumulator pattern), plus a
  death-hit thud pool at kill events. Our 5-clip sfx_death pool needs
  to grow (~10) to survive the higher rate.

## Celebrating a rout

- `unit_celebrate` state bank: per CELEBRATING unit, lod 5/10/30
  (small/small/large cheer clips), `volume 0` (full), `fadein 1
  fadeout 3 randomdelay 1 effect_level .5`. Rolling group cheers as
  long as the celebrate state lasts, not a one-shot.
- `Individual_Celebrate`: p .08 per soldier, individual whoops.
- Ours today: single vox_cheer one-shot on the enemy-break edge,
  vox-gated. We ALREADY track the state M2TW keys on:
  `GroupData::celebrate` ticks (render celebration, set on the
  hostile_near falling edge). Wire the audio to that state: per
  celebrating regiment a group cheer sheet on a ~2-3 s retrigger for
  the celebrate window, plus a budgeted individual whoop layer.
  vox_cheer pool (2 clips) can seed the group sheets; needs a
  small/large pair + individual whoops generated.

## Retreat / rout ambience

- `unit_retreat` state bank exists with fadein .25 fadeout 1
  `delay 1.5 randomdelay .5` — but every clip line is COMMENTED OUT
  in SSHIP (vanilla's infantry_group_retreat_small/large_01
  disabled). So the audible M2TW rout is NOT a group sheet: it is
  `Individual_Retreat` (p .08, panic lines — "fall back!"-type
  vocals), the rout announcement horn/vox (we have horn_rout +
  vox_rout already), and the unit_run FOOTSTEP mass: per-soldier
  run footsteps (p .5, terrain-dependent clip sets, -20, tiny
  mindist) that make a fleeing block sound like drumming feet.
- Ours today: horn + rout vox on the break edge only; routing mass is
  silent while it flees. Plan: budgeted panic-yell pool from routing
  men near the camera (mirror of the charge yells, sparser p .08
  feel) + a running-feet wash per moving-fast regiment (this also
  upgrades ordinary running, and the march bed stays for formed
  marches). Skip the group retreat sheet: M2TW itself benched it.

## Suggested order (feel value per effort)

1. Death screams to the accumulator model + death-hit pool (pure
   code + a few clips; melee instantly denser).
2. Melee grunt/scream human layer (the biggest single immersion gap;
   one new system, same shape as charge_vox).
3. Celebrate state audio (state already exists; small asset batch).
4. Rout panic yells + running-feet wash.

Prompt sheet for the whole asset batch: next session, same format as
work/audio/charge-sfx-prompts.md, once the owner picks scope.
