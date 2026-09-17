# 0073 — Celebrate state audio (2026-09-17)

System 3 of the devlog 0070 battle audio plan, commit 0807bb6 on
fix/controls-audio-qol. Owner generated 21 clips into
assets/sfx_celebrate/ (7 small group cheers: mocking / laughing /
joyous variants, 5 large, 9 whoops). Loudness audit: -7..-21 dB
mean, the usual spread.

## Model (M2TW unit_celebrate + Individual_Celebrate, devlog 0070)

The M2TW cheer is a STATE, not an edge: while a unit celebrates, the
unit_celebrate bank rolls positional group cheer clips (lod-banded
by size, FULL volume — volume 0, the loudest bank in the set —
fadein 1 fadeout 3, randomdelay 1) and soldiers whoop individually
(Individual_Celebrate p .08).

## Implementation (audio.rs::celebrate_vox)

Keyed to the EXISTING GroupData::celebrate window (150 ticks = 5 s,
set by frontline.rs when the last nearby unbroken foe routs or dies,
cancelled if hostiles return; both teams). Same shape as charge_vox:

- Group sheets: per-regiment clock, first sheet immediately on the
  celebrate edge, retrigger 2.5 + 1.0*rand so the 4-6 s clips
  overlap into a rolling roar. Size band at 300 men (small pool /
  large pool). Gain 0.40 x (0.3+0.7 prox) x zoom.max(0.5) — the
  loudest sheet in the ledger, above the charge sheets (0.22), per
  the M2TW full-volume evidence.
- Whoops: budget = celebrating men near camera x .08 over the 5 s
  window, rate cap 10/s, cap 2/frame, gain 0.22-0.30.

Replaced: the vox_cheer one-shot on the enemy break edge in
event_cues (and its new_break_enemy tracking). The rout vox + rout
horn on the break edge stay. Timing note: the cheer now lands when
the LAST nearby foe is gone, not on the first break — that is the
M2TW read and it stacks correctly with the rout vox instead of
colliding with it on the same frame.

Benched, owner call ("they suck at this point"): the old
sfx_new/vox_rally_03/04_celebrate crowd clips — no longer loaded,
still on disk; owner will delete them himself if the new set holds.
Also benched: vox_whoop_04_whoops_4s(skipfornow).mp3, by its own
filename.

## Verification

OWNER-APPROVED 2026-09-17 ("feels more like proper cheers now").
vox_rally_03/04_celebrate DELETED at his direction (de9f874; they
survive in git history). Next batches, owner order: section 5 (rout)
then section 1 (death).

Build + clippy zero warnings. Original listening notes: break an
army's flank and the victors' line should roll cheers for ~5 s as
each regiment's neighborhood clears, mocking laughter close up,
massed roars from the big blocks. Knobs: sheet gain 0.40, retrigger
2.5-3.5 s, whoop rate cap 10/s, band threshold 300.

Remaining from the 0070 plan: death rework (section 1 assets), rout
(section 5 assets).
