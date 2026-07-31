# 0013 — Perf PR #1: profiling + render path (2026-07-09)

Branch `perf/profiling-and-render-path`, 3 commits. The design-agnostic
optimization stack agreed in chat: Tracy → upload path → culling/LOD.
(SIMD sim kernel + parallel grid rebuild deferred to a future PR — scale
doesn't demand them at 100k.)

## 1. Tracy profiling (`ce6ef1e`)

`cargo run --profile opt-dev --features tracy` + attach the Tracy UI.
Feature forwards to bevy's `trace_tracy` (automatic per-system spans);
manual spans subdivide the hot systems: `grid_rebuild`, `integrate`,
`nn_audit`, `density_field`, `contour`, `process_deaths`,
`sync_instances`. Spans are ~free when no profiler is attached.

Gotcha: the local cargo sparse-index cache was stale ("cannot select
tracy-client ^0.18.3" despite it existing) — fixed by deleting
`~/.cargo/registry/index/*/.cache`.

## 2. Upload path (`21110a4`)

Was: `ExtractComponentPlugin` cloned the full instance Vec every frame
(fresh 3+ MB allocation) and `create_buffer_with_data` made a new GPU
buffer every frame. Now: custom extract copies into a persistent
render-world Vec (allocation reused), and a capacity-tracked persistent
buffer is updated in place with `queue.write_buffer` (1.5× slack on
growth, realloc only when instance count exceeds capacity).

**Gotcha (cost us a black-screen debug)**: `impl SyncComponent` alone does
nothing — `SyncComponentPlugin::<C>` is what registers the
`SyncToRenderWorld` required component. ExtractComponentPlugin used to add
it internally; drop the plugin and you must add SyncComponentPlugin
yourself or the entity never syncs and the extract query matches nothing.

## 3. Per-instance frustum culling + LOD buckets (`ff3e4d3`)

`sync_instance_data` sphere-tests every unit against the camera `Frustum`
(radius 1.3 m; far-plane test skipped so long vistas keep units) and
splits survivors at 300 m into near/far buckets — two instance entities,
each with its own mesh slot. Far mesh is the same cube today; the slots
are where cheaper LOD meshes and per-unit-type meshes plug in without
further infra. Overlay now shows `drawn N (near/far)`.

Measured: wide battle view draws 73.8k/96.6k; zoomed close-up draws
**6,085/96,472 (94% culled)** at 280 fps. Cull test cost is ~0.3 ms for
100k, paid back immediately in upload + vertex work.

## Fix: stale-frustum over-culling (owner-reported, `833c19f`)

Units visibly culled "right in front of you" while moving the camera.
Cause: culling read the `Frustum` COMPONENT, which bevy recomputes in
PostUpdate — so Update-time culling always used last frame's frustum, and
the sync system wasn't ordered after the camera-move system either. At RTS
pan/rotate speeds that's several degrees/meters of error → a band of
missing units inside the leading screen edge. Fix: build the frustum
fresh from this frame's camera `Transform` + `Projection`
(`Frustum(ViewFrustum::from_clip_from_world(..))` — note `ViewFrustum`
lives in bevy_math), order sync after `apply_camera_transform`, cull
radius 1.3 → 2.5 m margin, `FL_NO_CULL=1` escape hatch for A/B.

Lesson: never cull in Update against PostUpdate-computed camera state.

## Fix 2: LOD buckets removed (owner-reported, `92261e7`)

Units vanished at wide views ("no units in the default view"). Pattern:
every confirmed-good view had an empty far bucket; every broken view had
a populated one → the second (far-LOD) instance entity's draws never hit
the screen. Root cause in the multi-entity instanced draw path is NOT yet
diagnosed (suspects: retained-phase item keying with identical sort keys,
mesh-allocator slab slicing for identical meshes). LOD split deleted per
owner call — culling is now strictly visibility (sphere vs frustum,
FL_NO_CULL=1 to disable). Verified: default view draws 50.7k/100k
(rest genuinely behind camera), 269 fps.

**WARNING for the unit-type milestone**: per-type meshes = multiple
instance entities = the exact path that just failed. Diagnose properly
(RenderDoc / per-draw instance counts) before building on it.

True occlusion culling (units hidden by hills) is a separate, later,
GPU-side feature (hi-z); from RTS camera angles the payoff is small.

## Next PR candidates (when scale demands)

SIMD-tightened separation/melee kernel (split x/z arrays), parallelized
counting-sort grid rebuild, GPU-driven culling (only past ~300k units).
