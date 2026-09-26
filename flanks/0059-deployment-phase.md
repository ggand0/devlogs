# 0059 — feat/deployment-phase: TW-style pre-battle deployment

**Date:** 2026-07-26
**Branch:** `feat/deployment-phase` (design in docs/plans/005-deployment-phase.md)

## What landed

A normal battle now opens frozen in a deployment phase: both armies
stand in their spawn arrangement, the player's zone is drawn as gold
gizmo lines (bright boundary toward the enemy, dim sides/back, matching
the M2TW yellow-border convention), and a Begin Battle button (or
Enter) releases the sim. The army picker is deliberately NOT here; it
is the next branch.

## How it works

- `Deployment { active: bool }` RESOURCE in game_state.rs, not a
  sub-state: `setup_battle` decides it inside OnEnter(Battle), and a
  NextState write there would only apply through the next
  StateTransition run, leaving one frame where a stale phase from the
  previous battle could let SimSet tick. A resource write is
  immediate. Cleared on OnExit(Battle).
- Gates: `SimSet` = Battle AND `deployment_done`. `ai_think` +
  `auto_engage` (Update systems, not SimSet) get their own
  `deployment_done` run_if so the enemy issues no pre-orders and
  nobody auto-engages across the line. `BattleInputSet` untouched:
  camera, lasso, unit cards, HUD buttons all live while frozen. Not a
  Time pause because BattleInputSet requires `not(time_paused)`.
- Entry rule: `Scenario::Normal` + no FL_TEST_* scripts + FL_DEPLOY
  != 0. Every test battery and debug scenario starts fighting
  directly, so acceptance tests are untouched.
- Placement reuses the whole RMB gesture path in orders.rs. The fork
  is only at effect time: drag release calls `deploy_place_line`
  (anchor/facing/files from the same `line_layout`, then teleport),
  click calls `deploy_move` (arrangement-preserving centroid
  translation, facing kept). No attack branch while deploying.
- `formation::snap_to_slots` is the teleport primitive: assign_slots,
  then pos/pos_prev/yaw/yaw_prev/vel snapped per member, centroid
  pinned to the anchor (the sim owns centroid and it is frozen;
  selection, banners, and follow-up gestures read it).
- Zone = player's side of the no-man's land: x in [min+30, max-30],
  z in [min+8, -army_gap/2] (same margins as the spawner;
  regiments::army_gap made pub). Clamping happens on the PREVIEW
  layout (clamp_layout_to_zone), so the green slots always show the
  final spot; drags outside slide along the edge, TW-style. Clamp
  accounts for the rotated block's AABB half-extents so the whole
  block stays inside, blob shape clamps by its disc radius.
- Formation commands during deploy (F/L/B keys, HUD buttons):
  `apply_reforms` lives in the frozen SimSet, so `deploy_reform_snap`
  (Update, after BattleInputSet, deploy-gated) snaps any raised
  `reform` flag immediately.
- UI: top banner + bottom-center "Begin Battle (Enter)" button,
  DespawnOnExit(Battle) + explicit despawn on begin.

## Notes / follow-ups

- ai_think's 5 s orientation grace counts from battle-scene entry
  (virtual time runs during deploy), so after a deployment of any
  length the AI advances immediately on Begin Battle. TW-like, kept.
- Enemy keeps its spawn arrangement; AI deployment variety belongs to
  the scenario format branch.
- Verified: cargo build --profile opt-dev + clippy clean. Owner
  play-tested and approved the phase ("ok it works").

## Cleanup pass (22190c7, 4-angle review)

- Dead code out: the OnExit flag reset (every reader is
  battle-gated and setup_battle reassigns on entry) and the
  team re-checks in deploy_place_line/deploy_move (Selection is
  player-only by construction; the battle-order fns already trust
  it). deploy_move now takes Selection::picked_controllable.
- One gating idiom: deployment_done deleted in favor of
  not(deploying); draw_deploy_zone gated by run_if like every other
  deploy system instead of an in-body flag check.
- Perf: assign_slots returns its member list so snap_to_slots no
  longer rescans all 200k units per regiment — a full-army placement
  was 2 scans/regiment (~40M filtered iterations at 100 regiments),
  now 1 (the assign pass itself). The single-bucketing-pass variant
  (1 scan total) was considered and skipped: sim is frozen and the
  gesture is one-shot; revisit only if release hitches at 200k.
- Geometry home: block_half_extents moved to formation.rs next to
  slot_offsets; blob_radius extracted and shared with assign_slots
  (density retunes can't diverge from the zone clamp anymore).
- Shared facts named: regiments::SIDE_MARGIN/EDGE_MARGIN replace the
  30.0/8.0 literals in both the spawner and deploy_zone — the zone
  contains the spawn strip by construction now.
- Deploy UI merged into one root (banner top, button bottom via
  SpaceBetween); begin_battle despawns one entity.

## Feature-complete? For this branch, yes

In-scope items all landed. Known gaps are deliberate branch
boundaries, not omissions: army picker (next branch), enemy AI
deployment variety + scenario-defined zones (scenario format
branch), no reset-to-default-deployment button (polish, cheap to add
if wanted), and the zone clamp is rectangle-only — it does not
exclude the shallow river if one crosses the player strip (rivers
are wadeable since 0044, so harmless, but M2TW would mask it out).
