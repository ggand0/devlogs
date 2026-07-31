# 0066: the Select Units army picker

Branch: feat/army-picker. First milestone of 0.1.0 (picker, then
scenario format, then itch packaging).

## What was built

An M2TW-style Select Units screen between the menu and deployment
(new GameState::UnitSelect, src/picker.rs). The owner asked for the
M2TW custom-battle feel: drag cards from a roster pane into the army
list. Reference screenshot: the Battle of Arsuf custom battle setup.

- The budget is regiment slots: army size / regiment size. No unit
  costs. One regiment fills one slot regardless of kind. The owner
  asked about zero costs; we skipped the field entirely instead of
  storing a dead zero. Point costs can arrive with the scenario
  format if ever needed.
- Right pane: four roster cards (Knights, Spearmen, Men-at-Arms,
  Bowmen) with the existing rasterized SDF kind icons and a live
  count per card. Click adds one regiment, drag-and-drop into the
  army list also adds, Shift multiplies by 10. Hover swaps a fixed
  description strip under the panes.
- Center pane: the army list as a wrap grid, one cell per slot,
  filled cells in battle-ladder order (heavy, spear, light, archer).
  Clicking a filled cell removes that regiment. Partial armies are
  allowed like M2TW: slots are a cap, not a quota. An empty army
  blocks Start.
- Team arrows flip to Team 2. The enemy page adds a Random/Manual
  toggle. Manual gives the same grid. Random (the default) rolls an
  army style at battle start: Balanced Host, Iron Wall, Spear Hedge,
  Arrow Storm, or Skirmish Horde, each defined as heavy/spear/archer
  fractions with light as the rest, jittered plus-minus 20 percent
  per draw. The owner asked for an enemy that is "effective in its
  own way every time" plus full manual control; this is both.

## Wiring

- BattleConfig gains player_regs and enemy_regs (Option; None =
  random roll). No new env OnceLocks. Changing army size on the menu
  resets both comps (stale counts could overflow the new budget).
- do_spawn_battle consumes explicit per-kind counts and builds the
  kind ladder from them. The FL_*_FRAC envs and FL_ENEMY_REGS
  override the picker comps, and scripted runs (scripts_active) are
  pinned to the classic split too, so no test battery ever meets a
  random enemy. Verified by log: frac override and FL_ENEMY_REGS
  runs spawn identical counts to pre-change math.
- Start Battle and Enter route Menu -> UnitSelect -> Battle;
  scripted scenarios and FL_DEPLOY=0 skip the picker exactly like
  they skip deployment. Esc goes back to the menu.

## Bevy picking notes (0.19)

First use of observer-based UI in the repo. Facts checked against
the vendored crate source, not memory:

- On<Pointer<E>> derefs to the event; during propagation the entity
  field is rewritten to the current node, so a card observer sees
  the card even when the press landed on its Text child.
- Release order is Click -> DragDrop -> DragEnd. PickerDrag.kind
  doubles as the guard so the release that ends a drag cannot also
  count as a click-to-add.
- Entities without Pickable block lower entities by default, so only
  the topmost node is hovered and each Click/DragDrop fires exactly
  once per gesture. The drag ghost carries Pickable::IGNORE or it
  would occlude the drop target under the pointer.
- The ui umbrella feature includes the picking backend, so no
  Cargo.toml change was needed.

## Self-checks

opt-dev build and clippy clean. Screenshot flow (sanctioned):
FL_VOLUME=0, xdotool Return past the menu, import -window. Picker
renders prefilled 25/25 at 50k with correct roster counts. Starting
from the picker spawned the picked comp exactly; the enemy rolled
Skirmish Horde and the totals in the spawn log reconcile per kind.
Drag-and-drop and team flipping await the owner's live play-test.
