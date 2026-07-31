# 0069 — M2TW charge audio + arrow aim evidence, aim fix (2026-07-30)

Owner played live M2TW and flagged two divergences from our build.
Both confirmed against local data: the SSHIP mod in the owner's M2TW
install ships the unpacked vanilla-derived sound configs
(`mods/sship/data/descr_sounds_*.txt`), and the M2TWEOP struct dump
was already cached at `tmp/archer-evidence/eop_unit.h` (devlog 0060).

## Charge audio: two layers, not one roar

From `descr_sounds_units_charge.txt` and
`export_descr_sounds_soldier_voice.txt`:

- `Individual_Charge` soldier vocal: `probability .2 priority 80
  mindist 4 volume -30`, pool of ~40 single-man `Yell_Charge_*.wav`
  clips. One in five charging soldiers yells his own positional clip.
  This layer carries the moment.
- `unit_charge` state bank: one 3d group crowd clip per charging
  unit. `DEFAULT: volume -25 probability 1 effect_level .25 delay 2.0
  randomdelay .5`; clip picked by unit size (`lod 5/10/30` = small /
  medium / large). Retriggers every 2.0 +- 0.5 s while the charge
  state lasts. Subtle background texture.
- The clean single "Charge!" voice is the order confirm
  (`Unit_Attack confirm`, units_voice bank) — our ui_attack click +
  horn already fill that slot.

Our single loud crowd clip at the charge edge matched neither layer;
the loudness hierarchy was inverted.

REBUILT (2026-07-31, audio.rs::charge_vox). Owner generated the pool
on ElevenLabs (prompts: `tmp/charge-sfx-prompts.md`): 29 single-man
yells + 3 group sheets in `assets/sfx_charge/` (all ~-11 dB mean,
0 dB peak — hotter than the old warcry's -14.8, hierarchy lives in
the code gains). Two layers, driven by `GroupData::charging`, both
teams:

- Yells: fractional accumulator (the clang/loose pattern) fed by
  charging men weighted prox^2, at the M2TW rate 0.2 per man over
  the ~4.5 s mid of the measured 2.8..7 s charge window. Caps: acc 3,
  2 spawns/frame, MAX_LIVE_ONE_SHOTS allowance (our 1000-man
  regiments would otherwise flood the mixer that M2TW's 60-150-man
  units never could). Volume 0.20-0.30 x prox x zoom.
- Sheets: per-regiment clock, first sheet on charge entry, then
  2.0 + 0.5*rand retrigger (the M2TW delay/randomdelay), clip banded
  at 300 men (medium/large; no small asset, medium covers), volume
  0.10 — the -25 dB + effect_level .25 analog, well under the yells.

The cry-edge machinery in event_cues (prev_order/prev_cry retarget
detection, devlog 0024) is deleted with the vox_warcry bank. The old
`sfx_new/vox_warcry_01/02` clips stay in the repo FOR THE RECORD
(owner decision): benched, loaded by nothing, do not delete. The
order-confirm slot remains ui_attack + horn. `burn_sfx_lol` /
`misc_combat_ambience0` in sfx_charge/ are owner extras, untracked,
not loaded.

Mix fix after first listen: the sheet at 0.10 gain sat near half a
single yell and vanished under yells + beds. Raised to
`0.22 * (0.3 + 0.7 prox) * zoom_att.max(0.55)` (~0.12 linear at
battle zoom) and the layer reads. OWNER-APPROVED 2026-07-31: "much
better, the charging voices sound good instead of the old unison
warcry". Charge audio feel pass done; remaining knobs are the yell
gain (0.20-0.30) and rate (0.2 per man / 4.5 s) if a later battle
reads too thick.

## Arrow aim: per-soldier targets, not the footprint disc

Evidence: `soldierInBattle` carries `aimTargetValid : 1` and
`aimTargetX/Y/Z` (eop_unit.h ~996-1009) — every M2TW soldier aims at
his own point. Victim side has `interceptedMissileTarget`, matching
our whoever-the-shaft-crosses flight. Accuracy is the
range-independent `accuracy_vs_units` scatter (devlog 0060).

Ours aimed every shot at a uniform random spot on the target
regiment's footprint disc (centroid + 0.85r). Interpenetrated melee
blocks put friends inside that disc, so forced volleys into a fight
soaked allies evenly — the owner's complaint.

FIXED (movement.rs): step_sim now builds a living-member index list
for each regiment under fire this tick (only the targeted few), and
each shot hash-picks one member and aims at his tick-start position,
keeping the block-drift lead and the scatter sigma. The disc spot
remains only as a fallback when the member list is empty. Friendly
fire persists solely through real misses and interception — no team
check was added to the flight (devlog 0060 model intact).

Calibration note: aim now follows soldier density instead of a
uniform disc. If kill rates read hot/cold in the feel pass,
`missile::BASE_DMG` / `FACTOR_MULT` stay the knobs (devlog 0064).

## Verification

Build + clippy zero warnings. Owner feel pass pending: forced volleys
into melee should mostly find enemy bodies now; FF should drop
noticeably but not to zero.
