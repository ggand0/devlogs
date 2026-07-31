# 0014 — Multi-entity instanced draws: root cause + fix (2026-07-09)

The blocker from devlog 0013 ("far LOD bucket invisible") is diagnosed and
fixed. The render path now drives N instance entities correctly — the
prerequisite for per-unit-type meshes (one instanced draw per type).

## Intuition

Two terms, used consistently below:

- **Instance** = one cube on screen. 100k of them.
- **Object** = a mesh + a list of positions + ONE draw call that stamps
  that mesh at every position in its list. (In code: the instance entity
  in `render_units.rs` — `Mesh3d` + `InstanceMaterialData` +
  `InstanceBucket`.)

Before this change the game had exactly one object drawing all 100k
instances, both teams included — color is a per-instance attribute, so
blue and orange never needed separate anything. One draw call for
everything, the ideal instancing setup. Nothing about it was broken.

The limit: one object can only use ONE mesh. A draw call is "stamp *this
mesh* N times" — an archer mesh and a swordsman mesh can never share it,
no matter what attributes instances carry. So unit types (the M2TW
direction) require one object per mesh type: 5 types ≈ 5 objects ≈ 5 draw
calls. Still fully instanced — the win of instancing is 5 draw calls
instead of 100,000, not 1 instead of 2.

And the render path could not actually run more than one object. That was
the latent bug: bevy builds a per-frame to-do list of draw calls and runs
an automatic merge pass meant for ordinary meshes — two entries that look
identical (same shader/material/mesh kind) get folded into one, assuming
the first covers both. True for ordinary meshes; false for our objects,
each of which drags its own private instance list. After the fold, only
the first object's draw call runs. The second object's entire army is
skipped — no error, no log. With one object there is nothing to fold,
which is why the prototype always looked fine and why the 0013 LOD
experiment (the first-ever second object) broke instantly and
mysteriously.

The fix is `NoAutomaticBatching` on each object: bevy's marker for
"don't merge my draw calls". One component; the merge pass leaves the
objects alone and every draw call runs.

Why the code now splits units into two objects by TEAM, when teams don't
need it (color already did the job): purely a live test. With a single
object the fix would sit unexercised until the unit-type milestone; split
by team, the normal game runs the two-object path every frame and the
proof is simply that both armies are visible. Costs nothing (transparent
pass 0.57 ms, unchanged). Milestone #3 replaces the team split with the
real per-type split by changing `bucket_of()` and adding meshes.

## Symptom (recap)

Two instance entities through the custom instanced pipeline: the second
entity's draws never reached the screen. CPU-side counts said its instances
were submitted; the GPU never drew them. No errors anywhere.

## Root cause: bevy's sorted-phase batcher merges the phase items

Read straight from bevy 0.19 source (`bevy_render/src/batching/
gpu_preprocessing.rs`, `render_phase/mod.rs`, `bevy_pbr/src/render/mesh.rs`):

1. Any entity with `Mesh3d` is registered in `RenderMeshInstances` with the
   `AUTOMATIC_BATCHING` flag set — including our custom-pipeline instance
   entities. Batching is opt-out, not opt-in.
2. `batch_and_prepare_sorted_render_phase` walks ALL phase items in
   `Transparent3d`, even ones queued by third-party code with custom draw
   functions. For each item it fetches compare metadata
   (`get_index_and_compare_data`); two entities with the same pipeline id,
   draw function, material binding slot and mesh slab compare batch-equal
   and are folded into one batch.
3. `SortedRenderBatchSet::flush` writes the merged `batch_range` into the
   FIRST item of the batch only.
4. `SortedRenderPhase::render_range` executes the first item's draw
   function, then does `index += batch_range.len()` — every later item in
   the batch is SKIPPED. Its draw function never runs.

With one instance entity the batch has length 1, so everything works —
which is why the bug only appeared when the LOD experiment added a second
entity. Both suspects from 0013 are now resolved: phase-item batching was
the bug; mesh-allocator slab slicing was innocent (verified below).

## Fix

`NoAutomaticBatching` on every instance entity
(`bevy::render::batching::NoAutomaticBatching`). That clears the
`AUTOMATIC_BATCHING` flag at extraction, `should_batch()` returns false,
the batcher emits one single-item batch per phase item, and every draw
function runs. This is bevy's designed opt-out for exactly this case, wired
through extraction with change detection — not a workaround.

## Render path is now bucketed (production, not scaffold)

- `InstanceBucket(usize)` component; `NUM_BUCKETS` instance entities, each
  with its own mesh asset. Bucket key today: team (`bucket_of()` in
  `render_units.rs`). The unit-type milestone changes that one function to
  key by type and adds meshes — no new render infra needed.
- `sync_instance_data` fills all buckets in one sweep over the SoA
  (replaces the `single_mut()` that silently no-ops with >1 entity).
- Distinct mesh assets per bucket land in one vertex slab at different base
  offsets, so the allocator/base-vertex path is exercised every frame.
- Overlay + periodic log show per-bucket drawn counts:
  `drawn 50744 [35523/15221]`.

## Verification (A/B, same default view)

- WITHOUT the fix (buckets split by team, no `NoAutomaticBatching`):
  overlay `drawn 50744 [35523/15221]`, screenshot shows ONLY blue — the
  orange bucket (15,221 submitted instances) is invisible. Exact repro of
  the 0013 symptom, on demand.
- WITH the fix: same counts, both armies visible. GPU
  `main_transparent_pass_3d` 0.57 ms (unchanged vs single-draw baseline) —
  the second draw call is free at this scale.
- `FL_NO_CULL=1`: `drawn 100000 [50000/50000]`. Clippy clean.

## Lesson

Bevy's render-phase batcher owns every phase item, including ones you queue
yourself with a custom draw function. If you add phase items for entities
that live in `RenderMeshInstances`, you MUST opt out with
`NoAutomaticBatching`, or any batch-equal pair silently drops all but the
first draw. Nothing logs, nothing errors — draws just vanish.
