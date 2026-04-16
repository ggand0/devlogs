# Grid UI polish (2026-04-16)

Post-multi-camera UI fixes for grid/tiled/subgrid views.

## Changes

### Grouped selection borders (show_flat)

In tiled mode (`cam_count > 1`), clicking an episode selects all its camera
panes via `toggle_episode_selection`. Previously each pane drew its own
`rect_stroke`, making selected episodes look like individual highlighted cells.

Fix: collect pane rects during the drawing loop. After the loop, group selected
panes by `episode_index`, compute the bounding `Rect::union` across all panes
in the group, and draw a single border around the whole group. Per-pane borders
are skipped when `cam_count > 1` (single-cam grid still draws per-pane).

### Label modes (LabelMode enum, L key)

`LabelMode` enum on `App` with three states cycled by `L`:

- **Compact** (default): `"ep 001"` badge at top-left with semi-transparent
  dark background (`from_black_alpha(160)`) and light text. Always readable.
- **Verbose**: full `"ep 001 overhead  137/300"` at bottom-left, also with
  dark background. Accent color when selected.
- **Hidden**: no label.

Shared `draw_pane_label` helper used by both `show_flat` and `show_subgrid`.
In subgrid mode, verbose reserves `PANE_LABEL_HEIGHT` at the bottom for the
label; compact/hidden reclaim that space for video content.

### Footer slimdown

Removed all keyboard hint text from the right side of the grid footer.
Removed grid dimensions (self-evident from looking at the grid). Footer now
shows only: play/pause icon, episode range (`ep 0-3 / 75`), and either camera
count (tiled) or camera name (flat single-cam).

### Shortcut bar (? key, Help menu)

A single-line `TopBottomPanel` below the menu bar, toggled by `?` or
`Help > Keyboard Shortcuts`. Hidden by default. Content is context-sensitive:

- **Single video**: `[G] grid [Space] play/pause [A/D] episode [M] multi-cam ...`
- **Flat grid**: `[G] exit [M] subgrid [N] tiled [C] camera ...`
- **Subgrid**: `[G] exit [M] single-cam [N] tiled ...`
- **Tiled**: `[G] exit [M] subgrid [N] untile ...`
- **MultiCamera**: `[M] exit [Space] play/pause [Esc] reset`

M/N hints reflect the actual target mode of the transition, not the current
mode (e.g. subgrid shows `[N] tiled` because N switches TO tiled).

State: `show_shortcut_bar: bool` on `App`.

### Help menu

New top-level `Help` menu:
- **Keyboard Shortcuts [?]**: toggles the shortcut bar
- **About...**: opens the About modal

### About modal

`src/about.rs`, following viewskater-egui pattern. Two `egui::Area` layers:

1. **Backdrop**: `Order::Foreground`, semi-transparent black, click-to-dismiss
2. **Card**: `Order::Tooltip` (renders above backdrop), centered, rounded frame
   with version, commit hash, platform, author, clickable GitHub link

Dismiss via Escape or clicking the backdrop. Uses existing `BuildInfo` and
`UiTheme` (backdrop, card_bg, card_stroke, accent, muted colors).

`show_about: bool` on `App`, rendered at the end of `update()` so it sits on
top of all other UI.

## Files changed

| File | Changes |
|------|---------|
| `src/grid.rs` | Grouped selection border logic in `show_flat`, `draw_pane_label` helper, `LabelMode` param threaded through `show`/`show_flat`/`show_subgrid` |
| `src/app.rs` | `LabelMode` enum, `show_shortcut_bar`/`show_about`/`label_mode` fields, about modal call |
| `src/ui/panels.rs` | `show_shortcut_bar` method, Help menu, footer slimdown |
| `src/ui/input.rs` | `L`/`?` key handlers |
| `src/about.rs` | New file, About modal |
| `src/build_info.rs` | Removed dead_code comment |
| `src/theme.rs` | Removed dead_code comment |
| `Cargo.toml` | Added `webbrowser` dep |
