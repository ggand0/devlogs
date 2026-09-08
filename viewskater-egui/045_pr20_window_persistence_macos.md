# PR #20: Window persistence, macOS zoom state

PR: https://github.com/ggand0/viewskater-egui/pull/20
Author: hml-pip. Branch: hml-pip:feat/window-state.
Status: contributor branch at 425aec4 plus our f493737, tested on macOS and
Ubuntu 2026-09-08. Owner posted the comment and marked the PR ready for review
the same day; waiting for hml-pip / YelovSK to test on Windows. #29 still open.

## What the PR does

Persists window size and position only. A quit while maximized, zoomed or
fullscreen relaunches as a normal window at the last normal geometry. eframe
persistence stays on for the restore side; `persist_window` is off and the app
writes eframe's `"window"` entry itself from geometry it records every frame
while the window is in its normal state (`src/window_state.rs`, `App::save`).

## The macOS bug

Green-button zoom, quit, relaunch: the window came back as a normal window
covering the whole screen. `app.ron` held the zoomed frame (1440x792 at
(0,50) on the 2560x1600 MacBook display at 2x).

Two causes, one behind the other.

1. egui-winit skips the runtime `is_maximized` query on macOS. The guard in
   `update_viewport_info` (egui#3494, winit#4071) exists because winit's
   `is_maximized` temporarily rewrites the style mask of borderless windows
   to make `isZoomed` work, and that floods egui with redraw requests.
   Decorated windows never hit the rewrite, but the guard is unconditional.
   So `ViewportInfo::maximized` stays at its init value and every zoomed
   frame looks normal to `capture()`.
2. Even when the query runs, `NSWindow.isZoomed` is a comparison of the
   current frame with the zoomed frame. It stays false for every frame of
   the ~350 ms zoom animation. `capture()` runs per frame, so the last
   animation frame (1429x788 at (8,54)) gets recorded as the normal
   geometry.

The iced version hit the same race in Feb 2026 (viewskater devlogs 005-007)
and solved it by deciding at save time with `isZoomed()`, which is
authoritative once the animation is over, and not writing the size when
zoomed. That worked there because iced kept a separate on-disk windowed
size from earlier save points. Here the per-frame capture already holds the
animation frame in memory by the time `save()` runs, so the decision has to
be made per frame.

## Attempts, in order

### egui-winit fork f574430: query maximized at runtime on decorated windows

Written on gota-home: `is_init || !macos || (decorated && resizable)`.
Correct for cause 1, and testing on the Mac confirmed it: capture now
stopped the frame `isZoomed` went true. But it stops one frame too late;
the saved geometry became the last animation frame instead of the final
one. Not pushed to the fork, superseded, can be dropped from the local
clones (one unpushed commit ahead of dd7862a on both machines).

### Settle timer (rejected)

Commit a captured geometry only after it has been unchanged for longer
than the animation. Rejected by the owner before implementation: a magic
duration, same class of heuristic as the 85%/70% screen thresholds the
iced version threw out in devlog 007.

### winit fork: implement `windowShouldZoom:toFrame:` in winit's delegate (rejected)

AppKit calls the window delegate's `windowShouldZoom:toFrame:` from `zoom:`
before the animation starts, for the green button, Option-click, title bar
double-click and un-zoom alike. winit's `WindowDelegate` does not
implement it. Adding it to the winit fork (ggand0/winit
custom-dnd-0.30.13) would be the textbook fix. Rejected by the owner:
the winit fork exists only for drag and drop and is not to be extended.

### Runtime subclass of winit's delegate (implemented, then rejected)

Committed as 75ec9ee and reverted the same day. Kept on branch
`backup/macos-zoom-delegate-patch` and in
`tmp/patches/0001-Track-macOS-zoom-transitions-for-window-persistence.patch`.

Same mechanism `platform/macos.rs` already uses on winit's application
delegate for Finder "Open With": `ClassBuilder` makes a runtime subclass of
winit's `WinitWindowDelegate`, `AnyObject::set_class` swaps the delegate
object's class to it, and four methods are added: `windowShouldZoom:toFrame:`
(new; sets `Transitioning`), `windowDidEndLiveResize:` (settles from
`isZoomed`), `windowDidResize:` and `windowDidMove:` (refresh when not
transitioning). Each override called winit's original through `super`. A
static `ZOOM_STATE` was read by `capture()`. It passed every test.

It is monkey patching: mutating another library's object at runtime. The
session presented it as "the pattern the file already uses" without naming
it as such, and the owner rejected it on that basis once it was clear.
Whoever debugs a resize event later would find winit's delegate is not
winit's class. The existing Finder swizzle in the same file is a precedent
for the technique, not a justification for a second instance.

What the tracing during this attempt did establish, and what made the final
fix possible: AppKit runs the zoom animation as a live resize.
`windowWillStartLiveResize:` fired 0.1 ms after `windowShouldZoom:toFrame:`
and before the first `windowDidResize:`; `windowDidEndLiveResize:` fired
351 ms later with the frame equal to the target and `isZoomed` true. My
earlier assumption that the animation was not a live resize is what had
ruled out the getter approach in the first place.

### Public getters on the NSWindow (shipped, f493737)

`capture()` takes `frame: &eframe::Frame` and, on macOS, returns `None`
when `window.isZoomed() || window.inLiveResize()`. The NSWindow comes from
`frame.window_handle()` (raw-window-handle, AppKit variant, `ns_view`) and
`NSView.window()`. 27 lines in `platform/macos.rs`, only getters. No
delegate, no class changes, no fork changes; `Cargo.toml` stays on the
egui-winit fork's main (dd7862a) and gains `NSView`/`NSWindow` features
plus `raw-window-handle`.

Per-frame timeline of a green-button zoom:

```
frame before click   isZoomed=false  inLiveResize=false  captured   900x600
zoom: begins         AppKit sets inLiveResize
animation frames     isZoomed=false  inLiveResize=true   skipped
landing frame        isZoomed=true   inLiveResize=true   skipped
zoom: done           inLiveResize cleared
zoomed               isZoomed=true   inLiveResize=false  skipped
```

Un-zoom is the mirror; the first frame after landing has both false and is
captured with the restored frame. A user drag is skipped while the mouse is
down and its final frame is captured after release. A half-screen tiled
window is a normal frame (not zoomed, no live resize once placed) and is
restored as-is, which is the intended behaviour.

Calling `NSWindow.isZoomed` directly never touches the style mask, so the
stall egui's guard protects against does not apply.

Documented vs observed: `isZoomed` is documented. `inLiveResize` is
documented for user drags; that it is also true for the whole zoom
animation is observed AppKit behaviour (10.6 onward), verified here, not a
written guarantee. The doc comment on `window_zoomed_or_resizing` says so.

Linux and Windows compile the check out; behaviour there is the PR's.

## Tests on macOS (Darwin 24.5, built-in 2560x1600 at 2x)

Scripted through the Accessibility API with no keystrokes (a `keystroke "q"`
goes to whichever app is frontmost; it quit the terminal twice before that
was replaced by an AX click on the app's own Quit menu item). Geometry via
AX window position/size; zoom via `AXZoomWindow` on the green button;
fullscreen via the `AXFullScreen` attribute.

| test | result |
|---|---|
| zoom, quit, inspect app.ron | inner 900x572, outer (400,300)px = pre-zoom 200,150 900x600 |
| relaunch | 200,150 900x600 |
| zoomed for 5 s, AX round trips + CPU | 0.1-0.3% CPU, responsive; un-zoom then quit saves 900x572 |
| fullscreen quit, relaunch | app.ron unchanged; normal window, AXFullScreen=false |
| resize+move 300,200 700x500, quit, relaunch | 700x472 @ (600,400)px; relaunch 300,200 700x500 |

Owner's manual pass with Cmd+Q: green-button zoom, title bar double-click,
zoom then un-zoom then move, fullscreen, drag-resize then quit at once,
and half-screen tiling. All restore as expected. clippy clean on touched
files; `cargo test` 6 passed, 1 ignored.

Inner height is outer minus the 28 pt title bar; positions are physical
pixels (2x), the `WindowSettings` convention eframe restores from.

## Ubuntu

Desktop session on gota-home, GNOME Shell on X11, two monitors: 2560x1440
primary at (0,0) and a 1080x1920 portrait secondary at (2560,0), pixels per
point 1.25. Built f493737 with `--locked`; clippy clean, `cargo test` 6
passed, 1 ignored. Scripted with xdotool/wmctrl for resize, maximize and
close, `xwininfo` and `xprop _NET_WM_STATE` for geometry and state, GNOME's
Super+Shift+Left/Right to change monitor (Mutter ignores client-side move
requests). Window at 900x600 physical before each quit.

| test | app.ron after quit | relaunch |
|---|---|---|
| maximize on secondary (1014x1883), quit | maximized:false, 720x480 pt, pos (2690,179) | 900x600 at (2690,179), not maximized |
| F11 fullscreen on secondary (1080x1920), quit | unchanged | 900x600 at (2690,179) |
| Super+Shift+Right then maximize, quit | pos (2690,179) | second monitor, 900x600 |
| Super+Shift+Left to primary, maximize (2494x1363), quit | pos (960,161) | 900x600 at (960,161) on primary |

Identical to the 425aec4 results earlier in the day, as expected: the macOS
check compiles out on Linux and nothing else in the capture path changed.

One X11 quirk, present before f493737 and shared with stock eframe
persistence: winit caches the frame extents and refreshes them only on a
reparent, so a session in which the window never got a configure after the
WM laid out its frame reports `outer_position == inner_position`. The next
launch then places the frame where the content was, a one-time shift down
by the title bar height (37 px here), after which it stays put. Seen twice
today, last after the primary-monitor run above. Fix would go in the winit
fork (invalidate the cache on configure events), not this PR.

Wayland not tested. There egui-winit reports no window position, so only
the size is persisted, and `capture()` falls back to `screen_rect` for it.

### Off-screen and oversized windows

Owner's question after the re-test: what happens when the window protrudes
past the screen edge, or is wider than the screen, at quit.

Position: saved as-is on every platform. On restore, Windows is the only
platform where eframe itself clamps (`clamp_position_to_monitors`, fits the
window on the monitor it was on, plus 32 px for the title bar), because a
window placed off-screen there becomes invisible. Linux and macOS pass the
position through and the window manager applies its rules. Measured on
GNOME: a window dragged 1100 px below the bottom edge and straddling both
monitors was saved at (2082,1358); Mutter mapped it fully on the monitor it
overlapped most, snapped to the edge, at (2626,712). Nothing gets lost.

Size: eframe clamps the saved inner size to the largest monitor's size in
points before creating the window (`clamp_size_to_sane_values`, comment in
eframe says a window larger than the monitor crashes some Linux systems),
and to a 64 px minimum. The window manager then fits the frame into its
work area, which is why a window that was wider than the screen comes back
a little narrower than the screen: the dock margin on Ubuntu (66 px here),
the menu bar and dock on macOS. Measured on GNOME the question does not
even reach persistence: a resize request of 3000 px on the 1080 px portrait
monitor was cut to 1014 by Mutter immediately, and that is what got saved.

Stock eframe persistence behaves identically in both cases; this PR only
changes what is written, not how it is restored. The size clamp can be
switched off with `ViewportBuilder::with_clamp_size_to_monitor_size(false)`;
left on, since an oversized window has no restore that is both faithful and
visible, and the clamp is the safe pick.

## State (end of 2026-09-08)

- `feat/window-state`: 425aec4 (hml-pip) + f493737 (ours), pushed to hml-pip,
  PR #20 marked ready for review, mergeable. Owner's comment posted, asks
  hml-pip and YelovSK to test on Windows.
- #29 open; closing comment drafted in
  `tmp/drafts/2026-09-08_pr20_pr29_window_persistence.md`, not posted yet.
- Windows is untested. The maximize-then-quit path should be fine since the
  window is never maximized before it is shown. The second-monitor case for a
  normal window is unchanged from June: the monitor lookup in egui-winit 0.31
  is the one YelovSK fixed in egui#8191, and mixed-DPI position conversion
  (eframe divides by the target monitor's scale, winit multiplies by the
  initial window's) is not handled upstream either.
- egui-winit fork main at dd7862a (constructor). f574430 on the local clone
  on gota-home is dead, not pushed.
- `backup/macos-zoom-delegate-patch` and `tmp/patches/0001-*.patch` on the
  Mac hold the rejected subclass version.
