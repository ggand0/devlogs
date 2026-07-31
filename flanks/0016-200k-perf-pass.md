# 0016 — 200k perf pass: parallel hot paths, hitch eliminated (2026-07-09)

Owner verdict on the 0015 test: 200k felt slow in play, and 200k is now the
default standard (`FL_UNITS` default 100k/team) so VFX/animations/real
meshes land on top of headroom, not against the ceiling.

## Step 1: instrument everything first

The overlay only timed grid + step. Added per-system ms to the overlay and
periodic log: `field` (density splat/blur + contour), `audit` (nn sweep),
`sync` (per-frame instance cull + bucket build). The baseline immediately
explained the "feels slow":

| System | Baseline @200k battle | Cadence |
|---|---|---|
| nn_audit | **12–18 ms** | every 2 s — a visible HITCH every 2 seconds |
| step (integrate) | 5.3–6.4 ms | per tick (already parallel) |
| grid rebuild | 3.8–4.6 ms | per tick, serial |
| sync_instance_data | **1.8–4.9 ms** | EVERY FRAME, serial |
| field | 1.1–2.1 ms | per tick, serial |

Lesson repeated from 0013: never judge perf by average fps — the audit
spike never showed in smoothed fps, only in frame pacing.

## Step 2: fixes (all structural, no tuning)

- **Grid rebuild → parallel counting sort** (`spatial.rs` rewrite).
  Count pass: per-chunk histograms + cached per-unit cell ids, in the
  compute task pool. Merge: one serial cells×chunks pass builds prefix
  sums AND per-chunk write cursors (deterministic: cell-major, chunk order
  within cell — identical layout to the old serial sort). Scatter pass:
  chunks write disjoint precomputed slots (one small `unsafe` raw-pointer
  wrapper; soundness = counting-sort offsets partition [0, n)).
- **Grid queries now return `SortedUnit`** — position/team/index packed in
  16 bytes, stored in grid-cell order by the scatter pass. Neighbor loops
  read CONTIGUOUS memory instead of gathering `pos_prev[o]` / `team[o]`
  through random indices. (This mattered less than expected for `step` —
  it's compute-bound, see below — but it's what makes the parallel audit
  cheap and it removes two SoA arrays from the sim's read set.)
- **nn_audit → parallel** over chunks, each returning (min, sum, count),
  reduced at the end. Same full-population coverage.
- **sync_instance_data → parallel** cull/build into per-chunk per-bucket
  scratch vecs (allocations persist via `Local`), then one memcpy concat
  per bucket.
- **Density splat → parallel** per-chunk fields merged linearly; blur
  stays serial (~12k cells).

## Results @200k full engagement (opt-dev, 3900X/3090)

| System | Before | After |
|---|---|---|
| nn_audit | 12–18 ms | **1.6–2.1 ms** (hitch GONE) |
| grid | 3.8–4.6 | **1.4–2.0** |
| sync (per frame) | 1.8–4.9 | **1.2–2.0** |
| field | 1.1–2.1 | **0.6–1.2** |
| step | 5.3–6.4 | 4.6–6.2 (unchanged — dominant now) |
| **fixed tick total** | ~12–13 + spikes | **~7.5–8** |

Behavior verified identical: same nn min/avg (0.36–0.40 / 0.75–0.78
mid-battle), same casualty progression, both buckets drawn, front-line
contour intact (screenshot), clippy clean, FL_TEST_FRONT full battle run.
GPU transparent pass 1.5–1.7 ms — rendering is nowhere near the limit.

## What was deliberately NOT done

`step` (integrate) at ~5 ms is now the dominant tick cost, and it is
compute-bound (sqrt/div chains per close pair), not cache-bound. The
backlog fix is the SIMD separation kernel — but the swing-timer combat
milestone rewrites this exact loop (per-unit targets/cooldowns replace the
gather count). SIMD-ifying the current loop would be thrown away in weeks.
Decision: design the swing-combat kernel SIMD-friendly from day one; the
grid's `SortedUnit` rows are already the right layout for it.

Extract copy (~6 MB/frame into the render world) also measured acceptable;
double-buffering it is a known option if instance data grows (yaw etc.).
