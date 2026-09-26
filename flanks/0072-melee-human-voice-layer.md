# 0072 — Melee human voice layer (2026-09-14)

Owner returned from travel (month+ gap; state recap = the 2026-08-14
handoff, nothing had moved) and generated the melee vocal batch from
work/audio/battle-sfx-prompts.md section 2: 57 clips in assets/sfx_melee/
(attack_grunts 21, attack_screams 12, hit_grunts 16, battle_screams
8). This is system 2 of the devlog 0070 plan, landed as commit
6ceffd8 on fix/controls-audio-qol. `stabbed?.mp3` renamed to
stabbed0.mp3 — `?` is illegal on Windows and would break the itch
build; watch generated filenames for `? * : " < > |`.

## Model (M2TW soldier_voice, devlog 0070 / work/research/m2tw-sounds/)

Per-soldier vocals during combat: Individual_Attack_Grunt p .4 per
swing (-20 dB), Individual_Attack_Scream p .25 (-15),
victim Individual_Grunt/Groan p .25 per hit (-20),
Individual_Battle_Scream p .2 (-10). All tiny mindist (0.75-2 m):
strictly close-up sounds.

## Implementation (audio.rs::melee_vox)

Two feeds, charge_vox shape (accumulator + caps + shared 64-decoder
allowance + pause gate):

- Per-hit vocals: budget = SimStats.events x 30 x dt x 0.035 x
  prox^2 x zoom (clangs use 0.02 — voices outnumber steel in M2TW).
  Each spawn picks by hash at the normalized M2TW ratio: 45% attack
  grunt, 28% victim hit grunt, 27% attack scream. Whiffed swings are
  unheard (the sim counts hits, not swings) — accepted undercount,
  the honest available signal.
- Ambient battle screams: budget = engaged men near camera
  (prox^2-weighted count) x 0.0008/s — about one scream per 1-2 s
  over a close 1000-man melee. Cap 1/frame.

Gains (clips measured -7..-25 dB mean, spread already matches the
M2TW hierarchy): grunts 0.16-0.22, hit grunts 0.18-0.24, attack
screams 0.22-0.30, battle screams 0.26-0.34, all x (0.25+0.75 prox)
x zoom x battle volume. Knobs if the feel pass reads wrong: the
0.035 hit fraction, the 0.0008 ambient rate, the per-pool gain
bands.

## Round 2: rate model corrected (cbe6b94, owner feel pass)

Owner: "not bad but underwhelming — I only hear small groups
clashing, and I liked hearing many clangs." Both complaints traced
to MY mix decisions, and the owner asked the right question (was
the diagnosis evidenced, and why was the model not reproduced):

- Evidence re-verified: weapon_hit bank DEFAULT is `probability 1`
  with NO rate knob anywhere in the format — M2TW fires every hit's
  steel positionally; vocals ride at p .4/.25/.25 and QUIETER
  (grunts -20 vs material hits -10..0). The claim that priority
  culling finishes the mixing is engine inference (Miles voice cap);
  the claim that rate never depends on the camera is config fact.
- The original port applied M2TW's per-event probabilities to our
  global thinning fractions (different base — flipped steel:voice to
  1:1.75) and inherited the devlog-0023 clang formula's prox^2 x
  zoom on the RATE, against the freshly mined model. Lesson: when
  new evidence lands, re-derive the old approximation, don't extend
  it.

Fix (combat_one_shots + melee_vox): hits are attributed to engaged
regiments by strength and weighted by earshot (audible share), the
rate caps stand in for M2TW's voice limit (steel 30/s, vocals 18/s =
0.6 per steel), and camera distance/zoom shape VOLUME only. Grunt
gains dropped a notch (0.13-0.21) so steel carries; screams keep
0.22-0.30. Ambient scream rate no longer zoom-scaled either. At
battle zoom a big lock now stays DENSE and gets quieter, instead of
getting sparse.

## Verification

Build + clippy zero warnings at both commits. Round 2
OWNER-APPROVED 2026-09-14 ("sounds good now"). Claim tiers on
request: the structure (steel p 1, vocal ratios, rate-from-sim,
distance-as-volume) is M2TW-confirmed; the audible-share hit
attribution, the 30/18 rate caps, the 0.6 vocal ratio, and the
volume curves are OUR-CALIBRATION stand-ins forced by the aggregate
no-per-unit-audio architecture — those four are the honest knobs.

Remaining from the 0070 plan: death rework (assets not yet
generated), celebrate, rout. Sheet: work/audio/battle-sfx-prompts.md.
