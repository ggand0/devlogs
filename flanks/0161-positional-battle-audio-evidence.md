# 0161 — Positional battle audio: M2TW evidence and diagnosis (2026-09-29)

Written by Claude Opus 5.5.

Gota's play notes before 0.2.0: charge yells and archer looses often do not play when the camera is near them, sword clangs play too much, and the mix sounds flat, with no sense of direction or distance. The goal is to follow M2TW more closely this time, revisiting the round-2 melee rate model of devlog 0072 (the unit models were low poly then and the clang density read as the fight). This entry is the investigation: code read of src/audio.rs, the mined SSHIP configs in work/research/m2tw-sounds/, and the bevy_audio 0.19 / rodio 0.22.2 sources. Figure: work/notes/vis/025-audio-proximity-and-voices.png.

## Diagnosis

1. Missing charge yells and bow looses: voice starvation. Every one-shot shares MAX_LIVE_ONE_SHOTS = 64. The systems run chained in the order beds, combat_one_shots, melee_vox, archer_one_shots, arrow_fly_loops, event_cues, charge_vox, celebrate_vox, rout_vox. combat_one_shots never checks the allowance. In any melee within earshot the steel sits at its 30/s cap with 1.1 s clips (about 31 live), the vocals at 18/s with 1.03 s clips (about 18.5 live), and the arrow fly loops hold up to 12. That is about 62 of 64 before charge_vox and archer_one_shots run, so they get 0 to 2 voices. M2TW orders this the other way: the archer group fire bank has priority 190 and the charge sheet 170, a weapon hit 90.
2. Clang spam: the steel rate is the global landed-blow count times the "audible share" of engaged men within about 200 m of the camera focus, capped at 30/s. At 200k it sits on the cap whenever any front is within reach, and every clang plays at the gain of the nearest engaged regiment. So a big fight 150 m away and three men 5 m away both give 30 clangs a second at the same loudness.
3. Flat, no direction: no sound is positional. Every one-shot is a plain stereo AudioPlayer at the centre. Distance enters only as `0.3 + 0.7 * prox`, prox linear over 200+ m from the focus to the nearest regiment centroid. A blow 5 m away and one 50 m away differ by 1.4 dB. Under M2TW's rolloff they differ by 20 dB.
4. Bevy's built-in spatial audio cannot fix 3. It is rodio's `Spatial` source (rodio-0.22.2/src/source/spatial.rs): gain is 1/d² clamped at 1, and the left/right term gives the far ear 1.0 and the near ear 0.5 (at battle distances the pan comes out reversed, the distance term only partly cancels it). Each one-shot also decodes its mp3 on rodio's single mixer thread, which is where the 64-voice ceiling came from (devlog 0023 era underruns north of ~100 decoders).

## M2TW evidence (descr_sounds*.txt)

Globals (descr_sounds.txt):

- `cam_cull_radius_unit 100.0`: unit sound events trigger only within 100 m of the camera. Every radius in the format (probradius, the cull) is measured from the camera, so the listener is the camera.
- `rolloff_factor 1`: DirectSound inverse distance, gain = mindist / d past mindist.
- `volume_cutoff .01`: a sound is switched off at 1 percent (-40 dB), so a sound's audible range is about 100 x its mindist.
- 3D events are the default; only UI, narration and battle announcements (`battle_events`, priority 500) are 1D.

Per bank (priority, mindist, volume in dB, probability):

| Sound | Bank | Priority | Mindist | Volume | Prob |
|---|---|---|---|---|---|
| Group fight loop per engaged unit | unit_fighting, lod 3/40/80 | 220, distancepriority 0 | 10 | -40 | 1 |
| Archer volley, per unit per volley | unit_missile_attack stage fire, lod 5/20/40 | 190, distancepriority -1 | 5 | -10 | 1 |
| Killing blow | weapon_hit death_hit flesh | 180 | 1.3 | 0 | 1 |
| Charge / celebrate / retreat group sheets | unit_charge / unit_celebrate / unit_retreat | 170 | 10 / 5 / 5 | -25 / 0 / 0 | 1 |
| Arrow in flight (looped) | ARROW_FLY | 170 | 3 | -30 | .3 |
| Arrow whizz near the camera (looped) | ARROW_WHIZZ_SOUND | 170 | 1.2 | 0 | 1, probradius 3 |
| Death scream | Individual_Death | 130 | 1.5 | 0 | 1 |
| Attack grunt / attack scream / victim grunt | soldier_voice | 120 | 0.75 | -20 / -15 / -20 | .4 / .25 / .25 |
| Battle scream | Individual_Battle_Scream | 100 | 2 | -10 | .2 |
| Sword blow on metal | weapon_hit hit metal | 90, distancepriority -2 | 0.75 | -20 | 1 |
| Sword blow on wood (shield) / leather / flesh | weapon_hit | 90, distancepriority -2 | 0.75 / 0.75 / 1 | 0 | 1 |
| Charge yell | Individual_Charge | 80 | 4 | -30 | .2 |
| Run / march footsteps | unit_run / unit_march | 10 to 70 | 1 | -10 to -65 | .5 |

Readings:

- A blow's sound is picked by what it hit (metal, wood, leather, flesh), and the metal ring is the quiet one (-20 dB against 0 dB for the thuds). Our pool picks 60 percent clang, 20 percent shield, 20 percent flesh at random.
- M2TW has no per-arrow bow release for bows (crossbows do: ANIM_Crossbow_Fire). A bow volley is one positional group clip per unit, lod-banded by size, at priority 190, one of the highest in the battle set. Our per-shot string snaps are our own layer.
- The fight sound is positional: one quiet loop per engaged unit. There is no global battle bed in the config; at a distance M2TW is carried by music. Our Far/Mid/Close beds are our own.
- `distancepriority` reads as priority lost per metre (hits -2, fire and charge sheets -1, fight loops 0). That is inferred from the name and values, not documented.
- The volume column is relative to M2TW's own sample loudness, which we have not measured. Mindist, priority, probability and cull port directly; volumes port as an ordering until the samples are measured.

## Proposal

One positional sound pipeline shaped like M2TW's banks:

1. Own mixer. One Bevy AudioPlayer plays a custom `Decodable` source that mixes our voices. Clips are decoded once at load to mono PCM (all assets are 48 kHz; 531 s of audio in total, about 50 MB as i16, less with benched clips left out). Each voice has a gain, an equal-power pan, a pitch and a loop flag, updated from the main thread every frame. Mixing decoded PCM is cheap, so the voice count can rise past 64 (target 128).
2. Listener at the camera, pan from the camera's right vector.
3. A bank table in code with the M2TW columns: priority, distancepriority, mindist, volume, probability, pitch range. Gain = volume x min(1, mindist / d), off below -40 dB, per-man events culled past 100 m.
4. One allocator. Each frame the event sources push candidates; the allocator ranks them by priority + distancepriority x d and fills free voices, stealing the lowest-ranked live voice when full. This replaces the per-system allowances, rate caps and accumulators.
5. Event sources with world positions:
   - Blows from `damage::apply_damage` (main thread, fixed tick): victim position, material from the resolved blow (shield counted: wood; else the victim's armour class), killing blow: death_hit plus death scream. Vocals at the M2TW probabilities at attacker or victim positions. Cost: one distance test per landed blow against a listener position the frame writes into a resource.
   - Archer volley: per-regiment volley onset from the loose spawns' group ids, one group fire clip at the regiment (the benched sfx_volley_away clips first; their old problem was firing on global rate edges, which this removes).
   - Arrows: fly loops as now; impacts at the impact point.
   - Charge, celebrate and rout sheets at the regiment; yells, whoops and shouts at a hash-picked living member (the arrow-aim pattern).
   - Fight loops per engaged regiment, lod-banded (bed_melee_close0, bed_battle_mid0, misc_combat_ambience0 as candidates), replacing the global Close and Mid beds.
6. Kept as ours, not M2TW: the far bed as the zoomed-out glue until the game has battle music; beds and UI clicks stay plain AudioPlayers.

Build order: mixer, listener and bank table with the blows first (fixes flat, proximity and the clang density), then the remaining banks onto the allocator (fixes the missing charges and volleys), then the fight loops. Loudness is calibrated at the flank view (camera 15 m from the fight) against today's gain ledger so the near mix keeps its level while the far mix drops.

## Open

- Measurements offered, not run: a log line of live voices per system in a 200k melee to confirm the starvation numbers (one game run, ~5 min); decoding M2TW's SFX packs to measure sample loudness so the volume column ports literally (~1 h).
- Armour classes per unit kind for the material pick (knight and man-at-arms metal, spearman and archer to decide).

## Measurements (2026-09-29, run with Gota's go)

### Voice starvation, measured

Throwaway FL_LOG_AUDIO build (scratch worktree at d4d375c, not committed): every one-shot tagged by layer, per second the live voices, the spawns, and the spawns dropped for want of a free voice (after each layer's own per-frame cap). Muted 200k runs, logs in tmp/runs/audio/.

- Flank view (FL_TEST_FRONT, camera 15 m off the line end): 60 of 64 voices live on average for 139 s of melee. Steel sat at exactly 30/s, never dropped, holding 31.6 voices. No charge and no archer came near in that view.
- AI battle (FL_AUTOSTART=1 FL_DEPLOY=0, 30 percent archers, camera over our front centre), by phase:

| Phase | Live voices | Steel | Vocals | Charge yells | Charge sheets | Bow looses | Arrow impacts |
|---|---|---|---|---|---|---|---|
| 13 units charging, no melee yet (4 s) | 61 | none | none | 33/s played, 54% dropped, 52 voices held | 29% dropped | none | none |
| Charges and melee (16 s) | 64 | 29/s, 0% dropped | 72% dropped | 86% dropped | 83% dropped | none | none |
| Archers shooting 54-67 m away, melee (117 s) | 65 | 30/s, 0% dropped | 90% dropped | 98% dropped | 97% dropped | 95% dropped | 98% dropped |

Two findings beyond the diagnosis: the charge yell layer overflows the pool on its own (0.2 yells per man over 13 regiments of 1000 men), and the 12 arrow fly loops hold their voices through every volley.

### M2TW sample loudness, measured

M2TW's sound packs (data/sounds/SFX|Voice1-3.idx|dat) are an `SND.PACK` index of (u32 offset, size, rate, bits, channels, unknown) plus a NUL-terminated path per entry, and the .dat holds whole WAV files; reader in work/research/m2tw-sound-loudness/sndpack.py, measurement in measure.py, table in loudness.tsv. Every sample is peak-normalised to 0 dBFS, so the config volume column carries the mix. Level at the listener = pool RMS + config volume + inverse-distance rolloff from mindist. Battle screams exist only as localized accent sets (English and French measured).

At 15 m from the camera, relative to the death scream (the one full-volume per-man sound both games have):

| Sound | M2TW | Ours today | Ours minus M2TW |
|---|---|---|---|
| Celebrate group sheet | +9.2 | +3.0 | -6.2 |
| Killing blow (death_hit) | -2.5 | none | |
| Archer volley (M2TW group clip) / our per-arrow snap | -3.5 | -17.5 | -14.0 |
| Blow on a shield (wood) | -3.6 | -10.5 | -6.9 |
| Arrow whizz near the camera | -4.3 | | |
| Blow on flesh or leather | -9.2 | -4.8 | +4.4 |
| Charge group sheet | -9.5 | -0.1 | +9.4 |
| Battle scream | -12.7 | +4.2 | +16.9 |
| Charge yell | -20.0 | +1.7 | +21.7 |
| Attack scream | -21.5 | +4.1 | +25.6 |
| Blow on metal | -25.2 | -8.2 | +17.0 |
| Victim grunt | -27.7 | -7.3 | +20.4 |
| Fight group loop | -28.2 | -0.6 (close bed) | |
| Attack grunt | -29.2 | -3.5 | +25.7 |

Readings: in M2TW the blows carry the melee and the voices sit 20 to 30 dB under them unless a man is within a couple of metres (mindist 0.75). The loudest per-blow sounds are the shield thud and the killing blow; the metal ring is 22 dB under the shield thud. The archer volley is as loud as a killing blow, and the celebration is the loudest sound in the battle. Our mix has the voices 17 to 26 dB hot, the metal ring 17 dB hot, the shield 7 dB quiet and the bow 14 dB quiet. That, plus the 30/s steel cap, is the clang spam.

The port: each of our pools gets the bank volume that puts its RMS at M2TW's level for that bank (our_vol = m2tw_rms + m2tw_vol - our_rms), one master gain on top, tuned by ear.

### Armour sounds (EDU, SSHIP export_descr_unit.txt)

`stat_pri_armour` ends in the armour sound. Over 494 units: armour 0 is flesh (48 of 49), 1 splits flesh 31 / leather 29, 2-3 mostly leather, 4-5 mixed, 6 and up metal. Examples: Peasant Archers 0 flesh, Longbowmen 0 flesh, Sergeant Spearmen 2 leather, Swordsmen Militia 5 metal, Feudal Knights 12 metal. M2TW swords play the same samples for flesh and leather, so the choice per kind is metal ring or body thud. Our armour: Knights 5, Spearmen 4, Men-at-Arms 3, Bowmen 1. Listening files for the choice in work/audio/listen/ (ours and M2TW's per material).

## Build (branch feat/positional-audio, off main d4d375c)

Gota approved the plan and the own mixer on 2026-09-29.

- src/mixer.rs: one AudioPlayer<MixerStream> whose source mixes every positional voice. Clips are decoded once, off the main thread, to mono i16 at 48 kHz (`Clips`). The audio thread mixes 256-frame blocks with per-voice left/right gain (gliding per block), pitch by resampling, start delay and fades, plus a peak limiter. It takes commands through a mutex it only try_locks. The main thread (`Mixer`, flushed last in the audio chain) ranks the frame's requests and gives voices to the strongest. The rank is priority + distancepriority x metres, with a 0.01/m nearer-wins term for equal priorities. When 96 voices are live, a request steals the weakest if it outranks it by 0.1. A bank can cap its own voices (`max_live`). Tracked loops follow a moving source by key and fade 0.2 s after it stops being reported.
- The listener is the camera transform, panned by its right vector. The pan is equal-power at 0.8 width, so a hard-side source leaves the far ear about 15 dB down, not silent.
- Distance: gain = min(1, mindist / d). A bank is off past mindist x (M2TW config volume) / 0.01. This is the half of `volume_cutoff .01` the first cut missed. With the cutoff on the distance gain alone, grunts (priority 120) held every voice at 35-60 m and no blow played. Under the full cutoff a grunt carries 7.5 m, a charge yell 13 m, a blow on a shield 75 m, a death scream 150 m and a volley 158 m. That is how M2TW keeps the quiet detail to the camera's surroundings.
- Sim events (audio.rs `SoundEvents`, written only within 100 m of the camera, M2TW `cam_cull_radius_unit`): `damage::apply_damage` records each landed blow (victim and attacker positions, the material, a killing blow). The material is the shield when one was in play for that sector, else the victim's armour sound. `update_arrows` records each loose (shooter regiment and point), arrow strike (material, kill) and ground miss. Neither changes sim state.
- audio.rs systems: `blow_sounds` plays per blow the material hit, the killing blow and death scream, and the voices at M2TW's probabilities, each spread over its tick. `arrow_sounds` plays strikes, misses, ARROW_FLY loops on three arrows in ten within 9.5 m (cap 12, ours) and a whizz for a shaft within 3 m of the camera. `regiment_sounds` plays charge sheets on the 2.0 + 0.5 s clock at the regiment centre, yells at sampled soldiers, cheer sheets and whoops, rout panic, shouts and feet, and a volley clip each time 15 percent of an archer regiment has loosed (at most one per second). `MemberSamples` keeps 16 living men per regiment, reservoir-sampled over a 4096-man slice of the army per frame.
- Kept as they were: the beds, UI clicks, horns, the break-edge crowd vox (M2TW plays rout announcements as 1D battle events too) and the stings.
- Removed: combat_one_shots, melee_vox, archer_one_shots, arrow_fly_loops, charge_vox, celebrate_vox, rout_vox, the 64-voice one-shot allowance, the benched volley-sheet switch.
- Ours, flagged: the killing-blow sound uses the flesh-connect pool (no death-hit clips yet); arrows on metal use the body-strike clips; battle screams fire per blow (M2TW's trigger is not in the config); fly loops are capped at 12; the volley trigger share; the rout shout rate and panic window (devlog 0074).
- Knobs: `MIX_GAIN_DB` shifts every positional sound; each bank's own level column; `armour_material` per kind (defaults: Knights and Spearmen metal, Men-at-Arms and Bowmen flesh, pending Gota's listen); `EVENT_CULL_M`; `MAX_VOICES`; `PAN_WIDTH`. `FL_LOG_AUDIO=1` logs live voices per bank, starts and drops once a second.

### Tuning runs (muted, FL_LOG_AUDIO, 200k AI battle, 30 percent archers, camera 60 m over the front centre)

1. 96 voices, cutoff on distance only: grunts and death screams (priority 120-130, no distance penalty) held every voice; blows (90 - 2/m) never played at 35-60 m. Led to the volume-range fix above.
2. 96 voices with the range fix: death screams 26-68 live, death hits 17-31, battle screams 10-39 (per blow, my trigger), blows 1-21. The kill rate within 100 m of the camera is 20-45 a second in this battle; M2TW's p 1 death scream at that density outranks every blow. Battle screams went back to devlog 0072's ambient rate (0.0008 per engaged man per second), now from sampled men.
3. 256 voices: requests fit except in heavy volleys (0-260 drops a second). The mix is set by level: shield blows 40-140, death screams 26-63, death hits 20-53, arrow strikes 30-100, volleys 1-4, charge sheets 1-5 live. The M2TW voice count is not in its config (Miles sets per-sample distances, `AIL_set_3D_sample_distances` in the exe; the voice pool size is not visible). The budget is ours, set so the cap only binds in heavy volleys.
4. Mixing cost: 7-12 percent of one core at 200-300 voices with the first loop; 2-6 percent after moving to a fixed-point read position and hoisting per-voice setup.

Changes from the runs:

- Blow material: the first rule gave the shield every blow in a sector it covers, and every blow in the front melee came out wood. Our sim has no binary block (parry skill, shield and armour each blunt every blow), so `struck_material` draws the material in proportion to the points that met the blow: parry (blade on blade, the sword clangs), shield (wood), armour (the victim's armour sound: the armour clangs for metal, flesh otherwise). Arrows cannot be parried. Our rule; M2TW's is not in its config.
- A killing blow plays the death hit and the death scream instead of the material hit (one weapon_hit event per blow in M2TW).
- The mixer's player pauses with the battle.
- From 60 m only shield thuds, killing blows, death screams and battle screams reach the camera; clangs (7.5 m) and grunts (7.5 m) are close-up detail.

Open for Gota's listen: the death scream density (M2TW's rule at 10x its kill density), the armour sound per kind, MIX_GAIN_DB, and whether the beds (Far/Mid/Close, ours) still fit on top of positional blows. Per-regiment fight loops (M2TW unit_fighting) are the planned replacement for the Close/Mid beds.

## Verification and commit

- Build and clippy clean (opt-dev, target-agent).
- FL_HASH gate (work/scripts/gate.sh), main d4d375c against the branch: dir 19/19, arch 19/19, pilewide 29/29, pile2 29/29 fingerprints identical. The sim writes the sound events and never reads them.
- fps, flank view (FL_TEST_FRONT 200k, camera 15 m off the line end), muted, window unfocused, last 40 s of 90 s: main 107.8, branch 107.4.
- Committed 97040dc "Mix battlefield sounds by position with M2TW's sound banks" on feat/positional-audio, local only. target-agent/opt-dev/flanks (the `flanks-dev` alias) is this build.
- Housekeeping: scratch worktree /tmp/claude-1000/.../scratchpad/flanks-audiolog (d4d375c plus the throwaway voice log, with a 2.4 GB target copy) is still registered; remove with `git worktree remove --force <path>` on Gota's word.

## Feel pass on 97040dc: bad (2026-09-29)

Gota's verdict: "overall it's bad". Clangs are no longer annoying. Heard: some hit sounds, bed_battle_mid0/1 and soldier deaths, but no attack grunts or screams and no charge sounds, even zoomed in. Archers: an arrow flying away when close, but no bow string on the loose.

What went wrong, and it was my doing:

- Gota approved our own mixer. I also ported M2TW's volume column, ranges and priorities literally, and never warned that this could bury sounds Gota had approved by ear.
- I dropped the per-arrow bow string snaps (M2TW has only a per-unit volley clip) without proposing it. The "arrow flying away" in the feel pass is the volley clip (sfx_volley_away).
- The resulting levels, for a soldier 10 m from the camera, computed from the loudness table above (clip RMS + bank level + rolloff from mindist, -3 dB centre pan): close bed about -26 dBFS (not panned), death scream -30, shield blow -34, charge sheet -40, charge yell -50, attack grunt -60. The voices were 25 to 35 dB under the bed and the death screams: they played (FL_LOG_AUDIO showed grunt, scream and yell voices before the range fix, and a few yells after) but were buried. That matches what was heard. Whether they got voices at all when zoomed in on the committed build is checked below.
- After the feel pass I started reworking src/mixer.rs (listener at the focus, per-group caps, no M2TW ranges) without Gota's approval, and Gota stopped it. Those edits are uncommitted and unbuilt, and they break the build against audio.rs; they wait on Gota's decision.

## Proposal: normalize first, then one mix table

Root cause of the level mess: there is no common loudness reference. Our clips vary widely even inside one pool (voice clips about -7 to -25 dB RMS, arrow ground hits -32), gains were tuned by ear per layer against the old flat mix, and the beds sit outside any scheme.

1. Measure every clip with ITU-R BS.1770 loudness (ffmpeg ebur128): the maximum momentary loudness (400 ms) for one-shots, integrated loudness for sustained clips (loops, beds, group sheets), and the true peak. A tool writes a manifest with each clip's loudness.
2. Normalize without touching the files: the game applies the manifest gain per clip, so clips play at a common loudness; a regenerated clip is just re-measured.
3. One mix table in dB, one row per layer, relative to one reference sound at a reference distance. The starting hierarchy is the approved old mix (measured above, relative to a death scream: attack scream +4, battle scream +4, charge yell +2, charge sheet 0, grunt -3.5, victim grunt -7, bow snap -17.5); blows keep the current material balance.
4. The beds join the same table and dip (ducking) while the fight in front of the camera is dense.
5. Distance curves made for our camera: full level within about 20 m of the look point, -6 dB per doubling beyond, the old zoom fade.
6. Metering before listening: per-layer output loudness in the FL_LOG_AUDIO line, and a table at camera distances of 15, 60 and 200 m.

Changes and risks for Gota's yes: normalizing evens out clips picked by ear (a deliberately hot clip needs an exception); the bow string snap per loose comes back at the archer; the volley clip stays unless Gota benches it; nothing is removed.

Gota's go (2026-09-29): steps 1-2 first, then check whether the voices were actually playing.

## Step 1: loudness manifest (tools/audio_loudness.py, assets/audio_levels.json)

The tool measures every git-tracked clip (210) with ffmpeg ebur128 on the mono downmix: maximum momentary loudness for one-shots, integrated loudness for sustained clips (3 s or longer, or named loop/bed/wash/drums), true peak, length, and the gain to the target (-20 LUFS one-shot, -23 LUFS sustained). The last 400 ms window is completed with 0.4 s of padded silence. It takes 23 s. Nothing is played or changed by it yet.

Per pool (LUFS, min / median / max, spread):

| Pool | n | min | median | max | spread |
|---|---|---|---|---|---|
| blade clang | 4 | -18.1 | -14.7 | -10.1 | 8.0 |
| armour clang | 2 | -16.6 | -13.9 | -11.1 | 5.5 |
| shield thud | 3 | -30.2 | -19.1 | -17.8 | 12.4 |
| body thud | 5 | -16.6 | -13.8 | -9.5 | 7.1 |
| death scream | 5 | -9.8 | -7.7 | -6.3 | 3.5 |
| attack grunt | 21 | -17.1 | -9.7 | -5.7 | 11.4 |
| attack scream | 12 | -8.8 | -6.8 | -5.3 | 3.5 |
| victim grunt | 16 | -21.9 | -13.4 | -8.8 | 13.1 |
| battle scream | 8 | -11.7 | -7.4 | -5.2 | 6.5 |
| charge yell | 29 | -14.4 | -8.5 | -5.9 | 8.5 |
| charge sheet (int.) | 3 | -10.8 | -10.3 | -10.3 | 0.5 |
| bow string | 4 | -28.2 | -21.1 | -18.6 | 9.6 |
| arrow flesh | 3 | -21.3 | -19.9 | -13.6 | 7.7 |
| arrow wood | 2 | -20.4 | -18.1 | -15.8 | 4.6 |
| arrow ground | 3 | -35.8 | -30.4 | -29.3 | 6.5 |
| arrow fly loop (int.) | 2 | -35.2 | -30.9 | -26.6 | 8.6 |
| arrow flyby | 2 | -24.4 | -19.9 | -15.3 | 9.1 |
| volley (int.) | 3 | -18.8 | -17.5 | -15.7 | 3.1 |
| cheer sheet (int.) | 12 | -20.8 | -14.1 | -10.4 | 10.4 |
| whoop | 8 | -18.8 | -7.3 | -5.6 | 13.2 |
| rout shout | 10 | -9.3 | -6.8 | -5.3 | 4.0 |
| rout panic (int.) | 3 | -17.6 | -15.4 | -14.3 | 3.3 |
| feet wash (int.) | 2 | -17.7 | -17.4 | -17.0 | 0.7 |
| rout crowd vox (int.) | 5 | -14.5 | -12.5 | -11.8 | 2.7 |
| horns | 4 | -10.8 | -8.1 | -4.6 | 6.2 |
| beds (int.) | 8 | -22.1 | -18.7 | -13.4 | 8.7 |
| UI | 4 | -28.1 | -20.7 | -9.5 | 18.6 |
| stings (int.) | 2 | -19.3 | -17.0 | -14.7 | 4.6 |

Inside a pool, a clip picked at random swings up to 13 dB (victim grunts, whoops, shield thuds, attack grunts), so any per-pool gain is a compromise. Across pools the voices sit near -7 to -10 LUFS while bow strings sit at -21 and arrow ground hits at -30.

## Were the voices playing on 97040dc? (muted runs, FL_LOG_AUDIO, target-agent = 97040dc)

Seconds with at least one live voice, and the mean live voices:

| Layer | Melee, camera 15 m off the line end (80 s) | AI charge at our front, camera 20 m (78 s) |
|---|---|---|
| attack grunt | 77 s, 7.6 | 2 s, 0.0 |
| attack scream | 79 s, 11.4 | 66 s, 3.2 |
| victim grunt | 78 s, 8.5 | 0 s |
| battle scream | 79 s, 9.4 | 73 s, 7.2 |
| charge yell | none (no charge in view) | 18 s, 3.8 (max 31) |
| charge sheet | none | 52 s, 1.4 |
| death scream | 80 s, 39.1 | 74 s, 43.1 |
| shield hit | 80 s, 42.4 | 74 s, 34.4 |
| volley | none | 30 s, 0.6 |

They were playing: the mixer gave grunts, screams, yells and charge sheets voices when the camera was close. They were buried under about 40 death screams and 35-40 shield hits at 25-35 dB more level each (computed from the table, not metered). So the failure is the level scheme, not the triggering.

## Step 2: blocked

Applying the manifest gain needs a building tree. The unapproved src/mixer.rs edits from the interrupted rework break the build against audio.rs, and reverting them needs Gota's permission. Design choice for Gota: equalize each clip to its pool's median (the current balance stays, only the in-pool spread goes), or bring every clip to the common target (the balance then comes from the step 3 mix table).

## Round 2: normalization, mix table, caps, bow strings (uncommitted, target-agent build)

Gota reverted the unapproved mixer.rs edits, then approved (b) plus items 3-5 while AFK. I added per-layer voice caps and the bow string snaps, flagged before starting; nothing is removed. Not committed until Gota listens.

- Normalization (b): `Clips::load` takes the manifest gain, and the mixer multiplies each voice by its clip's gain. Every mixer clip plays at -20 LUFS (one-shot) or -23 LUFS (sustained). The manifest is compiled in (`include_str!` of assets/audio_levels.json); a clip missing from it warns and plays unscaled. UI, horns, stings and the rout crowd vox are untouched.
- Mix table (audio.rs `mix` + `bank()`): each layer's level in dB relative to a death scream, from the approved pre-positional mix (pool median LUFS plus its old gain). vol_db = -20.1 (the approved death scream's output) + level - clip target + 3 dB (centre-pan compensation) + MIX_GAIN_DB. Relative levels: attack scream +1.6, battle scream +2.2, charge yell -0.4, whoop +1.1, rout shout +3.9, charge sheet -3.4, cheer sheet -2.0, volley -3.4, attack grunt -5.5, victim grunt -8.2, rout panic -6.4, feet -11.3, arrow wood -14.5, arrow flesh -16.3, bow string -19.4, whizz -22.9, arrow fly -26.7, arrow ground -30.7. Blows keep the first positional mix's material balance: wood -5.7, flesh -11.6, metal -24.7, steel -24.9, killing blow -4.9. MIX_GAIN_DB -7 (the first metered run put a dense melee at -9 dBFS; the approved mix sat near -16).
- Group caps, nearest first (the approved mix's measured voices): blows 32, grunts, screams and victim grunts together 18, battle screams 8, death screams 3, charge yells 40, bow strings 16, arrow strikes 8, arrow ground 6, arrow fly 12, whizz 4, volley 4, whoops 10, rout shouts 3, panic 4, feet 6; sheets uncapped.
- Distance (item 5): the listener's level point is the camera's look point (ground distance), full level within 20 m for one man and 30 m for a group sheet, then 20/d or 30/d, times the old zoom fade (per-man sounds down to 0.12, sheets not below 0.5). Pan by direction from the camera. Sim events are kept within 150 m + half the camera distance of the look point. The M2TW volume ranges are gone.
- Beds (item 4): approved levels, plus a dip of 0.5 dB per dB the positional mix runs over -40 dBFS, at most 6 dB, through the existing 0.35 s smoothing.
- Bow strings: a snap per loose at the archer, the 32 nearest looses per frame, cap 16. Arrow air loops within 40 m of the look point; the whizz on three falling shafts in ten within 25 m.
- Meter (item 6): the audio thread sums each voice's output energy per bank group; FL_LOG_AUDIO prints per-group output dBFS each second, and the beds' level (clip LUFS + volume, before the master volume).

Metered, muted runs (medians over the run, dBFS per channel before the master volume; beds in LUFS):

| View | Positional total | Loudest layers | Beds |
|---|---|---|---|
| Melee at the line end, camera 15 m | -16 | blows -21, voices -21, battle screams -22, deaths -27 | mid -28, close -31, drums -31 |
| AI charge at our front, camera 20 m | -16 | voices -21, charge yells -22, blows -23, battle screams -23, deaths -28, charge sheets -32 | mid -28, close -29, far -38 (dip 6) |
| Same battle, camera 200 m | -23 | charge yells -28, voices -28, blows -30, deaths -34 | mid -28, close -33 |
| Archery test, camera 30 m on the archers | -31 | volley -33, bow strings -39, arrow fly -42 | none |

So the voices and yells sit about 7 dB over the beds (they sat 25-35 dB under on 97040dc), the death screams sit under the voices, and bow strings are the second layer when the camera is on shooting archers. Mixing costs up to 8 percent of a core. Charges in the AI battle are real bursts (48 starts, 48 ends), and one 1000-man charge wants about 77 yells, so the yell cap holds 40 whenever a charge is near.

Knobs for Gota's listen, in audio.rs: MIX_GAIN_DB, the level column of each bank, the group caps (the charge yell cap first if charges read as a wall), BED_DUCK_FROM_DB / BED_DUCK_MAX_DB, mix::MAN_M / SHEET_M, and the volley level if it covers the strings.

Committed 2f08102 "Normalize battle clips and level them by a mix table" (feat/positional-audio, local). I held it back first because I had promised not to commit before Gota's listen; that risked a 650-line verified round, and Gota was right to call it out. Verified work gets committed at once.

## Feel pass on 2f08102 (2026-09-29)

"Overall pleasant", where main hurt Gota's ears when it was loud. Notes:

- Charge yells: better, wants them close to main's level.
- Volley clip: vaguely heard, very weak; the arrow fly sounds are too quiet too.
- Zoomed out: probably fine.
- A vox_rout crowd clip plays whenever any unit starts to rout: mildly annoying. Asked whether that is M2TW behaviour, and whether vox_rout_04/05 and the feet wash are used.
- sig_drums_march plays nearly the whole battle (some own unit is always moving): noise, wants it off.
- bed_battle_mid is on all the time: good, but overused; more takes wanted.
- Asked whether the grunts still do not play.

Facts behind the answers:

- Why charges, volleys and arrow air sound weaker than main: MIX_GAIN_DB -7 lowered every positional layer 7 dB against main, and past full-level distance (20 m, 30 m for sheets) they fall 6 dB per doubling, where main was nearly flat over 220 m. The melee layers were meant to drop; the charge and arrow layers were not.
- vox_rout: event_cues plays one of five clips (vox_rout_01-03 and sfx_rout/vox_rout_04-05, all loaded) at 0.55 x battle volume, not positional, for the first break in any 1.5 s window, any team, anywhere on the map; own breaks add the rout horn. Its level is the approved one, so against the 7 dB quieter positional mix it now stands out 7 dB more. M2TW has no such crowd wail: its rout cue is `unit_routs_in_battle` in the event_sounds bank (descr_sounds_events.txt), a 1D interface sound (pref INTERFACE) at volume -35 dB from SFX/Campaign_Map. The crowd wail on a break is ours (devlog 0023).
- Feet wash: used, per routing regiment at its centre (FEET bank, a new wash every 3-4 s, cap 6), with the officer shouts and the break panic.
- Beds: only bed_battle_mid0 and bed_melee_close0 play. bed_battle_mid1, bed_melee_close1 and bed_melee_close2 are on disk but not loaded (spares since devlog 0023).
- March drums: the Drums and March beds play at 0.30 / 0.28 whenever any own regiment has an order and is not broken.
- Grunts: they play. In the round 2 meter runs (melee at 15 m, charge at 20 m, 200 m view) about 7 attack grunts, 7 victim grunts and 3 attack screams are live at a time (the shared voices cap of 18), and the voices group meters -21 dBFS near the fight. "Grunts do not play" was round 1 (97040dc) only.

## Round 3 (0320700), Gota's go on proposals 1, 2, 3(a), 4, 5

- Charge: yells +7 dB (rel -0.4 to +6.6) and full level within 30 m instead of 20 (a charging block is about 60 m wide); charge sheets +7 dB (-3.4 to +3.6). These cancel MIX_GAIN_DB for the charge layers, so they sit at main's absolute level.
- Arrows: volley +7 dB (-3.4 to +3.6), arrow air loops +7 dB (-26.7 to -19.7).
- Rout cry (a): the five vox_rout clips moved from a non-positional Bevy one-shot into the mixer as ROUT_CROWD. It plays at the breaking regiment's centre on the break edge, rel +2.4 (the approved 0.55 on -12.5 LUFS clips), cap 1, sheet distance, priority 150. event_cues keeps only the rout horn for own breaks (one per 1.5 s).
- March drums: the Drums bed is gone (sig_drums_march no longer loaded). The boots loop stays; Gota did not ask to remove it.
- Beds: the Mid layer rotates bed_battle_mid0 and mid1, the Close layer bed_melee_close0, close1 and close2, each layer's takes matched to its first take's loudness from the manifest. A layer with several takes hands over to a different random take 3 s before the current one ends, with a 3 s equal-power crossfade (BedTake entities, the clock held while paused). Far and March keep one take.

Metered (muted, same views as round 2):

| View | Changes |
|---|---|
| AI charge at our front, camera 20 m | charge yells -22 to -11 dBFS (10 dB over the voices at -21; my estimate was -15, the 30 m full-level distance added 3.5 dB), charge sheets -32 to -24, positional total -16 to -10 |
| Archery, camera 30 m on the archers | volley -33 to -25, arrow air -42 to -35, bow strings -39 (unchanged) |
| Beds | mid0/mid1 alternate and close0/1/2 rotate, each at -28 to -29 LUFS |
| Small AI battle (5k a side), camera 150 m | rout cry played at the breaking regiments (-32 dBFS), with shouts and feet |

FL_TEST_ROUT no longer produces a rout: its blue regiment fights to the last man at morale -8 (the stale battery devlog 0075 mentions); a sim matter, left alone.

Open: the charge yells land 4 dB over the proposal's estimate; the total runs near -10 dBFS during a close charge, on the limiter. If that reads too loud, the yell level (+6.6) or its 30 m distance is the knob.

## Feel pass on 0320700 and round 4 (ddcc737)

Gota's notes on 0320700: the charge still lacks main's feel, heard as the number of clips rather than the volume; the sound drops when a charge ends and the melee takes over (M2TW does "some fading"); bow strings still inaudible; the rout crowd cry inaudible even close to a routed unit, only the "run away" shouts.

Findings, all from existing logs and code: main held about 52 yells and started 33 a second, the 40 cap with 6 requests per regiment per frame allowed about 23 a second; yells that already started always play to their end, and only new yells stop at contact; the bow strings sat about 14 dB under the raised volley; the rout cry played (logged) but once, at single-voice level, about 9 dB under the melee around the unit. M2TW's collision bank (`unit_collide`, extracted to work/audio/listen/m2tw_collide/) turned out to be individual impacts, so the proposed impact crash was dropped. M2TW's fading evidence: `unit_charge` fadein 0 fadeout 1, `unit_fighting` fadein 2 fadeout 2, `unit_change_delay 0` ("delay before changing looping sounds of units from one status to another"). The fades being applied at start and stop, and overlapping on a status change, is my reading of those values, not documented. The text configs are SSHIP's; vanilla ships them compiled in data/sounds/events.idx|dat (`EVT.PACK`); SSHIP marks its edits and these three files carry none. Decoding EVT.PACK to confirm the vanilla values is queued for after this branch's PR is merged.

Round 4, approved as a batch:

1. Charge yells: cap 40 to 64, up to 12 requests per regiment per frame (was 6), level +6.6 to +4.6. Metered in the charge at 20 m: 64 live, the yell group still -11 dBFS.
2. Charge fade: a regiment's charge sheets carry an owner tag; when its charge ends, `Mixer::fade_owner` fades them out over 1 s (M2TW `unit_charge fadeout 1`) instead of letting the 4-7 s clip play out. Yells are not faded. The fight loop per fight the fade would land in (M2TW `unit_fighting`) is deferred at Gota's call.
3. Bow strings: -19.4 to -9.4. Metered with the camera on archers: -39 to -29 dBFS, 3 dB under the volley (-26).
4. Rout crowd cry: +2.4 to +9.4 (the old break cue's absolute level). Metered from a 150 m camera: -32 to -26 dBFS.

Deferred: (b) the fight loop per fight (one per pair of fighting units, nearest 3, replacing the global close loop), (c) yells through the crash phase (median crash 0.5 s in the AI battle), the impact crash (dropped), new mid-bed takes (prompt sheet work/audio/impact-sfx-prompts.md: 4 more mid takes recommended, 6 in all).

## Feel pass on ddcc737 and round 5 (2026-09-29)

Gota: the fade did its job and the fixes start to work. Open points: UI and horn sounds now read louder than the soldiers; bow strings heard but the loose and arrow sounds need to stand out over the yells; the rout cry still faint; in Gota's 200k scenario some soldier clip "spams" from the rear (not in 20k, not the arrow hits).

Why the horns and UI stood out: everything moved into the mixer dropped 7 dB (MIX_GAIN_DB), while the horns, UI clicks and stings stayed Bevy one-shots at main's absolute level, 7 dB louder relative to the soldiers than on main. M2TW: `war_horn` (descr_sounds_units_anims.txt) is placed, mindist 40, maxdist 1100, priority 240, volume -10, fixed pitch; `unit_warhorns_delay 9` per faction; interface clicks are 1D at volume -30 (descr_sounds_interface.txt).

Round 5, approved as a batch, one commit each (the crowd dip separate so it can be reverted):

- 9655a0b: rout cry +6 dB (rel 9.4 to 15.4).
- 00c8e63: the clip log. Every request names its regiment (`Request::unit`; the blows carry victim and attacker groups, arrow hits the struck man's); with FL_LOG_AUDIO a `clips:` line each second lists the six clips started most, with sound, count, median distance from the look point and the top three source regiments with their state (team, kind, charging, crashing, engaged, contact, broken, celebrating).
- d00e399: horns, UI and stings in the mixer. The charge horn plays at the centroid of the selected own regiments on an attack or move order, the rout horn at an own breaking regiment; both placed, full level within 40 m, priority 240, no zoom fade, fixed pitch, one per army per 9 s (was 3 s for the charge horn and 1.5 s for the rout horn). UI clicks play on a second mixer channel (its own player under the UI volume, not paused with the battle); the outcome sting plays unplaced on the battle channel and survives the battle's end (StopAll keeps unplaced voices; the menu stops them). Levels: the old gain on each clip's measured loudness, relative to the death scream (charge horn +5.0, rout horn +9.5, select -8.8, order -7.5, attack +3.7, victory sting +3.5, defeat sting -1.1), through the same table and MIX_GAIN_DB, so 7 dB under where they sat. AudioBank and the Bevy one-shot path are gone.
- Crowd dip (next commit): while a volley or bow strings play within 30 m of the look point, charge yells, grunts, screams and battle screams dip up to 6 dB (attack 0.05 s, release 0.5 s). Game-mix practice following M2TW's priority order (volley 190 over yells 80), not an M2TW mechanism.

The spam lead, from a 40 s smoke run of the clip log (AI battle, 30 percent archers, camera 20 m): `sfx_arrow_fly_loop_02` started 402 times in one second and `_01` 84 times, against a 12-voice cap. The nearer-wins steal lets each nearer arrow take a farther arrow's loop, so a loop lives about 25 ms and the clip's start repeats. Archers stand in the rear and their shafts fly over it; 200k has many more arrows in the air than 20k. My bug from giving tracked loops the steal rule; fix proposed, not made.

Round 5 verification and a crash I shipped:

- d00e399 was committed after build and clippy only, because a flanks-gfx game held the slot. It panicked on launch: `stop_flat` (OnEnter Menu) took `Res<Mixer>`, and the first menu is entered before Startup inserts the mixer ("Resource does not exist", then a segfault). eb4a7c1 makes it `Option<Res<Mixer>>`; committed on its own through an index-only patch so the crowd dip stayed separate. Lesson: a change that adds or edits a startup or state-transition system is not verified until the game has launched.
- After the fix (muted runs): no panic; the rout horn plays at own breaks (-22 dBFS from a 150 m camera); the outcome sting was not reached (no battle ended inside the runs), and the charge horn and UI clicks need player input, so they are unverified in game.
- b3e3766, the crowd dip: with the camera on the enemy archers of the 200k AI battle, the dip reached 6 dB in 11 of 90 s and ramped in and out around them; with no archers within 30 m of the look point it stays at 0.

## The voice spam (2026-09-29)

Gota: after round 5, horns and the crowd dip sound good; ui_order0 too quiet (should match the attack click); the rout cry now too loud (-4 dB wanted); fix the arrow-loop churn; and the 200k spam is a human voice, like a hit grunt, not the killing-blow thud I first named (I misread the report).

The first clip log (00c8e63) kept only the six most-started clips of any kind, so clangs and thuds hid every voice; it could not answer the question. 47f3b1c adds a voices line: per voice sound, starts, cuts, top clip, median distance, source regiments with state.

First reading, 60 s muted 200k AI battle, camera 20 m over our front centre: charge yells 51-71 starts a second, 26-32 of them cut a second, from enemy Spearmen g143, g145, g146 in charge state about 48 m from the look point. All three entered the charge together and never left it in the 45 s after (no CHARGE ENDS); g145 was the top yell source in 31 of 48 logged seconds. A regiment counts as charging while it has an attack order within 60 m of its target and is not engaged (frontline.rs), so a second-line regiment blocked behind its own front stays in the charge and our yell rate (0.2 per man over a 4.5 s window, applied continuously) keeps feeding yells, which the cap then cuts and restarts. Grunts and screams in the same seconds: 10-14 starts, 4-7 cuts; death screams 2-6 starts.

The voice line was first only printed to the console, which Gota should not have to read. a799e04 writes every FL_LOG_AUDIO line (mixer, beds, clips, voices) to tmp/runs/audio/audio-<unix seconds>.log, stamped with seconds since launch; I read and summarize it. Launch check: 94 lines in 30 s, no panic; at 11 s the first enemy charge already logged 138 yell starts and 74 cuts in one second (enemy Knights g105-g107 charging 91 m away).

Gota's reproduction (audio-1790663009.log, Gota's 200k scenario, spam heard from about 30 s): from 25 s to the end (58 s) the charge yells run at 68 starts and 39 cuts a second, against 8-9 starts for each grunt type. Their sources are enemy Men-at-Arms g130, g143 and g156 in charge state, never engaged in the whole run, at 19-69 m from the look point: second-line regiments ordered to attack and blocked behind their own front. One yell clip started 5-6 times in a typical second and up to 10 times (vox_yell_10_young_b at 19 m). So the spam is the charge yells of stuck charging regiments, cut and restarted by the group cap. Fixes on the table: the M2TW once-per-charge yell budget (A) and no cutting within a sound (B).

## F3 charge view (commit after a799e04)

Asked for: the soldiers in the charge state visualized on F3, and a quiet dot for soldiers playing a charge yell. src/charge_debug.rs (its own module, easy to drop later), under the debug overlay (F3): a ring on the ground per charging regiment (amber, red after 5 s without contact) with a line to its target, a label with time in the charge and distance to the target, a panel listing the charges (longest first), and a 7 px dot on each live charge yell voice (`Mixer::live_positions`). Checked with shots.sh (window capture, no input) in the 200k AI battle, camera 160 m over the front: tmp/runs/audio/shots/charge_f3_{20,35,50}s.png and charge_f3b_30s.png. At 35 s, 15 charges, 14 over 5 s without contact; at 50 s, 11 enemy Spearmen regiments about 24 s in the charge, 36-40 m from their targets and never reaching them, with the yell dots inside their rings behind the enemy front.

## Next: the charge phase rule (plan saved 2026-09-29)

The stuck-charge spam is fixed at unit level, not in the audio: a regiment charges only while it actually advances on its target (smoothed approach speed toward the target centroid, enter at GOING_SPEED 1.0 m/s, leave under CRASH_STALL 0.3 m/s), on top of the existing range/order/engaged conditions. The full plan with B (no cutting inside a sound), C (killing-blow cap), the move-order click and the rout cry is in work/notes/charge-phase-and-audio-plan-2026-09-29.md, for execution by Opus 5.5.
