# 0074 — Rout soundscape (2026-09-19)

System 4 of the devlog 0070 plan, commit 948f763 on
fix/controls-audio-qol. Design direction came from the OWNER'S film
research (Napoleon 2023, The Patriot, Monty Python's castle scene):
after the first panicked moments, fleeing men are mostly SILENT —
what reads is commanders shouting "Retreat!" / "Withdraw!" over the
mass, and the running itself. This matches the M2TW config exactly:
Individual_Retreat is officer voice lines at sparse p .08, the group
retreat sheet is benched in M2TW's own files, and the mass sound is
the unit_run footstep bank.

## Assets (assets/sfx_rout/, 23 files committed)

- 10 command shouts (fallback/retreat/withdraw/run_away variants,
  1-1.5 s, hot at -8..-12 dB).
- 3 initial-panic screams (vox_panic_01..03).
- vox_rout_04/05: extend the break-edge crowd pool (01..03 kept at
  owner's call).
- 8 single/two-man footstep source loops. ElevenLabs would not
  produce a massed wash, so `work/scripts/build-feet-wash.sh` layers 12
  jittered copies (asetrate pitch/tempo, delays, per-layer volume,
  soft fades) into feet_run_wash_mass_01/02 (6 s, -12/-14 dB mean).
  Rerun the script after regenerating sources.

## Implementation (audio.rs::rout_vox)

Per BROKEN regiment (Routing and Shattered), all camera-thinned by
volume like the rest of the reworked mix:

- Panic window: 7 s opened on the break edge (own prev_broken
  tracking, self-contained); screams budget from panicking men near
  the camera, cap 1/frame. After the window the mass goes quiet.
- Command shouts: global accumulator at best_prox x 0.35/s — one
  officer voice every ~3 s while any rout runs in earshot, never a
  machine gun during a mass collapse. Gain 0.30-0.38.
- Feet wash: per-regiment rolling clock (3-4 s retrigger under the
  6 s clips), first wash on the break edge, gain scaled by regiment
  strength (300-man band) — base 0.20.
- The break EDGE stays in event_cues: vox_rout crowd (now 5 clips) +
  horn_rout for own breaks.

## Rally vox benched

vox_rally_01/02: owner ruled "don't use anywhere". Removed from the
bank and the rally branch in event_cues plays NOTHING now — a rally
cue is an open asset gap (needs a fresh clip, e.g. a rallying horn
or an officer "hold, reform!" line) for a future batch.

## Verification

Build + clippy zero warnings. Owner feel pass pending: break a
flank, chase it — first seconds scream, then the fleeing block goes
quiet under shouted orders and drumming feet; the feet bed follows
each fleeing regiment, not the camera. Knobs: shout rate 0.35/s,
panic window 7 s + rate 0.00066/man, wash gain 0.20 + 3-4 s
retrigger.

Remaining from the 0070 plan: section 1, the DEATH rework (assets
next per owner: screams 06..10 + regen 01..05, death_hit_01..06,
bodyfall_01..03 -> assets/sfx_death/). That closes battle audio.
