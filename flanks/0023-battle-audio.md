# 0023 — Battle audio: generation, mixer, zoom tuning (2026-07-10)

Commits `da8f988` + `cb1e48e` on `feat/battle-feel`. Owner generated two
ElevenLabs batches (`assets/*.mp3`, `assets/sfx_new/*.mp3`); the mixer
plays them from aggregate sim signals. Full asset list + prompts + retry
notes: `tmp/audio-plan.md`.

## Architecture (`src/audio.rs`)

Design rule at 200k units: NEVER per-unit audio. Two mechanisms:

1. **Beds** — 4 looping `AudioPlayer` entities, per-frame volume targets
   with ~0.35 s smoothing (`update_beds`):
   - `Far`: total engaged regiments (the distant din).
   - `Mid`: engaged regiments within 300 m of camera focus × proximity.
   - `Close`: proximity² × hits/tick × zoom attenuation — the inside-the-
     melee wall of steel.
   - `Drums`: any own regiment marching (order set, not broken).
2. **One-shots** — fire-and-forget `PlaybackSettings::DESPAWN` entities
   with volume/pitch jitter:
   - Hit connects (`combat_one_shots`): budget = hits/tick × 0.02 ×
     proximity² × zoom, fractional accumulator, hard 2/frame cap. Pool:
     60% sword/armor clangs, 20% shield thuds, 20% flesh-damage connects.
   - Death screams: kill-count deltas, rate-limited (0.5–1 s), hard zoom
     cutoff.
   - Event cues (`event_cues`, transition-edge detection on Groups):
     charge horn on new own orders (3 s gate), war cry on first contact,
     rout wail + alarm horn on breaks, rally cheer, selection click,
     victory/defeat stings. One vox per 1.2–1.5 s window, most dramatic
     wins.

Zero coupling: the audio systems only READ Groups/SimStats/CombatStats/
Selection/BattleOutcome/RtsCamera and detect edges themselves.

## The zoom problem (owner feedback after first listen)

Clangs and screams played at full volume at any zoom — "I keep hearing
it until the game ends". Fix: `zoom_attenuation(cam.distance)` = full
inside 90 m, floor 0.12 surveying the map; it scales the clang RATE and
volume, the scream volume (plus a hard cutoff — you should never pick
out one scream from a hilltop), and 65% of the close bed. Zooming out
hands the mix to the far/mid din, zooming in lands you in the steel.
Base one-shot levels also roughly halved. Proper per-source spatial
audio (bevy spatial or kira) is the known next step; this approximation
already fixes the "wrong at wide zoom" feel.

## Asset state

- In use: 6 clangs (batch-2 sword 06–09 + unplanned armor ×2 — owner
  preferred these), 3 shields, 5 damage connects, 5 deaths, 3 rout vox,
  4 rally vox, 2 war cries, 2 charge horns, new rout horn, drums, new
  ui_select, 2 stings, far/mid/close beds.
- Still missing (retry prompts in audio-plan): `bed_march` (army-scale
  footsteps), `bed_wind_field` (base ambience), `ui_order`, optional
  bodyfalls. Unused spares: first-batch clangs, mid1/close1/close2 bed
  variants, single-person warcry.
- ElevenLabs prompt lessons: the model latches onto the first concrete
  noun — put crowd/scale words FIRST and plural ("massive crowd of
  thousands…"), add "no words" to prevent solo yells, avoid "gentle"
  (near-silence) and bar counts. Single-event sounds are reliable;
  crowd-scale sounds need 2–3 attempts.

## Plumbing notes

- bevy `mp3` feature enabled (generated files are mp3; default is
  vorbis-only).
- `AssetPlugin.file_path` anchored to `CARGO_MANIFEST_DIR/assets` —
  bevy otherwise resolves assets relative to the EXECUTABLE for direct
  `./target/...` runs (only `cargo run` gets the manifest dir).
- `FL_VOLUME` master knob. Missing assets degrade to silence (asset-not-
  found errors in the log, nothing crashes).
- All mix constants live at the top of the systems in `audio.rs` —
  ear-tuning is one-line edits.
