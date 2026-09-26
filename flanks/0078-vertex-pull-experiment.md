# 0078: Vertex pulling experiment, step 3a. Result: POSITIVE (2026-09-20)

Branch `exp/vertex-pull`, from `main` e80dbe8. Plan: docs/plans/010-scale-to-1m.md, item 3. Background: devlog 0077, which measured a GPU cost per drawn soldier that mesh simplification could not remove.

## Question

Does drawing far soldiers in one plain draw, instead of one instance per soldier, remove that cost? The plan for a renderer that can show 1M soldiers assumed yes. This experiment checks it before the two-week GPU render data rewrite gets designed around it.

## Answer

Yes. In the 900 m view with all 197k soldiers on L3, the unit pass drops from 3.3 ms to 1.15 ms. With GPU clock differences removed the ratio is about 3.6 times. The picture is identical. The instanced path stays the default and is untouched.

A second finding matters for every GPU number in this project. The GPU core clock on this box swings between about 550 and 1935 MHz during a run, and pass times follow it. Devlogs 0076 and 0077 did not account for this. Corrections are below.

## What was built

`FL_PULL=n` draws the n farthest detail levels without instancing. Unset or 0 keeps today's renderer. `FL_PULL=1` is the experiment as planned, L3 only. `FL_PULL=4` pulls every level.

- A pulled bucket issues ONE non-indexed draw of `soldiers * corners` vertices with no vertex buffers bound.
- The shader entry `vertex_pull` takes `@builtin(vertex_index)`. Soldier is `index / corners`, corner is the remainder. `corners` is a shader constant per pipeline, so the divide is by a literal.
- It reads the soldier's unchanged 64-byte record and the mesh corner from two storage buffers at group 3. It builds the same `Vertex` struct the instanced entry receives and calls the same `unit_vertex` function. One pose code path, so the two can never look different.
- The mesh corner buffer is the level mesh with its index list expanded, built once at startup from the same `Mesh` the instanced path uses. `unit_meshes.rs` is untouched.
- A pulled bucket's instance buffer gets `STORAGE | COPY_DST` usage. Instanced buckets keep `VERTEX | COPY_DST` exactly as before.
- CPU work is unchanged. `sync_instance_data` does not know about pulling.

Files: src/render_units.rs, src/shaders/unit_instancing.wgsl.

## Bevy 0.19 gotcha

`SpecializedMeshPipelines` cannot hold a pipeline without vertex buffers. Its cache reads `descriptor.vertex.buffers[0]` and panics on an empty list (pipeline_specializer.rs:224). The pulled pipeline therefore goes through `SpecializedRenderPipelines`, with the mesh layout carried inside the key. `CustomPipeline` implements both traits. The instanced specializer is byte for byte what `main` has.

Bevy 0.19 is the latest stable release. 0.20 is at rc.1. The owner is open to an upgrade when a newer version enables something useful. Nothing in this experiment needed one.

## Measurement method, and what was wrong with it before

Recipe as in the handoff: muted, scripted battle, locked camera, read `main_transparent_pass_3d`. New this time: `nvidia-smi` logs power state and core clock twice a second next to the game log. The script is work/scripts/gpu-pass-measure.sh. It prints each pass sample with the drawn counts and the clock at that moment.

What the clock log showed:
- In a light view the 3090 sits in P3 or P5 at 700 to 1400 MHz. Under a heavier unit pass it holds P0 at 1935 MHz.
- Pass time follows the clock. The same instanced L3 frame read 2.2 ms at 1800 MHz and 3.5 ms at 1000 to 1100 MHz.
- Other jobs on the box loaded the CPU during these runs (load average 10). Frame rate swung between 20 and 95 fps, and the GPU clock swung with it. No other process used the GPU.
- A cheaper draw path makes the GPU less busy, so it clocks lower. Raw milliseconds therefore UNDERSTATE the gain of a faster path.

Normalizing: pass time times core clock in GHz gives a work figure that stayed steady within a run (instanced L3: 3.9 to 4.1 across eight samples). It is approximate. The clock sample is instantaneous and the pass value is smoothed. Memory clock also changes with power state.

The clean fix for future sessions is to pin the clock while measuring: `sudo nvidia-smi -lgc 1900,1900`, and `sudo nvidia-smi -rgc` afterwards. That needs the owner's sudo. Without it, always log the clock and compare at equal clocks.

## Results (RTX 3090, 1600x900)

### 900 m, culling off, everything on L3 (the planned A/B)

| Army drawn | Instanced | Pulled | Note |
|---|---|---|---|
| 197k, run 1 | 3.30 ms | 1.15 ms | medians of 10 samples, no clock log |
| 197k, run 2 | 3.2 ms at about 1250 MHz | 1.1 ms at about 1000 MHz | clock logged |
| 197k, normalized to 1.9 GHz | about 2.1 ms | about 0.58 ms | ratio about 3.6 |
| 97k (control) | 1.42 ms | 0.54 ms | no clock log |

The control shows the cost scales with soldier count in both paths. It is a per-soldier cost, not a fixed cost of the pass.

### 280 m default zoom, culling on, 81k drawn (62k L2, 19k L3)

| Variant | Raw, end of run | Work figure (ms x GHz) |
|---|---|---|
| Instanced | 1.96 to 2.43 ms | about 2.8 |
| `FL_PULL=1`, L3 pulled | 1.22 to 1.56 ms | about 1.6 to 2.0 |
| `FL_PULL=2`, L2 and L3 pulled | 0.84 to 0.96 ms | about 1.2 |

The three runs are deterministic in drawn counts, so rows compare one to one. Blue marches into view during the run, which is why counts rise from 42k to 81k.

UNEXPLAINED: pulling only the 19k to 26k L3 soldiers saved about 1 ms here. The 900 m data predicts about 0.3 ms for that many. Clock noise is large in this view, so the size of the effect is uncertain. The direction is not. A clock-pinned rerun would settle it.

### 900 m, half army (97k), one level forced, GPU held P0 by the load

| Level | Instanced | Pulled | Clock |
|---|---|---|---|
| L1 (120 to 168 tris) | 1.76 ms | 1.69 to 1.75 ms | 1.92 GHz against 1.78 GHz, so about 10 percent less work pulled |
| L0 (180 to 480 tris) | 2.86 ms | 2.58 ms | both 1935 MHz, samples steady to 0.01 ms |

These are the cleanest numbers of the session. Pulling wins at EVERY level, including the full mesh.

### Per soldier at about 1.9 GHz

| Level | Instanced | Pulled | Saved |
|---|---|---|---|
| L3 | about 10.7 ns | about 2.9 ns | about 7.8 ns |
| L1 | about 18 ns | about 16 ns | about 2 ns |
| L0 | about 29.5 ns | about 26.6 ns | about 2.9 ns |

Reading: instancing costs about 8 ns per soldier on this GPU whatever the mesh. The pulled path gives that back. On bigger meshes it pays some of it again, because a non-indexed draw shades every corner. L0 runs the vertex shader about 1.5 times as often as the indexed draw does. An indexed pull (one static index buffer of `soldier * verts + local index`) would keep vertex reuse and should recover most of that. Not built. It is a design option for item 2.

## Corrections to devlogs 0076 and 0077

1. "About 17 to 19 ns per drawn soldier, independent of mesh size." That was read on a downclocked GPU. At full clock the L3 figure is about 11 ns. The per-instance part is about 8 ns.
2. "Triangles dropped 8.6 times, cost 1.5 times." Partly a clock artifact. The LOD-off run (5.57 ms) held the GPU at full clock. The LOD-on run (3.7 ms) let it drop to about 1.1 to 1.2 GHz. At matched clocks the cost per soldier drops about 2.8 times from L0 to L3.
3. So LOD delivered more than devlog 0077 credited. The floor is real but about half the size recorded there.
4. Every pass time in 0076 and 0077 taken in a light view carries this error. Heavy views (LOD off, dense meshes) were at full clock and stand.

## What this means for the plan

- The assumption under item 3 holds. At 1M drawn soldiers on L3 the GPU will be at full clock: about 10.7 ms instanced, about 2.9 ms pulled. The plan estimated 3 to 4 ns per soldier pulled. Measured 2.9.
- Item 2 (render data built on the GPU) can be designed around ONE pulled draw path for all levels. Two draw paths are not needed. Pulled draws take their vertex count from an indirect buffer the same way instanced draws take an instance count.
- The indexed-pull variant should be measured during item 2, because near levels have the most to gain from vertex reuse.
- Order stays as approved: item 0 (tick off the frame path) is next.

## State

- Build and clippy clean with warnings as errors. Default behavior is unchanged with `FL_PULL` unset.
- A/B screenshots of a forced-L3 close-up at 10k: both paths draw the same soldiers, same shading, same lean. No wgpu validation errors in any pulled run.
- NOT done: the owner has not looked at a pulled run himself. No sim code changed, so no battery was run.
- The game is CPU-bound at 200k today, so the GPU saving does not show up as fps yet. It is headroom for authored models and for larger armies.

## Commands

```
FL_PULL=1 cargo run --profile opt-dev     # L3 pulled
FL_PULL=2 cargo run --profile opt-dev     # L2 and L3 pulled
FL_PULL=4 cargo run --profile opt-dev     # every level pulled
work/scripts/gpu-pass-measure.sh far_pull1 FL_UNITS=100000 FL_NO_CULL=1 FL_CAM_DIST=900 FL_PULL=1
```

## Open

- Owner decision: keep `exp/vertex-pull` as a reference for item 2, or merge it with pulling off by default.
- A clock-pinned rerun of the 280 m view, to size the L3 effect that the 900 m data does not explain.
- Integrated GPUs are still unmeasured, as in 0077.
