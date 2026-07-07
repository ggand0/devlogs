# 054: MG Turret Burst Audio & Combat Mix Balancing

## Overview

Replaced per-shot MG turret audio with a single burst clip per firing cycle, synced to the firing state via a fade-out system. Then fixed the two mix problems that surfaced from it: single-turret loudness (equal-power normalization) and inaudible explosions during firefights (asset choice + sidechain-style ducking).

## Problem

MG turrets fired 20 shots/sec, each spawning a `mg_3_single.wav` audio entity (throttled to 3/frame). With 5 turrets on the defense scenario (map 3):

- Up to 15 overlapping single-shot clips at any moment — harsh, fatiguing stacking, unlike the infantry laser sounds which stack fine
- Explosion sounds got buried under the wall of gunfire
- CoH1-style MG42 audio needs a continuous rattle, not stacked pops

## Approach: one clip per burst

`mg_1.wav` (a full MG42-style burst recording, ~3s audible with a 0.25s fade-out baked into its tail) plays once when a burst starts (`shots_in_burst == 1`). One audio stream per turret instead of 20/sec.

### Failed attempt 1: tune shot count to clip length

Reduced `max_burst_shots` 45 → 40 so the burst (~2.0s) would end inside the clip's fade-out. **This class of fix cannot work:** burst duration is framerate-dependent (the fire timer quantizes to frames) and bursts get interrupted early by target loss or LOS blocks. A fixed-length clip never matches a variable-length burst.

### Failed attempt 2: force-despawn the audio entity

Tracked the audio entity on `MgTurret` and `try_despawn`ed it when firing stopped (cooldown start / target loss). This hard-cut every clip mid-waveform — chopped tails, thin choppy mix, and the overall soundscape sounded broken/quiet. Reverted.

### Final architecture: fade-out synced to firing state

```
turret_hitscan_fire_system      spawns clip tagged MgBurstAudio { turret, volume }
        │
mg_burst_audio_sync_system      every frame: is the linked turret still firing?
        │                       (target present, not cooling down, burst in progress,
        │                        turret still alive)
        │  no → insert AudioFadeOut { remaining: 0.25, duration: 0.25 }
        │
audio_fade_out_system           lerps AudioSink volume to 0 over 0.25s, despawns
```

Key properties:

- **The clip always ends 0.25s after the last bullet**, regardless of clip length, framerate, or how the burst ended (cooldown, target died, LOS blocked, turret destroyed)
- **No hard cuts** — 0.25s fade matches the fade baked into the clip's own tail, so an early stop sounds like the natural clip ending
- **One-way fade**: `mg_burst_audio_sync_system` filters `Without<AudioFadeOut>` so the fade timer is never reset once started
- **Seamless target switching**: continuous mode reacquires targets inside the fire system, so the sync system sees an uninterrupted burst and the audio never restarts mid-mow
- `shots_in_burst` resets on every interruption path (idle, LOS block) so the next burst reliably retriggers its clip

## Volume balancing 1: single turret vs stacked (equal-power normalization)

Per-burst clips changed the loudness math: N overlapping per-shot clips summed acoustically, a single burst clip doesn't. A fixed per-clip volume of 0.125 sounded right for 5 turrets but thin for 1 — and no single constant can satisfy both.

Fix: per-clip volume = `VOLUME_MG_TURRET / sqrt(active_bursts)`, where `active_bursts` is the number of MG turrets mid-burst, counted (from last frame's state) at the top of the fire system. Equal-power summing keeps perceived MG loudness roughly constant at any turret count:

- 1 turret → 0.25 per clip
- 5 turrets → 0.25/√5 ≈ 0.11 per clip (≈ the previously approved stacked level)

## Volume balancing 2: explosions (the masking problem)

Turret death explosions were inaudible during firefights. Three rounds of "bump the constant" (0.3 → 0.45 → 0.6) all sounded **identical** — the key debugging clue. Two measured root causes:

### a) The asset was quiet at the source

`ffmpeg -af volumedetect` on the actual sample data:

| clip | mean | peak |
|------|------|------|
| `mg_1.wav` (MG burst) | −12.4 dB | −5.0 dB |
| `distant_explosion1.wav` ← turret death sound | **−19.8 dB** | −8.0 dB |
| `ground_explosion0/1.wav` | −15.2 dB | −3.5 dB |

The death sound was a *"distant"* explosion recording — pre-muffled by design, 7.4 dB (~2.3×) below the MG clip. Compensating via the constant would need a value >1.0. **Fix: turret deaths now play the punchy `ground_explosion0/1` clips (randomized) at 0.5.**

### b) Masking — one clip can't out-shout the gunfire bed

During a defense wave, infantry spawn up to 5 laser clips per frame (≈300/sec at base 0.3) plus MG bursts. The bed sums far past full scale; a single explosion voice a few dB hotter is perceptually invisible inside it. This is why constant-tuning changed nothing.

**Fix: sidechain-style ducking** (the CoH approach — big moments push the rest of the mix down):

```
turret_death_system            sets ExplosionDucking.timer = 0.7s
        │
explosion_ducking_system       while timer > 0: every GunfireAudio sink
                               plays at volume × 0.25; over the last 0.3s
                               the factor ramps back to 1.0 (no pop)
```

Every gunfire audio entity — infantry lasers, heavy turret shots, MG bursts — is tagged `GunfireAudio { volume }` at spawn. Fading MG clips (`With<AudioFadeOut>`) are excluded; they're already dying. The explosion fires into the dip and owns the mix for ~half a second.

## Components / systems / constants

| Item | File | Purpose |
|------|------|---------|
| `MgBurstAudio { turret, volume }` | `types.rs` | Tags a burst clip, links to its turret |
| `GunfireAudio { volume }` | `types.rs` | Tags all gunfire clips for ducking |
| `AudioFadeOut { remaining, duration }` | `types.rs` | One-way fade marker |
| `ExplosionDucking { timer }` | `types.rs` | Resource; >0 means gunfire is ducked |
| `mg_burst_audio_sync_system` | `combat.rs` | Detects firing stop, starts fade |
| `audio_fade_out_system` | `combat.rs` | Applies fade via `AudioSink::set_volume`, despawns |
| `explosion_ducking_system` | `combat.rs` | Dips/restores the gunfire bed |
| `MG_BURST_AUDIO_FADE_SECS = 0.25` | `combat.rs` | Fade length, matches clip tail |
| `VOLUME_MG_TURRET = 0.25` | `constants.rs` | Target MG loudness before √n scaling |
| `VOLUME_TURRET_EXPLOSION = 0.5` | `constants.rs` | Over a ducked bed |
| `EXPLOSION_DUCK_FACTOR = 0.25` | `constants.rs` | Gunfire multiplier while ducked |
| `EXPLOSION_DUCK_DURATION = 0.7` | `constants.rs` | Duck time in seconds |
| `EXPLOSION_DUCK_RELEASE = 0.3` | `constants.rs` | Ramp-back window |

Registered in `main.rs`: `mg_burst_audio_sync_system.after(turret_hitscan_fire_system)`, plus `audio_fade_out_system` and `explosion_ducking_system`; `ExplosionDucking` inserted as a resource.

## Gotchas learned

- **Measure audio duration by playing it** (`paplay` + timing), not metadata: ffprobe/mediainfo/wave-header math all reported 2.29s for `mg_1.wav`, but actual playback is ~3s+.
- **When repeated volume bumps sound identical, stop tuning** — it's masking or a dead code path, not level. Measure the assets (`ffmpeg -af volumedetect`) and count concurrent voices.
- Don't `try_despawn` playing audio entities as a control mechanism — hard cuts are always audible. Fade via `AudioSink::set_volume`, then despawn.
- `PlaybackSettings::DESPAWN` still handles natural end-of-clip cleanup; the fade system only intervenes when firing stops early. Stale `Entity` ids are safe — generational indices make `try_despawn`/query misses no-ops.
- Both hitscan fire systems sit at Bevy's 16-system-param limit. Tag audio entities with marker components and process them in a separate small system instead of threading new resources through the big ones.

## Possible follow-ups

- Trigger ducking from tower explosions and artillery too (currently turret deaths only)
- Scale `active_bursts` continuously instead of at burst start (first clip of a stack plays slightly louder than later ones)
