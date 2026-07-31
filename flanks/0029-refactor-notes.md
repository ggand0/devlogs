# 0029 — Pre-PR refactor: morale module + landmine docs (2026-07-13)

Commit `4cc190f` on `feat/battle-feel`. Owner: do the important
refactors on this branch, defer pure code motion to a refactor branch
(clippy sweep there too — "there'd probably be too many warns").

## Done here

- **`src/morale.rs`**: morale (constants, `MoraleFactors`,
  `MoraleReadout`, `update_morale`, now `MoralePlugin`) extracted from
  regiments.rs — the most-tuned system in the game no longer lives
  inside spawn code. Pure motion; regression-checked via the rout-test
  break point (~50 % for the enveloped light) and cheer triggers.
- **`units.target` doc rewrite** (units.rs): it is a combat MEMO —
  swing target while a swing is in flight, sparse-closing memo
  otherwise — never cleared, silently reindexed by death-sweep
  swap-removes. Every consumer must validate on use. This field caused
  the false-engagement bug in 0024; the doc now says so loudly.
- **Anim z-channel map** (render_units.rs at `CELEBRATE_BASE`): the one
  authoritative encoding table — positive = style*2 + wind-up progress
  or CELEBRATE_BASE + cheer progress; negative = smoothed stance tiers
  (0.25 / 0.5 / 0.65 / 1.0). The shader decode mirrors it.

## Deferred to the refactor branch

1. orders.rs split (selection / input / orders proper — six concerns,
   ~800 lines; formations will grow it further).
2. audio.rs `event_cues`: separate battle-event detection from cue
   playback (other systems will want the events).
3. Unify the three O(n²) regiment-neighbor scans (threat/enemy_near in
   update_groups, support + contagion in morale) into one shared pass.
4. update_groups is the de-facto regiment-state pass — consider its own
   module next to morale.
5. step_sim per-unit body (~300 lines): named inline fns only, no real
   extraction (perf-sensitive kernel).
6. Shader vertex fn: split per-part pose fns.
7. Clippy sweep.

## Asset storage note

The repo does NOT use git-lfs — all 55 tracked binaries (~5 MB of mp3s)
are plain git blobs, and ALL of them live on unpushed feat/battle-feel
commits (origin/main is binary-free). If LFS is ever wanted, the moment
before this branch's first push is the cheapest it will ever be
(local-only history rewrite via `git lfs migrate import
--include="*.mp3"`); at ~5 MB plain git is also perfectly fine.
