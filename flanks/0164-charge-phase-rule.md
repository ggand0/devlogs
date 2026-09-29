# 0164: The charge phase rule, and no cutting inside a sound

Written by Claude Opus 5.5.

Follows devlog 0161 (positional battle audio). Plan: work/notes/charge-phase-and-audio-plan-2026-09-29.md (written by Fable 5.1, executed here). Branch feat/positional-audio, local.

## The charge phase rule (sim)

The problem: `charging` in frontline.rs was a per-tick predicate: attack order on a live target within CHARGE_RANGE 60 m, not engaged, not broken, not an archer with arrows. A regiment blocked behind its own front never engages, so it stayed in the charge for as long as it was blocked. It kept the 1.15x charge pace, the sprint pose and the charge cries. In Gota's 200k battle that gave 41 of 42 charges stuck over 5 s (resources/audio/debug0.png) and charge yells at 72 starts and 54 cuts a second (audio-1790663996.log): the voice spam.

The rule: a charge is a run at the enemy. On top of the old conditions, a regiment must be closing on its target:

- `approach_speed` (new GroupData field, orders.rs): the regiment centroid's velocity toward its target's centroid, smoothed like `adv_speed` (EMA 0.1 per tick). The centroid velocity is the mean of the men's velocities, so it reads "the average man is running at the enemy". It is measured toward the target, not along the facing, because the facing is set once when the order is given.
- A regiment enters the charge at `GOING_SPEED` 1.0 m/s (the sim's existing "moving with purpose" threshold, now pub(crate) in sim/soldier.rs) and leaves it under `CRASH_STALL` 0.3 m/s (the existing stall threshold of the crash). The two thresholds keep a creeping regiment from flickering in and out, where each flicker would restart the group sheet and the yells.
- The crash start (charging last tick, engaged now) is unchanged.

Verification:

- Build and clippy clean.
- FL_TEST_CHARGE (spearwall acceptance): charge, crash and charge-end timings identical to e389d40, and FL_HASH fingerprints 48 of 48 identical. The charge test's chargers run at their targets the whole way, so the rule never bites there.
- 200k AI battle, F3 view, camera 160 m over the front (tmp/runs/audio/shots/charge_rule_{20,35,50}s.png). At 35 s: 12 charges, 11 of them over 5 s, enemy Spearmen g140-g150 at 8.5-8.9 s in the charge and 41-42 m from their targets. They had come from 60 m in about 9 s, about 2 m/s, so they were still advancing and the rule rightly keeps them charging. At 50 s: no charges, no yells playing. The old build at 50 s had 11 charges about 24 s long that closed 2 m in 15 s.
- Charge lengths from the CHARGES / CHARGE ENDS log lines of the same battle: old build 32 charges ended (median 11.2 s, 90th percentile 21.1 s, longest 25.0 s) and 9 still charging when the run ended; new rule 39 ended (median 9.1 s, 90th percentile 10.5 s, longest 12.2 s) and none left charging.
- Charge yells per second from the audio log (audio-1790666362.log, same battle): 53, 71 and 42 starts a second in the 10-20, 20-30 and 30-40 s windows while the second lines advance, then 0 from 40 s to the end. The old build in the same recipe (audio-1790663779.log) held 69-84 starts a second to the end of the run. The cuts that remain while the lines advance (21-39 a second) are the group cap cutting its own voices, which B below removes.
- FL_HASH gate (dir, arch, pilewide, pile2) against e389d40: all four identical (tmp/runs/scripts/gates/cr-{old,new}-*.hash). None of them has a blocked attack order.
- fps, flank view (FL_TEST_FRONT 200k, camera 15 m off the line end), muted, mean of the last 40 s of 90 s: e389d40 122.0, charge rule 121.5 (noise).
- Committed as de38d23 "Charge only while running at the target".

## B: no cutting inside a sound (mixer)

The rule, as planned: a full group drops new requests instead of cutting one of its own playing voices; the nearest of a frame's requests take the voices that free up. At the global voice limit (256) a sound can still take the weakest voice of another group if it outranks it by STEAL_MARGIN. A group that fails at the limit is closed for the rest of the frame (the old single `pool_closed` flag was wrong once the victim must come from another group).

A bug found on the way: arrow air loops kept restarting about 440 times a second even with the rule in (tmp/runs/audio/audio-1790667387.log). The tracked-loop matching in `flush` sorted the frame's loop reports by key, binary-searched each live loop's key, and marked a found report by overwriting its key with u64::MAX. The overwritten key broke the sort order, so later searches missed reports that were there: those loops were faded as gone, and their reports started new voices. Devlog 0161 blamed the churn on nearer arrows stealing farther arrows' loops; that was part of it, the search bug the larger part. Fixed by marking reports in a separate `taken` list.

Runs: 200k AI battle, 30 percent archers, muted, camera over the front centre at 20 m and 160 m, 70 s, the old mixer (de38d23 build) against the new one (8c4175b build):

| per second | old, camera 20 m | new, camera 20 m | new, camera 160 m |
|---|---|---|---|
| voice cuts (all human voices), charge phase 10-40 s | 34-53 | 0 | 0 |
| charge yell starts, 10-40 s | 43-70 | 22-36 | 23-36 |
| charge yell voices live, 18-35 s | 63-64 | 64 | |
| arrow air loop starts, 40-70 s | 369-411 | 21-22 | 20-21 |
| all sounds started, 40-70 s | 512-560 | 107-112 | 108-115 |

The cost the plan did not list: with no cuts, the 64 charge yell voices go to whichever yells started first, and most charging men are far from the look point. The median distance of the yells started rose from 46 m to 74 m (18-35 s), and the charge yell layer's output fell from -9.7 to -13.4 dBFS (mean of 17 s), 3.7 dB, which is what the distance alone predicts (30/46 against 30/74 is 3.9 dB). In the archery phase the arrow air layer fell 3 dB and bow strings 1 dB; the blows and voices read 2 dB lower too, which may be run-to-run spread (the AI battle is not deterministic). Committed as 8c4175b as planned; the charge yell loss is reported to Gota with a proposal before the feel pass.

## Move-order click and rout cry (1fe6d2d)

UI_ORDER rel -7.5 to 3.7 (the attack click's level; both clips are normalized, so the same rel is the same loudness). ROUT_CROWD rel 15.4 to 11.4 (Gota: +6 dB was too loud, +2 over the earlier level wanted). Launch check, 5000-man AI battle, camera 150 m, 200 s: no panic, the rout cry played at the first break (116 s, own Spearmen g2, 38 m, -21 to -23 dBFS). The move-order click plays only on a player order and was not triggered (no synthetic input).

## Open: the charge yells lost 3.7 dB to B

Proposed to Gota, not built: let a sound cut a voice of its own group only when the new one is at least twice as near (6 dB louder under mindist / d), so the cut voice sits well under its replacement and the 30 ms fade is covered, while yells at similar distances never trade voices. Alternative: request yells only from the charging regiments nearest the look point, enough to fill the 64 voices, with farther regiments carried by their charge sheets.

## After Gota's play of 1fe6d2d (2026-09-29)

Gota: the charge spam is gone, but the 200k battle lost its atmosphere (the stuck rear units' yells had carried it), the move-order click is now too loud, and some hits play a swish like a sword swing. Decisions:

- Atmosphere: regenerate `bed_melee_close0-2` (Gota generates; prompt sheet work/audio/fight-loop-prompts.md). Measured against M2TW's fight loops (Group_Fight_Small/Medium/Large, extracted with the other atmosphere samples to work/audio/listen/atmosphere/): M2TW's are 92-95 % of their energy at 300 Hz-2 kHz (voices and body), ours 5-36 % there and 45-62 % at 2-8 kHz plus 14-32 % above 8 kHz (bright steel and hiss). The sheet targets M2TW's balance. In M2TW's text configs and sample packs, soldiers waiting behind the front play no voices (the group taunt bank is commented out and vanilla has no group taunt samples), only quiet fidgets and shield bashes within 1-2 m.
- defbce9: move-order click rel 3.7 to -1.9 (5.6 dB under the attack click); `sfx_spear_damage_01` out of the flesh-hit set (it plays on flesh hits, killing blows and killing arrow hits). Launch check: 5000-man battle, 90 s, no panic, the four remaining clips play.
- Vanilla's compiled sound events (data/sounds/events.idx|dat) go to another agent: work/handoffs/HANDOFF-m2tw-sound-events-decode-2026-09-29.md.

## Close melee beds: composites from Gota's ElevenLabs layers (2026-09-29)

Parked: my own fight loops built from the game's one-shots (work/audio/composite/fight_loops.py, outputs in work/audio/listen/atmosphere/composite/; they matched M2TW's measured balance but Gota did not like them) and the bed prompt sheet (work/audio/fight-loop-prompts.md).

Gota generated layers on ElevenLabs into assets_dev/sfx_dev/bed_close/: 7 voice takes (`voice/bed_melee_close0..6_voice*`), 7 shield/armor clatter takes (`clatter/bed_melee_close1,2,3,4,6,7,9_clatter_*`, tagged few-meter, 5m or 10m), 6 sword clatter takes (`clatter/bed_melee_close_swordclatter0..5*`), plus 2 optional clatter takes and 3 voice+clatter takes, not used. All 14.9 s, stereo, 48 kHz. Their loudness ranges from -10.1 to -20.7 LUFS and several clatter takes peak above 0 dBFS (up to +3.2 dBFS true peak).

Method (work/audio/composite/close_beds.py, seeded): decode each layer to float (no clipping on decode), level each to the same loudness, then voice as the lead, shield/armor clatter 5 dB under it, sword clatter 7 dB under it; mix to -16 LUFS with a peak limiter at -1 dBFS, 20 ms fades, mp3 192k. Sources are read only. Batch 1 (work/audio/listen/bed_close_composite/): Gota's four picks (shield/armor clatter + voice: 3+1, 7+0, 6+5, 1+2) with a random sword take, then 20 random other pairs.

Gota's ratings of batch 1 (05 not rated):

| rating | composites |
|---|---|
| very good | 08 voice0+close2 few-meter+sword5, 11 voice6+close9 few-meter+sword4, 15 voice6+close1 few-meter+sword1, 16 voice1+close6 5m+sword0 |
| good | 01 voice1+close3 5m+sword2, 02 voice0+close7 5m+sword1, 04 voice2+close1 few-meter+sword5, 12 voice1+close7 5m+sword4, 13 voice0+close3 5m+sword3, 14 voice3+close1 few-meter+sword0, 19 voice1+close2 few-meter+sword2, 22 voice5+close7 5m+sword4, 23 voice1+close1 few-meter+sword0 |
| fine | 09 voice4+close4 10m+sword4, 21 voice5+close3 5m+sword1, 24 voice6+close3 5m+sword4 |
| ok | 03 voice5+close6 5m+sword3, 06 voice5+close1 few-meter+sword1, 07 voice3+close7 5m+sword5, 10 voice6+close7 5m+sword0 |
| meh | 17 voice0+close6 5m+sword4, 18 voice5+close9 few-meter+sword1, 20 voice4+close7 5m+sword3 |

Patterns (scores very good 5 to meh 1; samples are small, one rating per composite):

- Few-meter clatter beats 5 m clatter: close1, 2 and 9 average 3.8, close3, 6 and 7 average 3.0; 3 of the 4 very good use few-meter clatter.
- voice1 was good or very good in all 5 of its composites; voice5 averaged 2.4 across 5 different clatters. Voices 2, 3 and 4 had only 1-3 composites each, so the batch cannot rank the voices (I first claimed the voice mattered most; Gota pointed out the unbalanced sampling).
- The sword take shows no consistent effect (averages 2.3-4.0 on 2-6 samples), as Gota expected.
- The meh ones are pairs that fail together, not bad parts: voice0 is very good in 08 and meh in 17; close9 is very good in 11 and meh in 18.

Batch 2 (work/audio/listen/bed_close_composite2/, 25-34): the undersampled voices, voice2 x4, voice3 x3, voice4 x3, each with a random shield/armor take not yet paired with it and a random sword take. Awaiting Gota's ratings.

Gota's ratings of batch 2: good 25 (voice2+close6 5m+sword4), 26 (voice2+close7 5m+sword4), 27 (voice2+close9 few-meter+sword1); fine 29 (voice3+close9+sword4), 30 (voice3+close6+sword1), 33 (voice4+close3+sword4); ok 31 (voice3+close4 10m+sword0), 32 (voice4+close9+sword0), 34 (voice4+close2+sword5). voice2 is good with every clatter tried; voice4 never rose above fine.

## The new close melee layer (7a4458a)

Chosen: one composite per voice take, since the voice leads the mix and two composites sharing a voice take sound like the same recording: 08 (voice0), 16 (voice1), 27 (voice2), 14 (voice3), 22 (voice5), 11 (voice6); voice4 left out (best: fine). Gota listened to the six (work/audio/listen/bed_close_selected/) and approved them for the game.

In the game they are `bed_melee_close0..5.mp3` (the old three replaced; they stay in git history and in work/audio/listen/atmosphere/ours/), and the close layer's take list grows from 3 to 6, so a take returns about every 72 s instead of 36 s. Level: the game takes a bed layer's level from take 0's file and plays beds in stereo, so the six were rendered (close_beds.py `select`) to the old close0's stereo loudness, -14.1 LUFS; the limiter left them at -14.3 to -14.6 LUFS, true peak -1.1 to -1.2 dBFS. The manifest measures them at -17.8 to -18.2 LUFS mono, so their take gains stay within 0.4 dB. Launch check: 5000-man battle, camera 30 m, 120 s, no panic, all six takes played in rotation.

## Close loop heard, then placed per regiment (6cd0b28, 7df4002, 7659cc9)

- 6cd0b28: `FL_BED_MUTE=mid,far` silences named bed layers, to hear the others alone.
- Gota could not hear the new close takes even zoomed in. Cause: the close layer ran 0.60 x proximity squared x hit rate x zoom, and dipped up to 6 dB under a loud fight at the look point; the old steel-led takes cut through at that level, the new voice-led ones sat under the single grunts and screams. 7df4002 raised it 4.4 dB and took it out of the dip. Gota: now audible, but not placed, global.
- M2TW (vanilla, devlog 0165 decoded events.dat and confirmed SSHIP): `unit_fighting` is one placed loop per fighting unit (Small / Medium / Large at 3 / 40 / 80 men fighting, -40 dB, mindist 10, fade 2 s in and out, priority 220 with distance priority 0, pitch 0.9-1.1), and unit sounds are culled beyond `cam_cull_radius_unit` 100 m from the camera. No global battle bed exists in M2TW.
- 7659cc9, with Gota's go on "M2TW's way": each engaged regiment within 100 m of the camera plays a tracked fight loop at its centroid (one of the six takes by its index, pitch by a hash of its index), priority 220, fading in and out over 2 s (new Bank `fade_in` / `fade_out`, a fade-in envelope on voice start, per-bank fade-out of tracked loops). The Close bed layer is gone. Our approximations: engaged (one man fighting, TW rule) stands for M2TW's 3-men threshold, which a 1000-man regiment passes at once; one size of take instead of three; the level is our mix table's, not M2TW's -40 dB. The six takes were re-rendered as seamless loops (the last 1 s crossfaded into the start, 13.9 s; rodio decodes mp3 gapless, the decoded length matched the render exactly).
- Level check, 5000-man battle, camera 20 m over the centre, 120 s, 7df4002 against 7659cc9 at rel 0: the old close bed ran about -27 LUFS in the main fight (20-80 s); the new loops, 4-6 of them, about -25.5 to -26.5 dBFS on the mixer meter. Kept rel 0 (within about 1 dB, the loops a little louder). The old bed fell to -35 as the hit rate dropped late in the run; the loops hold while the regiments fight. No panic in either run.
- Gota judged the loops with `FL_SOLDIER_SFX=0 FL_BED_MUTE=mid,far` (c1a0518 adds FL_SOLDIER_SFX, the share of melee soldier sounds that play): the fight sound is acceptable on the loops alone. Reference levels in the same view (camera 20 m over a fight): original close bed about -37 (not heard), 7df4002 about -27, 7659cc9 fight loops at rel 0 about -26 (the debugging level). Set to rel -4, about -30, on Gota's word.
- f4de8fc: `FL_BED_MUTE=close` silences the fight loops. 22b3629: fight loops rel -5, about -31 (Gota's pick). 290525b: the far bed was not heard at all (0.22 x engaged share x up to 6 dB dip, logged near -40 against the loops' -26 to -30); its multiplier 0.22 to 0.70, +10 dB, for Gota to judge.

## Taunt voices, text to speech through the ElevenLabs API (2026-09-29)

Why: vanilla M2TW's waiting soldiers shout taunts (`Individual_Taunt` on taunt animations, devlog 0165). Our game plays no taunts yet; the trigger is not built (M2TW's taunt animation has no counterpart here).

Tooling: `work/scripts/elevenlabs.py` (key in `.envrc`, loaded per command and never printed): transcribe (Scribe), speak (TTS), sfx, voices. Scribe transcribed M2TW's English taunts (six lines, each in three class voices; work/audio/listen/atmosphere/m2tw/17_taunts_english/transcripts.md) and five Mordhau voice videos as a style reference (work/audio/listen/mordhau_ref/, local only, not material for the game).

Research: `eleven_v4` follows bracket direction best, low stability is the most expressive, capitals add emphasis, and narration voices resist shouting. On the same lines, v3 with each voice's default settings against v4 with stability 0 and the prefix `[shouting at the top of his lungs, furious]`: voice pitch rose about 1.5x (a sign of real shouting); Gota: v4 "much better, sounds more like yelling". Scribe read every generated line back word for word.

Rounds (clips in work/audio/listen/tts_voices/): 13 voices on v3 defaults, the same 13 on v4, 9 regional British voices found after Gota asked for something like Mordhau's "Young" voice, round 2 (9 picks, six lines), round 3 (6 voices, the final lines). Rejected: American accents (Jerry B., Brock, Harry, Jack; one accent per army, as in M2TW), a narrator (John of the North), voices that do not carry (Northern Terry, Jack).

Final lines, ours (checked against 450 transcribed Mordhau lines; two of our earlier lines came out close and were dropped): "Over HERE, you dogs!", "I can smell your FEAR!", "I'll split your SKULL!", "Come closer, COWARDS!", "Beg for your LIFE!".

Chosen voices (Gota's picks and notes):

| voice | voice_id | note |
|---|---|---|
| Sam | DikmR0aoFXAp1A3NcovW | the best: very good at yelling (young, slightly Welsh) |
| Viking Bjorn | ljo9gAlSqKOvF6D8sOsX | the next best |
| Cassius | ktrGUw7rURIQyMrQZqCu | sounds similar to Gideon |
| Gideon | q1h5HGdnfVxp4TXTJRNN | sounds similar to Cassius |
| Scotty Marshal | NfUrCNRReUL9RXS9upG1 | for variety (Scottish) |
| Andy | kVBPcEMsUF1nsAO1oNWw | kept for the general and unit orders |

Generation settings for the final clips: `eleven_v4`, stability 0.0, the prefix above, the stressed word in capitals. Open: the taunt trigger, and which voice goes with which unit kind.
