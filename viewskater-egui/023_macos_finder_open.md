# 023 — macOS "Open With" / Finder double-click

Port of the iced version's Finder integration to the egui/eframe build. Users
can now right-click an image in Finder, pick ViewSkater from "Open With", or
double-click after setting it as the default viewer, and the app launches
directly into that image.

## What ships

- `resources/macos/Info.plist` — `CFBundleDocumentTypes` for every extension
  in `file_io::SUPPORTED_EXTENSIONS` (jpg, jpeg, png, bmp, webp, gif, tiff,
  tif, qoi, tga). All entries use `LSHandlerRank = Alternate` so we don't
  steal the default from Preview.app.
- `src/platform/macos.rs` — NSApplicationDelegate swizzle that adds
  `application:openFiles:` and `application:openFile:` methods, pushing
  resolved paths through an `mpsc::Sender<PathBuf>`.
- `src/app/handlers.rs::handle_external_open_requests` — drains the receiver
  each frame and calls `Pane::open_path`, the same entrypoint used by CLI
  args and drag-and-drop.
- `scripts/bundle-macos.sh` — wrapper around `cargo bundle` that merges the
  document-type plist via `PlistBuddy -c Merge` (cargo-bundle 0.6.1 silently
  ignores any plist-extension key).

No sandbox work: we skipped the iced version's security-scoped-bookmark
module (~1200 lines) since this build doesn't target the App Store. If
sandboxing is ever needed, that code still lives at
`../viewskater/src/macos_file_access.rs`.

## Design

```
Finder → AppleEvent (kAEOpenDocuments)
           ↓
NSApplicationDelegate.application:openFiles:   ← we swizzle this in
           ↓
mpsc::Sender<PathBuf>
           ↓ (next egui frame)
App::handle_external_open_requests → Pane::open_path
```

`Pane::open_path` is the single pre-existing entrypoint that CLI args, the
file dialog, and drag-and-drop all funnel through. No new message variant or
state machine — the channel is drained in the normal `update()` tick.

## The timing problem

The obvious place to install the swizzle is eframe's `CreationContext`
callback (`App::new`), because that's where we have the egui context. That
works for the "already running" case: if the app is open and the user picks
another image in Finder, the new `openFiles:` dispatch hits our swizzled
method, forwards the path, and the next frame loads it.

It does **not** work for the initial-launch case. macOS queues the AppleEvent
when Finder launches the bundle, and AppKit dispatches it during
`-[NSApplication finishLaunching]` — specifically between
`applicationWillFinishLaunching:` and the first iteration of the run loop.
Apple's documentation implies openFiles: fires *after*
`applicationDidFinishLaunching:`, but empirically it fires before
winit's `dispatch_init_events()` runs, and therefore before eframe invokes
our creation callback.

Concrete order observed on macOS 15:

```
main()
eframe::run_native
  winit EventLoop::new     — creates WinitApplicationDelegate, setDelegate
  winit EventLoop::run     — calls [NSApp run]
    [NSApp finishLaunching]
      applicationWillFinishLaunching:          ← our observer fires here
      (AppKit installs default AE handlers)
      dispatch kAEOpenDocuments → delegate.openFiles:   ← TOO LATE if we hadn't hooked yet
      applicationDidFinishLaunching:
        winit.did_finish_launching
          winit.dispatch_init_events
            Event::Resumed → eframe creation closure → App::new   ← what we previously used
```

If the swizzle goes in at App::new, the initial `openFiles:` has already
fired against an unswizzled `WinitApplicationDelegate` and AppKit shows
"ViewSkater cannot open files in the PNG image format."

## Fix

Install the swizzle from an `NSApplicationWillFinishLaunchingNotification`
observer, registered **before** `eframe::run_native`. By the time that
notification fires, winit has already called `setDelegate:`, so
`NSApplication.delegate` returns the `WinitApplicationDelegate` instance and
we can rewrite its `isa` pointer via `ClassBuilder`/`AnyObject::set_class`.

`src/platform/macos.rs::install_launch_observer` does the registration. It
allocates a tiny `ViewSkaterLaunchObserver` class whose single method calls
`swizzle_delegate()`, and leaks the instance so NSNotificationCenter's weak
reference stays valid for the process lifetime.

`register_file_handler()` (called later from `App::new`) is kept as a
belt-and-braces fallback for the case where the notification name ever
changes; both entrypoints feed the same idempotent `swizzle_delegate()`.

## cargo-bundle's plist blind spot

cargo-bundle 0.6.1 writes `Info.plist` from scratch from a small set of
`[package.metadata.bundle]` keys. Document types, URL schemes beyond the
single built-in field, UT declarations — all ignored, no extension hook.

Rather than fork cargo-bundle or move to cargo-packager, we ship a tiny
shell script (`scripts/bundle-macos.sh`) that runs `cargo bundle` and then
invokes `PlistBuddy -c "Merge resources/macos/Info.plist"` on the generated
plist. PlistBuddy's Merge command deep-merges top-level keys, which is
exactly the behavior we want: cargo-bundle owns the boilerplate,
`resources/macos/Info.plist` owns the document types.

Usage:

```
scripts/bundle-macos.sh             # opt-dev profile
scripts/bundle-macos.sh release     # release profile
```

## Registering with LaunchServices

A freshly-bundled app is invisible to Finder's "Open With" list until it's
registered with LaunchServices:

```
/System/Library/Frameworks/CoreServices.framework/Frameworks/\
  LaunchServices.framework/Support/lsregister -f \
  target/opt-dev/bundle/osx/ViewSkater.app
```

Shipping the .app via DMG/installer does this automatically on first run.
For local testing, re-run lsregister any time `Info.plist` changes.

## TL;DR for PR review

- **Swizzle.** Same ISA-swizzle as the iced version, copied nearly verbatim
  from `viewskater/src/macos_file_access.rs`. Subclass winit's delegate at
  runtime, attach `openFiles:`/`openFile:`, rewrite the delegate instance's
  class pointer. Winit's own methods still work (inheritance).
- **Timing difference vs. iced.** Iced can install the swizzle straight from
  `main()`. Eframe can't, because winit creates the NSApplication deep inside
  `run_native`. We hook an `NSApplicationWillFinishLaunchingNotification`
  observer *before* `run_native`; the observer fires at the one moment where
  the delegate exists and AppKit hasn't yet dispatched the initial openFiles:
  event. Not fragile — if the timing ever drifts, the failure is a loud
  "cannot open PNG" dialog, not a silent regression.
- **Info.plist.** `cargo bundle` auto-generates the full plist from
  `Cargo.toml`, but it has no way to express document types. We keep a
  fragment at `resources/macos/Info.plist` containing only the file-type
  entries; `scripts/bundle-macos.sh` runs `cargo bundle` and then uses
  Apple's `PlistBuddy -c Merge` to splice the fragment into the generated
  plist. Goes away if we switch to a bundler that supports plist extensions.
- **`LSHandlerRank = Alternate`.** We show up in Finder's "Open With"
  submenu without claiming the default opener for any type — Preview stays
  the default. Users opt in per file.
- **`lsregister -f` is dev-only.** Never called at runtime. End-user
  installs (DMG, Homebrew cask) trigger the LaunchServices scan
  automatically; the command just forces it to happen immediately during
  local iteration.

## Testing

- Initial launch: `pkill -f viewskater-egui; open -a ViewSkater.app foo.png`
  Image loads, no error dialog.
- Already running: open the app, then from a second terminal
  `open -a ViewSkater.app bar.png`. The running instance receives the new
  path and switches to it.
- Drag-and-drop onto the app icon in the Dock: same code path as "Open
  With".
