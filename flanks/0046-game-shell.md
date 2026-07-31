# 0046 — Game shell: menu, pause, results, scenario picker (2026-07-24)

## What this adds

A proper game loop wrapping the existing sandbox battle. Three
`GameState` states (Menu / Battle / Results) via Bevy 0.19's `States`
machinery — the first use of states in the project.

### Menu (default state)
- Full-screen translucent overlay with "FRONTLINE" title, subtitle,
  "Start Battle" button, and version text. Terrain renders behind it.
- Enter/Space keyboard shortcut also starts the battle.
- Scenario picker: Army size (20k/50k/100k/200k) and AI (On/Off)
  as clickable toggle buttons. `BattleConfig` resource replaces the
  old `OnceLock` env-var readers; env vars still set defaults.
- Debug Scenarios section: 8 compact buttons (Surround, Rout, Dir
  Defense, Arena, Charge, Pile-on, Join Fight, Rout Pass) — clicking
  any one sets the `Scenario` enum in `BattleConfig` and enters
  battle directly. `do_spawn_battle` routes via config instead of
  env vars. Env vars are synced back on battle start so existing
  log systems keep working.
- Debug overlay (FPS/units) hidden during non-battle states.

### Battle
- `setup_battle` runs in `OnEnter(GameState::Battle)`: resets Units,
  CombatStats, DirTestStats, Selection, BattleOutcome, Corpses, then
  calls `do_spawn_battle`. Replaces the old `spawn_battle` Startup
  system and `restart_key` (R key removed).
- `spawn_banners` moved from PostStartup to `OnEnter(Battle)` with
  `DespawnOnExit(GameState::Battle)` on banner roots — banners clean
  up automatically on state exit.
- Escape toggles pause via `Time<Virtual>::pause()/unpause()`.
  FixedUpdate stops accumulating (no tick burst on resume). Camera
  uses `Time<Real>` so orbiting works during pause. Audio beds fade
  to 0 when paused. Pause overlay with Resume + Quit to Menu buttons.

### Results
- `transition_to_results` watches `BattleOutcome`, waits 3 s for the
  victory sting, then transitions to Results.
- Results screen shows outcome (VICTORY / DEFEAT / MUTUAL DESTRUCTION)
  with per-team stats (alive, killed, fled). Play Again and Main Menu
  buttons.

## System gating

Two SystemSets configured with run conditions:

| Set | Schedule | Condition |
|-----|----------|-----------|
| `SimSet` | FixedUpdate | `in_state(Battle)` |
| `BattleInputSet` | Update | `in_state(Battle) AND NOT time_paused` |

SimSet gates: step_sim, process_deaths, update_morale, apply_reforms,
update_field/update_groups, clear_arrived_orders.

BattleInputSet gates: drag_select, update_hover, control_group_keys,
issue_order, draw_order_preview, halt_key, formation_keys, ai_think,
auto_engage.

Ungated systems (run in all states): camera, terrain, audio setup,
render sync, overlay (hidden in non-battle), banners (state-gated
separately).

## First-frame resource race

Setting `GameState::Battle` as the initial default caused a panic
("Resource does not exist") on the first frame — likely a command
flush ordering issue between `OnEnter` and the first `Update` systems.
The fix: always default to Menu, and auto-transition to Battle on the
first frame if any `FL_TEST_*` env var is active. The one-frame Menu
detour is invisible to test scripts and avoids the race entirely.

## Files changed

- `src/game_state.rs` (new): GameState, SystemSets, GameShellPlugin,
  all menu/pause/results UI.
- `src/main.rs`: added `mod game_state` + `GameShellPlugin`.
- `src/regiments.rs`: `do_spawn_battle` made pub, `spawn_battle` and
  `restart_key` removed.
- `src/banners.rs`: PostStartup → OnEnter(Battle), DespawnOnExit,
  BannerAssets made optional.
- `src/movement.rs`: step_sim → SimSet.
- `src/combat.rs`: process_deaths → SimSet.
- `src/morale.rs`: update_morale → SimSet.
- `src/formation.rs`: apply_reforms → SimSet, formation_keys →
  BattleInputSet.
- `src/frontline.rs`: FixedUpdate → SimSet, InfluenceField made
  optional.
- `src/orders.rs`: input systems → BattleInputSet, FixedUpdate →
  SimSet.
- `src/selection.rs`: input systems → BattleInputSet.
- `src/ai.rs`: ai_think/auto_engage → BattleInputSet.
- `src/camera.rs`: `Res<Time>` → `Res<Time<Real>>`.
- `src/audio.rs`: AudioBank made optional, pause fade added,
  `Time<Real>` for bed blending.
- `src/overlay.rs`: hidden in non-battle states, shown on
  OnEnter(Battle).
- `.gitignore`: added `assets_dev/`.

## Verification

- Menu renders (screenshot verified): title, subtitle, button,
  version, terrain backdrop, no debug overlay bleed.
- `FL_TEST_DIR=1`: auto-starts battle, dir-test logs appear, regiments
  engage, kills bucket by sector — identical to pre-shell behavior.
- No panics in normal (Menu) or test-flag operation.
- Clean build, zero warnings.
