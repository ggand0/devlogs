# 055 - Regiment Formations (M2TW-style)

Formations now matter: dynamic rectangle ranks with drag-to-shape, walls, loose
order, defend/at-ease stances, and a morale system that breaks squads into
fleeing blobs and rallies them. Plus a new procedural **spear infantry** unit.

## New unit: Spear Infantry

- `UnitClass::{Sword, Spear}` component + stored on `Squad`.
- Procedural spearman in `procedural_meshes.rs`: chainmail hauberk + skirt +
  coifed head (grey metallic body mesh), **kettle hat** child (brim + domed
  crown + apex knob, steel), **spear** child (3.6u shaft + leaf point, pivot at
  the grip), team-colored **tabard** child (commanders wear gold/orange).
- Armies alternate sword/spear squads (`squad_index % 2`), also in scenario
  garrison/waves and the debug map.
- Spawn code was deduplicated: `UnitSpawnKit` (per-team meshes + materials)
  replaces `create_droid_mesh`/`create_team_materials`, and `spawn_team_squads`
  now goes through `spawn_single_squad`.

## Formation model (types.rs / formation.rs)

Per-squad state: `formation_type` (Rectangle | **Blob**), `formation_width`
(files), `spacing` (Normal | Loose | Wall), `stance` (AtEase | Defend),
`morale`, `routing`.

- `formation_slot_offset()` is width/count aware; partial back rows are
  centered. Blob uses a deterministic golden-angle sunflower scatter (stable
  per slot, bounded radius) - the "mob" look without per-frame randomness.
- `formation_reflow_system` re-deals living members into slots (front ranks
  filled first) whenever width/spacing/formation changes or casualties land
  (`squad.needs_reflow`, set by `remove_unit_from_squad`). Casualty tracking
  now also works for explosion deaths (the old system queried the despawned
  entity and silently did nothing).
- Unit tests cover row centering, partial-row centering, blob determinism,
  commander slot (rear-center), and the line planner.

## Drag-to-shape move command (the big one)

RMB-drag now draws the **front line** M2TW-style (replaces the old
CoH-style "drag = facing" behavior):

- Drag length = total frontage; facing = perpendicular to the line, away from
  where the squads currently stand; front rank stands exactly on the line.
- Frontage is split among selected squads proportionally to living members
  (6u gap between squads); each squad's `formation_width` becomes
  `frontage / lateral_spacing`, clamped to [2, count]. Squads keep their
  left-to-right order along the line (no crossing).
- While dragging, `placement_preview_system` shows a seafoam ground disc per
  soldier (pooled entities, capped at 2500) - **exactly** the same planner
  (`plan_line_placement`) executes on release, so the preview is WYSIWYG.
- Plain RMB click keeps the old behavior (line of squads at default widths,
  facing the destination). Shift+RMB still = Attack Move.

## Hotkeys (selected squads)

| Key | Effect |
|-----|--------|
| Q | Defend (hold position) <-> At Ease |
| X | Wall toggle: **Shieldwall** (sword) / **Spearwall** (spear) |
| R | Loose spacing toggle |
| Z | Blob <-> Rectangle |

Majority-based toggles; routing squads ignore orders.

## Stances & auto-engagement (stance.rs)

- **At Ease** (default): if an enemy squad center comes within 220u (just
  beyond weapon range) and the squad has no in-flight order, it turns to face
  and advances to a **95u standoff - just inside optimal accuracy range**
  (RANGE_SEGMENT_0_END = 100u, zero range penalty), keeping formation. Runs
  for both teams, so battles come alive without a scripted AI.
- **Defend**: members get `MovementMode::Hold`, never auto-advance - they
  hold the line but eat the long-range accuracy penalty instead of closing.

(First pass used a 90u engage radius; with 200u guns nobody ever idled inside
it, so squads fired from max range and never advanced. Tying the standoff to
the accuracy bands makes the stance a real trade-off.)

## Formation combat effects

- **Walls**: collision mass x2.5 (sword) / x3.5 (spear) - braced ranks resist
  M2TW-style pushing; frontal hits (shot dir vs squad facing) blocked 35% /
  25%. Spearmen **visibly lower spears** in spearwall (`spear_pose_system`
  slerps the spear child ~66 deg forward).
- **Loose**: 25% dodge vs all incoming infantry fire (the anti-missile
  formation), mass x0.8.
- **Routing**: mass x0.6 - fleeing mobs get shoved around.

## Morale & rout

Tick every 0.5s per squad: -2 per casualty, -5/s flanked (enemy within 30u
behind the facing), -3/s friendly routing nearby, -2/s under 40% strength;
+2/s braced in a wall under pressure, +3/s when no enemy within 70u (+4/s
extra while routing so they can rally).

- Break at <25: squad becomes a **fleeing blob** (formation=Blob, drops
  targets, mass down), runs 80u legs away from the nearest enemy.
- Rally at >60: reforms ranks (Rectangle) where it stands.
- Routing squads don't acquire targets and ignore player commands.

## Verification (log-based)

10k-unit battle driven via xdotool (T advance + box select + RMB drag):

- `Box selected 4 squads` -> `Line placement ... for 4 squads` -> per-squad
  `width N files, M ranks` (proportional split confirmed).
- Q/X/R/Z transitions logged (`stance -> Defend`, `-> Shieldwall`,
  `spacing -> Loose`, `formation -> Blob/Rectangle`).
- Hundreds of `morale broke - ROUTING` / `rallied - reforming ranks` cycles
  over a few minutes of battle; armies clashed, broke and reformed at their
  lines (emergent, very TW-looking).
- 108 `(AtEase) advancing Nu to engage` events in a fresh run with zero input:
  the battle self-starts, squads close to optimal range, fight, rout, rally,
  re-engage.
- Screenshots: spearmen render with kettle hats/tabards/upright spears;
  spearwall visibly levels the spears; preview discs show during RMB drag.
- 122-145 FPS with 10,000 units + preview dots.
- `cargo test`: 6/6 formation/planner tests pass.

## Files

`types.rs`, `constants.rs`, `formation.rs`, `stance.rs` (new),
`procedural_meshes.rs`, `setup.rs`, `combat.rs`, `selection/{input,movement,
preview(new),mod,ui}.rs`, `scenario/{mod,wave}.rs`, `terrain.rs`, `main.rs`.

## Tuning knobs (constants.rs)

All under `FORMATION SYSTEM` / `MORALE SYSTEM` / `PLACEMENT PREVIEW` sections:
wall/loose spacing multipliers, block/dodge chances, mass multipliers,
engage/standoff radii, all morale rates and thresholds, preview dot look.

## Future ideas

- Peasant unit class defaulting to Blob + low morale cap.
- Charge bonus: mass spike for the first seconds of contact.
- Missile units to make Loose spacing earn its keep.
- Rout-to-map-edge + despawn (true M2TW routs) instead of rally-in-place.
- Formation-aware pathing so wide lines wheel instead of pivoting.
