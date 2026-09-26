# 0071 — HANDOFF: macbook transition, battle audio WIP (2026-08-01)

Owner is traveling from 2026-08-02 and switching to a macbook. This
doc is the next agent's entry point. Read it whole before acting.

## TRANSFER CHECKLIST (owner, before leaving this machine)

Git syncs NONE of the working context. Untracked by design:
`devlogs/` (all of them), `tmp/` (prompts, drafts, evidence cache),
`CLAUDE.md`, and two spare clips in `assets/sfx_charge/`
(`burn_sfx_lol.mp3`, `misc_combat_ambience0.mp3`).

1. PUSH the branch: `fix/controls-audio-qol` is 11 commits, LOCAL
   ONLY (no upstream). The agent cannot push (CLAUDE.md rule: no push
   without permission). `git push -u origin fix/controls-audio-qol`.
2. Copy to the macbook: `devlogs/`, `tmp/`, `CLAUDE.md`, the two
   untracked clips above.
3. The M2TW install (`/data2/SteamLibraryFlatpak/.../Medieval II
   Total War`) does NOT travel, but the mined data does:
   `work/research/m2tw-sounds/` = the full unpacked SSHIP sound config set (35
   files, every descr_sounds_* + the soldier/unit voice exports —
   the primary source for devlogs 0069/0070 and any future audio
   system), `work/research/archer-evidence/` = EOP structs + projectile/EDU
   docs, `work/research/m2tw-extraction/` = live morale probe CSVs. New
   LIVE-game probing still waits until the owner is back home.
4. Audio generation is WIP on ElevenLabs (browser side, owner does
   it): prompt sheets `work/audio/charge-sfx-prompts.md` (DONE, shipped) and
   `work/audio/battle-sfx-prompts.md` (IN PROGRESS — this is the active
   batch).

## Where the project stands

Branch `fix/controls-audio-qol` off main (`b937798`), 11 commits,
build + clippy clean at every commit. 0.1.0 plan context: this
bugfix/QoL/audio branch came FIRST by owner decision; scenario format
and itch packaging (see work/handoffs/HANDOFF-scenario-format) follow
after it merges.

Landed on the branch (devlogs 0068, 0069):

- Controls/QoL: M2TW drag-hand line facing (camera-derived hint,
  orders.rs::line_layout), UiCue message-driven UI sounds (repeats
  click now), one warcry then full charge-audio rebuild (below),
  Ctrl+A/I/M class selects, Ctrl gates camera WASD, Enter clears
  selection (not during deployment: Enter = Begin Battle there),
  shift-click card range select, box drag-select setting
  (Settings > Controls, lasso default).
- Archers: fire-at-will holds against melee-locked targets (explicit
  order still forces the shot); each arrow aims at a hash-picked
  LIVING SOLDIER of the target regiment (movement.rs, per-soldier aim
  targets are M2TW-evidenced), not the footprint disc. Flight keeps
  zero team checks.
- Charge audio REBUILT and OWNER-APPROVED ("much better"): 29
  single-man yells + 3 group sheets in `assets/sfx_charge/`
  (committed), system `audio.rs::charge_vox`. Yells: accumulator at
  0.2/man over ~4.5 s, prox^2-weighted, caps + the 64-decoder
  allowance. Sheets: per-regiment clock, first at charge entry, then
  2.0+0.5*rand retrigger, banded at 300 men, gain
  0.22*(0.3+0.7prox)*zoom.max(0.55) (first cut 0.10 was inaudible —
  owner caught it). Old vox_warcry clips: benched, kept in repo FOR
  THE RECORD, do not delete.

Feel-pass status: charge audio approved. Drag-hand facing, archer
hold-fire + per-soldier aim, box select, hotkeys: implemented, owner
play-test still pending. Do not merge the branch until he passes it.

## THE ACTIVE WORK: battle audio rework (devlog 0070 is the spec)

Owner priority statement: audio is the immersion carrier; if audio is
good, low-poly graphics are forgivable. Four systems, mined M2TW
models in devlog 0070, asset list in work/audio/battle-sfx-prompts.md.
Agreed order (feel value per effort):

1. DEATH: budget screams from deaths near the camera (accumulator
   pattern like clangs), replacing the 1-per-0.5-1s global cooldown
   in audio.rs::combat_one_shots (kill-delta + death_cooldown local).
   Add a killing-blow thud pool (M2TW death_hit, LOUDER than normal
   hits: volume 0 prio 180 vs -25 prio 90). Assets: sfx_death_06..10,
   death_hit_01..06, bodyfall_01..03 -> assets/sfx_death/.
2. MELEE HUMAN LAYER: new system, same shape as charge_vox. M2TW
   per-soldier probabilities: attack grunt .4, attack scream .25,
   victim grunt/groan .25, battle scream .2. Feed it from engaged
   soldiers near camera (SimStats.events is hits, not swings — pick
   the honest signal when implementing). Assets: ~30 clips ->
   assets/sfx_melee_vox/.
3. CELEBRATE: key off EXISTING GroupData::celebrate ticks (set on
   hostile_near falling edge). M2TW: rolling full-volume group cheer
   sheets (fadein 1 fadeout 3, retrigger ~randomdelay 1) + individual
   whoops p .08. Existing vox_cheer clips join the sheet pool; the
   one-shot cheer edge in event_cues gets replaced by the state
   audio. Assets -> assets/sfx_celebrate/.
4. ROUT: M2TW's own group retreat sheet is BENCHED in its config —
   skip ours too. Audible rout = individual panic yells (p .08) +
   massed running FEET. Plan: panic-yell budget from routing men near
   camera + feet wash per fast-moving regiment (also upgrades normal
   running). horn_rout + vox_rout stay as the break-edge announce.
   Assets -> assets/sfx_rout/.

Engine facts the next agent needs (all in src/audio.rs):

- charge_vox is the template: Local state struct, fractional
  accumulator, per-frame spawn cap, MAX_LIVE_ONE_SHOTS (64)
  allowance via the `playing` query, prox = 1 - dist/hear with
  hear = 220 + cam.distance*0.5, zoom_attenuation() for close-up
  gating, one_shot() with pitch jitter, pick() by hash.
- Loudness ledger: ElevenLabs clips arrive ~-11 dB mean / 0 dB peak.
  Current gains: yells 0.20-0.30, charge sheets 0.22 base, clangs
  0.16-0.26, deaths 0.18-0.30, UI 0.35-0.5, horns/vox 0.5-0.55.
  Owner tunes by ear fast — ship a reasonable gain, he corrects.
- Audit new clips with ffmpeg volumedetect before choosing gains
  (0024 lesson: "I can't hear it" is a mix claim, check numbers).
- FL_VOLUME=0 mutes test runs. Paused sim: gate on
  Time<Virtual>::is_paused (accumulators drip phantom sound
  otherwise).

## Process rules (bit us before; owner enforces)

- Propose feel-critical designs in plain words BEFORE building.
  M2TW-evidenced behavior ONLY; cite the devlog/config line.
- STE lint on prose AND commit messages: flat present-tense titles,
  no semicolons, sentences <= 25 words, no decorative words, no em
  dashes anywhere.
- Build: cargo build --profile opt-dev + clippy, ZERO warnings. No
  UI click loops; owner play-tests live. pkill must be
  `pkill -x flanks`. Never steal window focus.
- Devlog per work chunk (next: 0072), devlogs NEVER committed.
  PR drafts -> work/drafts/pr-<feature>.md. Repo is PUBLIC: no
  force-pushes, commit messages are public prose.
- No git push, rm, checkout of dirty trees, or history surgery
  without owner permission. Backup ref + verified bundle before any
  ref surgery.
- Commit per milestone. `git add` individual files, never -A.

## Open threads beyond audio

- Feel pass on the whole branch, then PR draft (work/drafts/), merge.
- Deferred from earlier milestones: show rolled enemy style
  in-battle, random-enemy scaling vs partial player army, comp
  persistence, scenario format -> itch 0.1.0 (the previous handoff).
- misc_combat_ambience0.mp3 (untracked): candidate for the optional
  per-regiment fight sheet (section 3 of battle-sfx-prompts).
- Terrain/maps milestone parked for 0.2.0 (owner decision 2026-07-30):
  map variety + slope fatigue/charge physics + scenarios, designed
  simulate-not-fake.
