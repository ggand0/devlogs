# File > Open Recent and the one open function

Date: 2026-10-09
Context: branch feat/recent-files off main fa95381. Not pushed.

| Commit | What |
|---|---|
| c2af2c7 | Add File > Open Recent |
| 827ffc5 | Shorten long paths in Open Recent |
| 87d5321 | Size Open Recent to its longest path |
 PR 1 of the three in
docs/plans/014_recent_rename_single_window.md (sections 2 and 3).
Handoff: tmp/handoffs/2026-10-09_recent_files_and_folders.md.

118 unit tests pass (106 on main plus 12 new), 5 ignored, clippy
clean with --all-targets. The owner ran the in-app test on Linux
after c2af2c7 and it works. 827ffc5 is not seen on screen yet (section
8). Not run on macOS or Windows.

Line numbers in sections 1 to 7 are as of c2af2c7.

## 1. What it does

File > Open Recent sits between Open File and the separator above
Close. It lists up to 10 files and folders, newest first, one row per
entry with the full path, then a separator and Clear. In single pane
a click opens the entry in pane 1. In dual pane each row is a submenu
with Pane 1 and Pane 2, like Open File. The button is disabled while
the list is empty. No shortcut.

An entry is the path as the user opened it: the file for a file open,
the folder for a folder open. Opening a path again moves it to the
top, without a second row.

## 2. Storage, src/recent.rs

`RecentPaths { paths: Vec<PathBuf> }` with `#[serde(transparent)]`,
so recent.yaml is a plain yaml list:

```
- /home/gota/ggando/rust_gui/data/debug_images_10
- /home/gota/ggando/rust_gui/data/debug_images_3
```

The file is `dirs::config_dir()/viewskater-egui/recent.yaml`, next to
settings.yaml. `load`, `load_from` and `save` copy the shape of
`AppSettings::load` and `save` (src/settings.rs 445, 454). A missing
file or one that does not parse as a list loads as empty.
`load_from` also truncates to `MAX_ENTRIES` (10) in case the file was
edited by hand.

`push` (63) removes an equal entry, inserts at the front and
truncates to 10. Paths are compared with `PathBuf`'s `==`, which
compares components. `/a/b` and `/a/b/` are equal, a symlink and its
target are not, and on Windows a case difference counts as a
different path. No canonicalize, so a slow network path costs
nothing here.

The list is not a field of `AppSettings`. settings.yaml is written
when Preferences changes, recent.yaml on every successful open.

## 3. App::open_in_pane, src/app/handlers.rs 69

Every user open goes through it:

| Call site | Where |
|---|---|
| launch paths, one per pane | `App::new` src/app.rs 401 |
| Open Folder dialog | `open_folder_dialog` |
| Open File dialog | `open_file_dialog` |
| Open Recent | `handle_menu_action` 126 |
| macOS open events | `handle_external_open_requests` 490 |
| drag and drop, both branches | `handle_dropped_files` 500 |

Order inside:

1. `pane_idx` out of range returns. This replaces the
   `panes.get_mut(pane_idx)` the dialogs had.
2. A path that does not exist shows the toast "Not found: <name>"
   (error style), removes the path from the list, saves and returns.
   The pane keeps what it had. `file_name` from src/app/culling.rs is
   now `pub(super)` for the name.
3. Otherwise `Pane::open_path` with `current_discovery_options()`. If
   the pane has images afterwards the path goes to the top of the list
   and the list is saved.
4. `ctx.request_repaint()`.

A folder without images is not recorded, because `Pane::open_path`
leaves `image_paths` empty for it.

`App::new` now pushes the empty second pane first when there are two
launch paths, then calls `open_in_pane` for each. The discovery
options are the same as before: `current_sort` equals the settings'
sort order at that point. The `perf.record_image_load()` lines stay
where they were.

These still call `Pane::open_path` directly: `set_dual_pane` copying
pane 1's folder, `reload_sorted_panes` after a sort change, and
src/app/bench.rs.

## 4. Benchmarks and the launch path

The handoff said keeping bench.rs on `Pane::open_path` is enough to
keep benchmarks out of the list. It is not. `--bench-nav <folder>`
passes the folder as the positional launch path, which goes through
`App::new` and so through `open_in_pane`. Seen in a smoke run: the
benchmark folder showed up in recent.yaml.

`open_in_pane` now reads `self.bench.opts.any()` and changes nothing
in the list during a `--bench-*` run, in both the record and the
missing-path branch. The toast still shows. Checked after the fix:
`--bench-nav` on debug_images_100 left recent.yaml unchanged.

Note for the next smoke run: `cargo test` and `cargo clippy` do not
rebuild target/opt-dev/viewskater-egui. Run `cargo build --profile
opt-dev` before launching the binary directly.

## 5. Menu, src/menu.rs 175 to 213

`MenuBarState` has `recent: &'a [PathBuf]`, filled from
`self.recent.paths()` in app.rs 870. The borrow is disjoint from the
`&mut self.settings` and `&mut self.current_sort` already in the
struct.

The whole Open Recent row is inside `ui.add_enabled_ui(!recent.is_empty(), ...)`,
which is how a submenu button is disabled in egui 0.31.1. Every row is
wrapped in `hover_row` with its own `setup_menu_hover`, the same as
Open File. In dual pane the per-entry submenu loops over
`[(0, "Pane 1"), (1, "Pane 2")]`.

`MenuAction::OpenRecent(usize, PathBuf)` and `MenuAction::ClearRecent`.
The menu never stats an entry, so a gone network path cannot stall the
menu bar. The check happens on the click.

## 6. Unit tests, src/recent.rs

- `push_puts_new_path_first`
- `push_moves_existing_path_to_front_without_duplicate`
- `list_stops_at_max_entries`: 13 pushes, 10 kept, the newest first,
  dir3 last
- `remove_drops_only_that_path`, including a path not in the list
- `clear_empties_list`
- `yaml_round_trip`: also pins the file format to a plain list
- `missing_file_loads_as_empty`, `unreadable_file_loads_as_empty`
  (tempfile)

## 7. Open after this

- macOS and Windows not run yet. On macOS the open event during launch
  now records the path too, through `handle_external_open_requests`.
- PR 2 (window setting) and PR 3 (rename) call `open_in_pane`. Rename
  needs a `RecentPaths` method that replaces the old path with the new
  one in place (plan 014 section 4).

## 8. Row labels, 827ffc5

How other apps show the entries (from memory, not looked up): macOS
document apps show names only and add the folder when two names
match, VS Code shows the full path with `~` for home and no cut, GIMP
shows names with the path on hover, IrfanView numbered full paths.
Full paths stay, because photo folders are often named DCIM, 100MSDCF
or 2026-10 and only make sense with their parents.

What egui 0.31.1 did with a long path at c2af2c7: a menu opens
`spacing.menu_width` wide, 400 px (egui src/menu.rs 186, style.rs
1260), and a button in a vertical layout wraps (`Ui::wrap_mode`, ui.rs
688). So a path over about 60 characters wrapped onto a second line.
The owner's test paths were shorter.

Now, in src/menu.rs:

- `recent_label` writes the home folder as `~` on Linux and macOS,
  through `dirs::home_dir` and `strip_prefix`. Windows keeps the full
  path, like VS Code there.
- The Open Recent submenu sets `wrap_mode = Extend`, so every row is
  one line and the menu is as wide as the longest row.
- egui keeps a popup inside the window by moving it left (`Area`
  constrains to `ctx.screen_rect()`). So the limit for a row is the
  window width, not the space right of the File menu.
  `recent_label_max_width` is the window width minus the menu margin,
  the frame stroke, the button padding and, in dual pane, the submenu
  arrow (`item_spacing.x + icon_width`).
- `elide_middle` cuts a label wider than that. It first drops
  characters from the end of the part before the last separator
  (`std::path::is_separator`), so the start and the name stay, for
  example `/mnt/nas/p…/DSC00116.jpg`. When `…` plus the name still
  does not fit, it drops characters from the start of the name. Widths
  are measured with `layout_no_wrap` in the button font.
- A cut row gets the full label as hover text. Rows that fit have no
  hover text.
- Clear is now Clear Recent.

Tests: `elide_middle_keeps_text_that_fits`,
`elide_middle_cuts_the_start_and_keeps_the_name`,
`elide_middle_cuts_the_name_last` (with a width of one per character),
and `recent_label_writes_home_as_tilde` (not on Windows).

Not seen on screen. The xdotool attempt opened the File menu of the
owner's own ViewSkater window first (same title, same position).
After that GNOME ignored `windowmove` and `windowactivate` for the
test window, so the check stopped and only the test instance was
killed. Next time, find the test window with `xdotool search --pid`
before any click.

## 9. The submenu stayed wide, 87d5321

The owner saw Open Recent as wide as the window with only short paths
in it. Cause in egui 0.31.1: `Area::show` keeps the area's size in
`AreaState::size` (containers/area.rs 417, set in `end` at 595 from
`content_ui.min_size()`), and `Areas::end_pass` (memory/mod.rs 1306)
never forgets an area's state. The next frame's `max_rect` is that
size, and the buttons in the menu's `top_down_justified` layout
stretch to fill it, so `min_size` never gets smaller. One wide row,
here the deep scratchpad folder from the section 8 test cut to the
window width, kept the menu that wide for the rest of the run, also
after Clear Recent.

Fix: the submenu measures every label first, then calls
`ui.set_width(widest + row_padding)` before drawing the rows, with
"Clear Recent" counted as a label. `recent_row_padding` (button
padding plus the dual pane arrow) is now its own function and
`recent_label_max_width` takes it. Not seen on screen by me, the
owner checks it.
