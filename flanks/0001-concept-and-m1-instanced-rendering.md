# 0001 — Concept & M1: Instanced rendering (2026-07-08)

## Concept

`frontline` is a large-scale swarm battle prototype: two armies of ~50k units
(100k total) fight on deformable terrain. Units are not individually microed —
they belong to **groups** that autonomously form and hold a frontline, like a
hybrid of Total War, War of Dots, and UEBS. The player commands at group level:
lasso a subset, order attack directions (salients), cut groups apart. Adjacent
groups keep their front curves connected to neighbors.

Hard target: **60 FPS at 100k units** on a desktop GPU. Every architecture
decision follows from that.

## Architecture (locked in)

- **Units are NOT entities.** All per-unit state lives in SoA buffers inside a
  `Units` resource (`Vec<Vec3>` positions, team ids, colors; velocities/health
  come with later milestones). Bevy entities exist per-group/army only.
- **Rendering**: one custom instanced draw per mesh type. Per-instance data
  (pos, scale, color — 32 B/unit) re-uploaded each frame from the sim buffers.
- **Sim** on `FixedUpdate`, rendering interpolates (from M2 on).
- **Spatial index**: uniform grid rebuilt per tick (M2+).
- **Terrain**: chunked heightmap, deformable via crater subtraction (M3).
- **Frontline solver**: 2D density fields on a downsampled grid → contact-band
  front curve → slot assignment (M5).

Milestones: 1 render → 2 movement → 3 terrain → 4 selection/orders →
5 frontline → 6 combat.

## M1 implementation

- **Bevy 0.19.0** (checked crates.io; released 2026-06-19). Required
  rustc ≥ 1.95 → updated stable toolchain to 1.96.1 via rustup.
- Deps: `bevy`, `bytemuck` (Pod-cast of instance structs for GPU upload).
- Renderer (`src/render_units.rs`) adapted from Bevy's official
  `examples/shader_advanced/custom_shader_instancing.rs` at tag v0.19.0:
  - Custom `RenderCommand` + `SpecializedMeshPipeline` on the `Transparent3d`
    phase (sorted phase is the simple path; opaque binning not worth it yet).
  - Instance-rate vertex buffer recreated each frame in `PrepareResources`
    (`create_buffer_with_data`), one `draw_indexed` for all 100k instances.
  - Entity needs `NoFrustumCulling` (instance positions aren't in its Aabb),
    camera needs `NoIndirectDrawing` (we issue direct draws).
  - Shader embedded via `embedded_asset!` so the binary runs from any cwd
    (asset-folder lookup fails when running `./target/.../frontline` directly).
- Shader (`src/shaders/unit_instancing.wgsl`): per-vertex lambert + hemispheric
  ambient on the cube face normals → flat-shaded chunky look, no PBR cost.
  Cubes are 0.7×0.9×0.7 (taller than wide, reads as a figure). Per-unit tonal
  variation (hash-based) baked into instance color so 50k units don't read as
  a flat texture.
- `src/units.rs`: SoA store + two 500×100 army blocks with position jitter.
- `src/camera.rs`: RTS camera (WASD pan scaled by zoom, scroll zoom,
  middle-drag rotate, pitch clamped). **Gotcha found**: X11 here delivers
  *pixel*-unit scroll deltas (~50 px/notch), not lines — must check
  `AccumulatedMouseScroll::unit` or one wheel notch zooms ~12 steps.
- `src/overlay.rs`: FPS / frame-ms / unit count text + periodic log, plus
  `RenderDiagnosticsPlugin` GPU pass timings.

## Perf findings

- Unit draw (`main_transparent_pass_3d`): **~0.9–1.3 ms GPU** for 100k units,
  measured while the GPU was shared with other apps and at reduced clocks (P3).
  Total GPU frame ~1.6 ms → 60 FPS target has ~10× headroom before sim work.
- **Measurement trap**: wall-clock FPS decays to ~60 after a few seconds even
  with *nothing* drawn — GNOME/X11 compositor present-throttling (60 Hz
  secondary monitor clock; primary is 144 Hz), not render cost. When the
  compositor lets it run free: ~206 fps in debug build. Conclusion: trust GPU
  pass timings from the periodic log, not raw FPS.
- CPU cost of the naive per-frame path (rebuild 100k instance vec, clone in
  extract, recreate GPU buffer) is fine so far (~4–5 ms total frame in debug
  build); revisit only if it shows up after sim work lands.

## Verification method (no human in the loop)

Run windowed on the dev desktop, screenshot via `import -window` (ImageMagick),
drive input via `xdotool` (XTEST — `--window`/XSendEvent is ignored by winit).
Confirmed: rendering, overlay, camera zoom/pan, army layout, lighting.
