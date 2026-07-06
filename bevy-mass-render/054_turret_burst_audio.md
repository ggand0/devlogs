# 054: MG Turret Burst Audio

## Overview

Replaced per-shot MG turret audio with a single burst clip per firing cycle, synced to the firing state via a fade-out system. Fixes audio stacking with multiple turrets and keeps the clip's end aligned with the last bullet.

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

## Components / systems

| Item | File | Purpose |
|------|------|---------|
| `MgBurstAudio { turret, volume }` | `types.rs` | Tags a clip entity, links to its turret |
| `AudioFadeOut { remaining, duration }` | `types.rs` | One-way fade marker |
| `mg_burst_audio_sync_system` | `combat.rs` | Detects firing stop, starts fade |
| `audio_fade_out_system` | `combat.rs` | Applies fade via `AudioSink::set_volume`, despawns |
| `MG_BURST_AUDIO_FADE_SECS = 0.25` | `combat.rs` | Fade length, matches clip tail |

Registered in `main.rs` with `mg_burst_audio_sync_system.after(turret_hitscan_fire_system)` so the fade starts the same frame firing stops.

## Volume balancing

Per-burst clips changed the loudness math: N overlapping per-shot clips summed acoustically, a single burst clip doesn't. A fixed per-clip volume that sounds right for 5 turrets is too quiet for 1.

Fix: per-clip volume is normalized by concurrent bursts — `VOLUME_MG_TURRET / sqrt(active_bursts)`, counted at burst start. Perceived MG loudness stays roughly constant whether 1 or 5 turrets are firing (equal-power summing).

Turret explosion volume also bumped (0.3 → 0.45) so kills read through the gunfire without dwarfing other SFX (0.6 was tested and too loud).

## Gotchas learned

- **Measure audio duration by playing it** (`paplay` + timing), not metadata: ffprobe/mediainfo/wave-header math all reported 2.29s for `mg_1.wav`, but actual playback is ~3s+.
- Don't `try_despawn` playing audio entities as a control mechanism — hard cuts are always audible. Fade via `AudioSink::set_volume`, then despawn.
- `PlaybackSettings::DESPAWN` still handles natural end-of-clip cleanup; the fade system only intervenes when firing stops early. Stale `Entity` ids are safe — generational indices make `try_despawn`/query misses no-ops.
