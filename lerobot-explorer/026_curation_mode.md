# Curation mode (2026-05-04, feat/curation-mode branch)

## Motivation

Datasets accumulate low-quality episodes (gripper stuck, operator error, 
incomplete grasps). Before implementing destructive dataset editing, we need a
way to visually review episodes and mark them with labels. Curation mode
provides this as a standalone tagging pass decoupled from the actual
deletion/editing step.

Use cases:
- Mark low-quality episodes for batch deletion later
- Bookmark failure patterns to instruct operators
- Group episodes by outcome category for analysis

## Design decisions

- **Multi-label per episode** (unlike annotation which is 1:1 episode-to-prompt).
  An episode can be both "low_quality" and "gripper_stuck".
- **Mutually exclusive with annotation mode.** Both want click-on-pane, so
  enabling one disables the other.
- **Labels created ad-hoc** via text field in the side panel. No YAML config
  needed (unlike annotation prompts). Colors auto-assigned from a 9-color
  palette.
- **Persistence:** `<dataset_dir>/curation.json` (same location strategy as
  annotations.json).
- **Visual feedback:** stacked colored dots in the episode list (left panel)
  and top-right corner of grid panes.
- **Selection model in curation mode:** single-click clears previous selection
  and selects one episode. Shift+click adds to selection. This differs from
  normal grid mode which toggles.

## Architecture

### New file: `src/curation.rs`

```
CurationState
  labels: Vec<CurationLabel>      // name + auto-assigned color
  episodes: BTreeMap<usize, BTreeSet<String>>  // ep -> set of label names
  active_label: Option<usize>     // armed label index for click-apply
  new_label_buf: String           // text field state
  dirty: bool
```

Serialized format (`curation.json`):
```json
{
  "labels": [{"name": "low_quality", "color": [220,60,60]}, ...],
  "episodes": {"3": ["low_quality"], "7": ["low_quality","gripper_stuck"]}
}
```

### App integration (`src/app.rs`)

- `curation_mode: bool` and `curation: CurationState` fields on App
- `enable_curation_mode()` loads from disk, sets `annotate_mode = false`
- `enable_annotation_mode()` sets `curation_mode = false`
- `curation_target_episodes()` returns selected grid episodes (or
  `current_episode` in single view)
- `save_curation()` writes to disk
- Title bar shows `[curate: N/total]` when active

### Grid changes (`src/grid.rs`)

- `show()` returns `Option<usize>` (clicked episode) so the app layer can
  decide what to do with clicks
- New `curation_select: bool` parameter: when true, the grid skips its
  internal `toggle_episode_selection` and the outer `show()` method handles
  exclusive selection with Shift multi-select
- `last_pane_rects: Vec<(usize, Rect)>` stored after render for overlay
  drawing (curation dots)

### UI (`src/ui/panels.rs`)

- Edit menu: "Curation Mode" checkbox (grayed out without dataset)
- Curation panel shown in both info_panel (single view) and cameras_panel
  (grid view)
- Panel lists labels with color dots, count badges, [N] shortcut hints
- Clicking a label arms it AND applies it to target episodes
- Text input for new label creation
- Save button + dirty indicator
- `draw_curation_dots()` overlays colored dots on grid pane corners
- Episode list shows stacked dots per episode

### Input (`src/ui/input.rs`)

- Skip all shortcuts when any text widget has focus (prevents "g" in label
  field from toggling grid)
- 1-9 keys toggle label N on target episodes (selected grid panes or
  current episode)
- Ctrl+S saves curation state

## Interaction flow

1. Open dataset, enter grid view (G)
2. Edit > Curation Mode (checkbox)
3. Type label name in text field, press Enter
4. Click label in panel to arm it (also applies to currently selected ep)
5. Click grid panes to toggle the armed label on that episode
6. Shift+click to multi-select, then click label or press 1-9
7. Ctrl+S to save (or File > Save Curation)

## File-on-disk format

Stored in dataset directory so it travels with the dataset. Separate from
annotations.json since they serve different purposes and have different
schemas.

## Next steps

- Dataset editing feature (consume curation.json to delete marked episodes)
- Possibly filter grid to show only episodes with a given label
- Export curation as CSV for external tooling
