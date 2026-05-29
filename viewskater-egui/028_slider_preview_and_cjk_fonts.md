# 028 — Slider preview & CJK font support

PR #18 by hml-pip adds a thumbnail preview when hovering the navigation slider.
This entry documents the review discussion and research into CJK font rendering.

## PR #18 overview

The PR adds a slider hover preview: when the cursor hovers over the nav slider,
a thumbnail of the image at that position is shown. Implementation touches three
areas:

- **`src/decode.rs`** — new `image_to_thumbnail()` that downscales via
  `FilterType::Triangle` to `THUMBNAIL_WIDTH x THUMBNAIL_HEIGHT` (300x200).
- **`src/cache.rs`** — `SlidingWindowCache` gains a `thumbnails: HashMap` that
  stores thumbnail textures. Thumbnails are generated alongside full decodes
  (both sync and background). A `current_thumbnail_for()` method returns a
  cached thumbnail or generates one synchronously on miss. `decode_sync` return
  type changes from `Option<TextureHandle>` to a tuple including the thumbnail.
- **`src/app.rs`** — `paint_nav_slider` takes `&mut [Pane]`, uses
  `response.on_hover_ui_at_pointer()` to render the thumbnail tooltip at cursor
  position.

## Review decisions

1. **Dual-pane support** — Not needed. Dual-pane is for comparing two images;
   a sliding preview in that mode would be visually noisy.

2. **SlidingWindowCache integration** — The contributor flagged conflicts with
   the cache design. Given decision (1), this is less of a concern but the
   thumbnail HashMap keying (using `file_index` vs `slot_idx` inconsistently)
   needs cleanup.

3. **CJK font support** — See dedicated section below.

4. **Preview positioning** — The current `on_hover_ui_at_pointer` makes the
   preview follow the cursor, which is jittery. Plan: render at a fixed Y
   offset above the slider rect, with X tracking the hover position. Similar to
   how video players show frame previews above the scrub bar.

## CJK font research

### The problem

egui ships only ASCII/Latin fonts. Non-Latin filenames (CJK, Cyrillic, emoji)
render as missing-glyph boxes. The contributor (who uses the app as a manga
viewer) wanted to show filenames in the preview but dropped it because of this.
egui issue #3060 confirms this is by-design — users must load their own fonts.

iced has the same fundamental issue. It uses `cosmic-text` which theoretically
supports system font fallback, but CJK rendering is buggy in practice.

### egui's font system

egui supports fallback chains natively. `FontDefinitions` maps font families
(Proportional, Monospace) to ordered lists of font names. When rendering, egui
walks the chain until it finds a matching glyph. The problem is just that egui
does not load system fonts automatically. There is an open PR by emilk (#5784)
for automatic system font loading but it has not been merged.

### Solutions evaluated

| Approach | Binary impact | Code | Dep maturity |
|---|---|---|---|
| `egui-system-fonts` | None (runtime) | ~2 lines | Very new, ~500 downloads |
| `egui-cjk-font` | None (runtime) | ~2 lines | ~66 downloads |
| DIY + `font-kit` | None (runtime) | ~20 lines | High (Servo, 15M+ downloads) |
| Embed font files | +10-20 MB | ~5 lines | N/A |

### `egui-system-fonts` deep dive

A thin adapter (186 lines) over the `system-fonts` crate (307 lines):

- **Font discovery**: Uses `fontdb` (not `font-kit`) — scans standard OS font
  directories rather than calling platform APIs (CoreText, DirectWrite). Lighter
  but can miss fonts outside standard paths.
- **Locale detection**: Uses `sys-locale` crate. Maps locale prefixes to
  regions (ko -> Korean, ja -> Japanese, zh -> Chinese, etc.). Many scripts
  (Arabic, Thai, Hindi) fall through to Latin/Unknown.
- **Font selection**: Hardcoded lists of well-known family names per region.
  E.g., Japanese Sans tries: Noto Sans JP, Yu Gothic, Hiragino Sans, Meiryo.
- **API**: `add_auto(ctx, style)` appends system fonts as lowest-priority
  fallback. `set_auto(ctx, style)` replaces all fonts entirely.
- **Maturity**: Created Jan 2026, 0 GitHub stars, ~500 crates.io downloads.

### `font-kit` DIY approach

`font-kit` (by Servo team) uses actual platform APIs — CoreText on macOS,
DirectWrite on Windows, fontconfig on Linux. More robust font discovery. The
code to integrate is ~20 lines:

1. Define a candidate list of CJK family names per platform.
2. Query `SystemSource` for the first match.
3. Load the font bytes and insert into egui's Proportional family at lowest
   priority via `ctx.add_font()`.

### Recommendation (revised)

Use the Sentrux-style approach: unconditionally load CJK system fonts at startup
by trying hardcoded platform-specific paths, appending as lowest-priority
fallback in egui's Proportional family. No locale detection — egui's per-
character fallback chain handles mixed Latin/CJK strings automatically. The
locale-based approach (`egui-system-fonts`) is a liability for a file viewer
where the user's locale has no relation to the filenames they're viewing.

Use `font-kit` for discovery instead of raw `std::fs::read` for more robust
cross-platform font resolution via platform APIs (CoreText, DirectWrite,
fontconfig). Make it configurable (default on) with a setting to disable,
similar to Sentrux's `load_cjk_fonts` toggle.

## Community best practices survey

Research into how real egui/iced apps handle CJK fonts in practice:

### What people actually do

| Approach | Frequency | Used by |
|---|---|---|
| Load system fonts by hardcoded paths | Most common | Servo, Sentrux, idevice_pair |
| Load system fonts via font-kit/fontdb by name | Common | Oculante, Ruffle |
| Embed via `include_bytes!` | Less common | Horizon, Japanese tutorials |
| Download on first run + cache | Rare | Talon |
| Conditional with user toggle | Very rare | Sentrux |
| On-demand/lazy by glyph detection | Nobody | (requested: egui #5233) |
| Load all system fonts | Nobody | (causes perf issues: iced #2455) |

### Notable implementations

**Sentrux** (`sentrux-core/src/app/update_loop.rs`) — the only app found with a
user-facing toggle. A `settings.load_cjk_fonts` bool (default `true`) plus a
`SENTRUX_NO_CJK` env var override. The `setup_fonts()` function tries hardcoded
platform-specific paths (PingFang.ttc on macOS, NotoSansCJK on Linux,
msyh.ttc on Windows) and loads the first found as a fallback. When disabled it
logs: "CJK font loading disabled — saving 10-30MB memory".

**Oculante** (image viewer, `src/ui/theme.rs`) — uses `font-kit` to find fonts
by family name. Builds a region-to-font-families map and tries each candidate
until one resolves. Appends as fallback (not prepended).

**Ruffle** (Flash emulator, `desktop/src/gui/controller.rs`) — locale-aware
priority system. Detects locale via `unic_langid`, adjusts CJK font priority
based on whether locale is `ja` etc. Queries `fontdb` by family name. Covers
Hebrew, Arabic, Thai in addition to CJK.

**Servo** (`ports/servoshell/desktop/gui.rs`) — `#[cfg(target_os)]` with
hardcoded font paths per platform.

### The locale problem

Locale-based detection (used by `egui-system-fonts`, Ruffle) is wrong for our
use case. A US-locale user viewing manga with Japanese filenames would not get
CJK fonts loaded. The correct approach for a file viewer is to always load CJK
fallback fonts regardless of locale, with a user toggle to disable.

### Key technical notes

- **Memory**: CJK fonts are 10-20MB each in memory. egui PR #5276 (merged Nov
  2024) wrapped font data in `Arc` so clones are cheap.
- **Performance**: CJK fonts can make egui slow in debug builds (issue #962).
  CJK fonts disable subpixel binning to save atlas space.
- **TTC collections**: Many CJK fonts are `.ttc` files with multiple faces.
  egui supports this via `FontData::index`.
- **Loading all system fonts is bad**: iced tried this and a user with 7GB of
  fonts had catastrophically slow startup (iced #2455).

### Plan for viewskater-egui

1. At startup, use `font-kit` to query the system for known CJK font families
   (PingFang SC, Hiragino Sans, Yu Gothic, Microsoft YaHei, Noto Sans CJK,
   etc.) unconditionally — no locale detection.
2. Append found fonts as lowest-priority fallback in egui's Proportional family.
   egui's per-character fallback chain then handles mixed Latin/CJK filenames
   automatically (Latin glyphs use default font, CJK glyphs fall through to the
   loaded CJK font).
3. Make it configurable — default on, with a setting to disable for users who
   want to save ~10-20MB memory.
4. This approach follows Sentrux's pattern but uses `font-kit` for more robust
   cross-platform font discovery.

## macOS slider preview delay (~1s to appear)

### Symptom

On macOS, the slider thumbnail preview takes ~1 second to appear on hover,
regardless of image resolution. On Linux (desktop, X11) it appears instantly.

### Root cause: egui tooltip delay + macOS pointer event frequency

The PR uses `response.on_hover_ui_at_pointer()` which internally goes through
egui's tooltip pipeline. In egui 0.31, `should_show_hover_ui()` (response.rs)
gates tooltip rendering behind two checks:

1. **`show_tooltips_only_when_still: true`** (default) — the tooltip is
   suppressed until `pointer.is_still()` returns true. egui checks whether the
   pointer moved between the last two frames.
2. **`tooltip_delay: 0.5`** (default) — after the pointer is deemed "still",
   an additional 0.5 second timer must elapse before rendering.

macOS reports pointer events at very high frequency (120-240Hz, tied to display
refresh). Even sub-pixel micro-movements from hand tremor are reported as motion
events, so `is_still()` rarely returns true. The user must hold the mouse
genuinely motionless for the stillness check to pass, then wait the 0.5s delay
— totaling ~1 second in practice.

On Linux X11, pointer events are coarser and batched at a lower rate. Small
jitter is filtered out, so `is_still()` becomes true almost immediately, making
the 0.5s delay the only wait (and it feels negligible because the stillness
check passes quickly).

Additionally, the tooltip system suppresses rendering entirely during drag
(`pointer.any_down() && has_moved_too_much_for_a_click` → return false), so the
preview would never show while dragging the slider — though the contributor's
current code only uses `hover_pos()` not `interact_pointer_pos()`, so this
doesn't affect the current UX.

The relevant egui code path (egui 0.31, `response.rs:748-769`):

```rust
if style.interaction.show_tooltips_only_when_still {
    if !self.ctx.input(|i| i.pointer.is_still() && ...) {
        self.ctx.request_repaint();
        return false;  // suppressed until mouse is "still"
    }
}
let time_til_tooltip = tooltip_delay - time_since_last_interaction;
if 0.0 < time_til_tooltip {
    self.ctx.request_repaint_after_secs(time_til_tooltip);
    return false;  // suppressed until delay elapses
}
```

### Fix

The contributor independently fixed the delay by switching from
`on_hover_ui_at_pointer` to `show_tooltip_at` in his force-push (see section
below). However, both `show_tooltip_at` and `egui::Area` caused a separate
flickering issue on macOS (also documented below). The final fix uses
`ctx.layer_painter()` to paint the preview directly — no widget-level APIs,
no hit-testing interference.

## Thumbnail cache key mismatch in `initialize()`

### Bug

In `SlidingWindowCache::initialize()`, the center image's thumbnail was stored
with a slot-relative key instead of the absolute file index:

```rust
let center_slot = center_index - self.first_file_index;
// slots use window-relative indexing — correct:
self.slots[center_slot] = Some(tex);
// thumbnails HashMap uses absolute file index — BUG: used center_slot
self.thumbnails.insert(center_slot, Some(thumb));  // e.g. key=5
```

But `current_thumbnail_for()` and `poll()` both look up thumbnails by absolute
file index (e.g., key=50). So the center image's thumbnail from `initialize()`
is stored at the wrong key and never found, causing a fallback to
`generate_thumbnail_sync()` — a redundant full `image::open()` + resize on the
UI thread.

### Scope

Only affects `initialize()` and `jump_to()` (which calls `initialize()`).
Background-decoded images go through `poll()` which correctly uses `file_index`
as the HashMap key. So normal sliding through cached neighbors was unaffected.

After the first cache miss, `current_thumbnail_for()` re-caches the thumbnail
at the correct absolute key, so subsequent hovers on the same index are fine.
The bug caused one redundant sync decode per `initialize()`/`jump_to()` call.

### Fix

```rust
self.thumbnails.insert(center_index, Some(thumb));  // was center_slot
```

## Contributor force-push: `show_tooltip_at` and dynamic sizing

After the initial review, the contributor force-pushed with a rebased history.
Key changes:

- **Tooltip API**: Replaced `on_hover_ui_at_pointer()` with
  `egui::show_tooltip_at()`. This also bypasses the tooltip delay because
  `show_tooltip_at` calls `show_tooltip_at_dyn` directly — it never goes
  through `Response::should_show_hover_ui()` where the stillness check and
  delay timer live. So he independently fixed the macOS delay issue.

- **Dynamic sizing**: Preview dimensions now scale with window size via
  `SCREEN_PREVIEW_UI_RATIO` (window size / 5, floored at THUMBNAIL_WIDTH x
  THUMBNAIL_HEIGHT). `screen_rect()` in egui returns the window rect, not the
  physical monitor.

- **CJK font support**: Hardcoded platform-specific font paths
  (PingFang on macOS, NotoSansCJK on Linux, msyh on Windows) loaded at startup
  as fallback fonts.

- **Rebased onto PR #19**: His branch now includes the mouse-wheel-zoom merge,
  which moved scroll-to-navigate handling to the app level.

## macOS preview flickering with `show_tooltip_at` / `egui::Area`

### Symptom

On macOS, the slider preview flickers — rapidly appearing and disappearing —
when the cursor hovers over the slider and is held still. Intermittent:
sometimes the preview is stable, sometimes it flashes. Does not reproduce on
Linux. Not present in the pre-force-push code (which used
`on_hover_ui_at_pointer` and had the delay issue instead).

### Root cause: widget-based rendering interferes with hit-testing

Both `show_tooltip_at` and a manual `egui::Area` register widgets in egui's
hit-testing system, which causes a frame-alternation cycle:

**Background: egui's hit-testing pipeline**

egui is immediate mode — widgets don't persist between frames. To determine
hover/click/drag state, egui records all widget rects during frame N, then at
the start of frame N+1 runs hit-testing against those saved rects. The
hit-testing walk (`hit_test.rs`) iterates widgets sorted by layer order
(front-to-back), checks spatial overlap with the pointer, and determines which
widget gets `hovered`, `drag`, and `click` status. Higher-order layers
(`Order::Tooltip`) are evaluated before lower layers (`Order::Background`).

**The flickering cycle**

- **Frame N**: Only the slider widget exists from the previous frame. Hit-test
  → slider is hovered → `hover_pos()` returns `Some` → preview rendering code
  runs → a widget is registered on the tooltip layer (either via
  `show_tooltip_at`'s internal `Area`, or our explicit `egui::Area`).

- **Frame N+1**: Hit-testing sees TWO widgets near the pointer: the slider
  (background layer) and the preview widget (tooltip layer). The tooltip layer
  is higher-order, so it's evaluated first. The preview widget's large
  `interact_rect` (300-400px tall, positioned close to the slider) can cause
  egui's layer-inclusion logic (`hit_test.rs:115-124`) to exclude the slider's
  layer, or alter which widget wins `hits.drag`. The slider loses its `hovered`
  flag → `hover_pos()` returns `None` → preview code doesn't run → no preview
  widget registered for next frame.

- **Frame N+2**: Only the slider exists again (preview widget wasn't rendered
  last frame) → slider is hovered → preview renders → widget registered.

- **Frame N+3**: Same as Frame N+1. Visible as rapid flashing.

The intermittency depends on exact pointer position, window size, and dynamic
preview sizing — all of which affect whether the preview's `interact_rect`
overlaps the hit-testing search area around the pointer.

**Why `egui::Area` with `.interactable(false)` didn't help**

`.interactable(false)` only changes the Area's default sense from
`Sense::click()` to `Sense::hover()` (`area.rs:458-465`). The Area still calls
`ctx.create_widget(WidgetRect { ... })` with that sense, so the widget rect is
still registered in the hit-testing system. A `Sense::hover()` widget doesn't
appear in `hits.click` or `hits.drag`, but its presence on the tooltip layer
still affects the layer-inclusion walk that determines which lower-layer
widgets are reachable.

**Why the pre-force-push code didn't flicker**

The pre-force-push code used `on_hover_ui_at_pointer`, which gates rendering
behind `should_show_hover_ui()`. The tooltip delay meant the preview rarely
appeared at all on macOS (the delay issue), so the hit-testing interference
cycle never got started. The flickering was latent but masked by the delay bug.

### Fix: `layer_painter` — pure paint, no widgets

`ctx.layer_painter(layer_id)` returns a `Painter` bound to a specific render
layer. It emits raw draw commands (`painter.rect()`, `painter.image()`,
`painter.galley()`) directly into the render queue — no layout, no widget
registration, no `create_widget()` call. The preview is visually present on
the tooltip layer but completely invisible to the hit-testing pipeline.

The tradeoff is fully manual positioning: we read `style.visuals.window_fill()`,
`window_stroke()`, `menu_corner_radius`, and `spacing.menu_margin` to match
the default popup appearance, then compute frame rect, content offsets, image
centering, and label position by hand.

```rust
let layer_id = egui::LayerId::new(
    egui::Order::Tooltip, egui::Id::new("slider_preview"),
);
let painter = ui.ctx().layer_painter(layer_id);
painter.rect(frame_rect, corner_radius, fill, stroke, StrokeKind::Outside);
painter.image(tex.id(), img_rect, uv, egui::Color32::WHITE);
painter.galley(label_pos, label_galley, text_color);
```

This is the lowest-level drawing API in egui. It sits below Area, Frame, and
the widget system. No widget rects are registered, so the slider's hover state
is never disturbed regardless of preview size or position.

## Post-87a2708 review (2026-05-23)

The contributor pushed three more commits after the initial review:

- `da086e8` -- Fixed macOS flickering by switching to `layer_painter` (see
  section above).
- `d3e9f10` -- Fixed the thumbnail keying bug (`center_slot` -> `center_index`),
  added `thumbnails.clear()` on cache reset, removed the min size floor for
  preview scaling, hides preview while dragging (`!response.dragged()`).
- `87a2708` -- Replaced the synchronous thumbnail loading in
  `current_thumbnail_for` with an async system: separate `mpsc` channel, `in_gen`
  HashSet, `pending_generates` VecDeque, and a spinner rendered while waiting.
  Motivation: the contributor is working on compressed file support and wanted
  async loading for large images.

### CJK font loading is needed

Initially assumed CJK font loading was unused because the tooltip only shows the
image index. Testing with CJK-named images confirmed the footer
(`src/menu.rs:377`, `paint_pane_footer`) displays filenames via egui text
rendering, which shows missing-glyph squares without CJK fonts loaded. The
window title bar renders CJK correctly without the loading code because it uses
OS-level text rendering, not egui.

### Async thumbnail loading (87a2708) is a regression

The sync approach in `d3e9f10` rendered thumbnails instantly -- decode and return
in the same frame. The async approach in `87a2708` returns `None` on cache miss
and shows a spinner. When scrubbing the slider, every new position is a miss, so
the spinner flashes constantly. The sync version had no perceptible lag even on
4K images since thumbnails are small and decode fast.

Two additional bugs in the async code:
- `spawn_thumbnail_thread` never calls `ctx.request_repaint()` after sending the
  result, so completed thumbnails sit in the channel until something else
  triggers a repaint.
- `poll()`'s repaint condition only checks `pending_uploads` and
  `pending_decodes`, not `in_gen` or `pending_generates`, so the repaint loop
  stops while thumbnail work is still in-flight.
- `poll()` line 204 calls `self.spawn_thread` (full-image decode) instead of
  `self.spawn_thumbnail_thread` when draining `pending_generates`. Pending
  thumbnail requests were spawning full-image decode threads, decoding at full
  resolution and sending results through the wrong channel.

### Performance regression: keyboard nav fps drops from >60 to 30-35

Tested on Linux, NVMe SSD, 24-core CPU, 4K images.

**Root cause 1: `img.clone()` in decode paths**

Both `decode_sync` and the background `spawn_load` thread cloned the full
`DynamicImage` before converting -- one copy for `image_to_color_image`, one for
`image_to_thumbnail`. For 4K images that's ~32MB per clone. Fix: change
`image_to_thumbnail` to take `&DynamicImage`, generate thumbnail by reference
first, then pass ownership to `image_to_color_image`.

**Root cause 2: thumbnail generation piggybacked on every full-image decode**

Every background decode and `decode_sync` call generated a thumbnail alongside
the full image, even during keyboard navigation when the preview isn't visible.
This added unnecessary work to every decode in the pipeline. Fix: removed
thumbnail generation from `decode_sync`, `spawn_load`, `DecodeResult`, and
`poll()` upload path. Thumbnails are now only generated on-demand when the user
hovers the slider.

After these two fixes, keyboard-only navigation returned to >60fps matching main.

**Root cause 3: `current_thumbnail_for` causes fps drop during hover + keyboard nav**

With the preview disabled entirely (hover block commented out), fps stays at 60
during keyboard nav. With the preview enabled but `current_thumbnail_for`
stubbed to return `None` (no thumbnail loading, just the frame/spinner drawing),
fps stays at 60. So the bottleneck is `current_thumbnail_for` itself.

The async path spawns a new OS thread (`std::thread::spawn`) for every cache
miss. During keyboard navigation while hovering, `cursor_index` changes every
frame, and each new index is a miss. This means dozens of thread spawns per
second, each doing a full `image::open()` on a 4K image. The thread spawn
overhead is the problem.

The sync path (`d3e9f10`) didn't have this issue because it blocked and returned
in one call -- no thread management overhead.

### Fix plan: persistent thumbnail worker thread

Replace `std::thread::spawn` per thumbnail with a single persistent worker
thread that receives requests via a channel. When a new hover position arrives,
the worker processes the latest request. This eliminates per-frame thread spawn
overhead and avoids queuing up stale requests for positions the user has already
moved past.

### Hybrid sync/async thumbnail loading (2026-05-24)

After extensive testing, settled on a hybrid approach:

- **Sync for files <= 20MB**: `current_thumbnail_for` calls `image::open()` +
  `image_to_thumbnail()` directly on the main thread. Returns the correct
  thumbnail in the same frame. Tested with 10MB 4K PNGs -- no perceptible
  lag even during fast hover scrubbing.
- **Async for files > 20MB**: sends request to a persistent worker thread.
  Returns the previous thumbnail as a placeholder until the new one is ready.
  No spinner, no empty frame.

**File size threshold**: measured decode times using PIL as a rough proxy:
- 10MB 4K PNG: ~150ms -- feels instant to the user
- 98MB PNG: ~2000ms -- would block the UI unacceptably with sync

20MB threshold covers all typical images (JPEGs, standard PNGs) with sync while
pushing giant PNGs and future archived/compressed files to async.

**Previous frame display**: when async returns `None` for the requested index,
`current_thumbnail_for` returns the last successfully displayed thumbnail
(`thumb_texture`) instead. The preview always shows an image -- the previous
one until the new one arrives. No spinner, no black background.

**Single reusable GPU texture**: instead of creating a new `TextureHandle` per
thumbnail (which grows GPU memory), a single `thumb_texture` is reused.
`tex.set()` swaps the pixel data in place without allocating a new GPU texture.

### Disable preview during keyboard navigation

Preview is hidden while nav keys (A/D/arrows) are held. The preview exists for
browsing the slider -- during keyboard nav, the main image already shows what
the user is looking at. This also avoids the fps interference from thumbnail
loading during rapid navigation.

### Bugs fixed in contributor's async thumbnail code

**Missing `request_repaint` in `spawn_thumbnail_thread`**

After the thumbnail background thread finishes decoding and sends its result
into the channel, nothing tells egui to render a new frame. egui only renders
when there's a reason to -- user input (mouse, keyboard) or an explicit
`request_repaint()` call. If the user's cursor is sitting still on the slider
waiting for a thumbnail to load, there's no input, so egui has no reason to
render. The decoded thumbnail sits in the channel until the user happens to
move the mouse or something else triggers a frame.

The full-image `spawn_thread` already calls `ctx.request_repaint()` after
sending its result. The contributor just missed doing the same in
`spawn_thumbnail_thread`.

Fix: clone `ctx` into the thread and call `ctx.request_repaint()` after
`tx.send()`.

**Wrong function in `poll()` thumbnail queue drain**

In `poll()`, when draining `pending_generates` (thumbnail requests that were
queued because the thread limit was hit), the code called `self.spawn_thread`
(the full-image decode function) instead of `self.spawn_thumbnail_thread`.

`spawn_thread` decodes the image at full resolution via `image_to_color_image`
and sends the result through `tx` (the full-image channel). The result ends up
in `pending_uploads` and gets uploaded as a full-size GPU texture -- not a
thumbnail. So any thumbnail request that got queued (because thread slots were
full) would decode the entire image at full resolution, send it through the
wrong channel, and never appear as a thumbnail. Only the first batch of
requests that fit within the thread limit actually produced thumbnails.

Fix: `self.spawn_thread(idx, &path)` -> `self.spawn_thumbnail_thread(idx, &path)`.

### CJK font loading is needed

Tested by disabling the CJK font loading code and viewing images with Japanese
filenames. The window title renders correctly (OS-level text rendering) but the
footer (`src/menu.rs:377`, `paint_pane_footer`) shows missing-glyph squares
because egui's built-in fonts don't include CJK glyphs. The contributor's
hardcoded-path approach works -- NotoSansCJK on this Linux system is at
`/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc`.

### Preview scaling should use window size

The contributor's scaling uses `ui.ctx().screen_rect()` which in egui returns
the window rect, not the monitor. However, the preview still felt too large at
typical window sizes. The `.max(THUMBNAIL_WIDTH/HEIGHT)` floor was removed in
`d3e9f10`, which helps. Further tuning may be needed.

### Thumbnail cache has no eviction

The `thumbnails: HashMap` grows without bound during a session. Unlike the
sliding window cache which evicts at the edges, thumbnails are never removed
(except on `initialize()` which clears the whole map). A cap or LRU eviction
should be added.

### PNG partial-resolution decoding is not possible

Researched whether PNG can be decoded at reduced resolution (like JPEG's 1/8
decode). It cannot -- PNG requires full decode then resize. Same for BMP, WebP,
TIFF. JPEG is the only common format supporting reduced-resolution decode, but
special-casing one format isn't worth it. The current approach (full
`image::open()` then downscale) is the only option for format-agnostic
thumbnails.

## Contributor response and follow-up (2026-05-25 -- 2026-05-29)

### Contributor's `e1758d8` commit

The contributor pushed a commit addressing the feedback from our review. Changes:

- Removed piggybacked thumbnail generation from `spawn_load`, `decode_sync`,
  `DecodeResult`, and the `poll()` upload path. Same fix we identified.
- Added hybrid sync/async with the same 20MB threshold we suggested
  (`BACKGROUND_FILE_SIZE = 20_971_520`).
- Restored `generate_thumbnail_sync` for the sync path.
- Moved `THUMBNAIL_WIDTH/HEIGHT` constants from `app.rs` to `decode.rs`.
- Added early return in `image_to_thumbnail` to skip resize for images already
  smaller than 400x300.

Remaining issue: `decode_sync` still had `img.clone()` even though thumbnails
are no longer generated alongside. The clone copied ~32MB of pixel data per
4K image for no reason.

### Contributor's comment

The contributor confirmed they found the spinner annoying too, and that the fps
drop from piggybacked thumbnail generation was unexpected. They tested sync
loading with `4k_PNG_10MB` images and 7z files and saw the fps drop with sync
on large files, which is why they switched to async. They suggested a config
toggle for the preview.

### Our follow-up commits

**`def8362` -- Remove redundant image clone and hide preview during keyboard nav**

- Removed the unnecessary `img.clone()` in `decode_sync`. Now that thumbnails
  are not generated alongside full-image decodes, the clone serves no purpose.
  For 4K images this was copying ~32MB of pixel data per decode.
- Added `nav_active` check in `paint_nav_slider`: the preview is hidden while
  keyboard nav keys (A/D/Left/Right) are held down. The preview exists for
  browsing via the slider; during keyboard nav the main image already shows
  what the user is looking at. This also avoids any fps interference from
  thumbnail loading during rapid navigation.

**`aec9760` -- Previous-frame fallback and single reusable texture**

- When `current_thumbnail_for` has an async cache miss (file > 20MB), it returns
  the last displayed thumbnail instead of `None`. The preview always shows an
  image -- the previous one stays visible until the new one arrives. No spinner,
  no black background, no empty frame.
- Instead of creating a new GPU texture per thumbnail (which grows GPU memory
  unbounded), a single `thumb_texture` is reused. `tex.set()` swaps the pixel
  data in place without allocating a new GPU texture. Only one texture exists
  at a time.
- Removed `generate_thumbnail_sync` (sync path is now inline in
  `current_thumbnail_for`) and the spinner rendering code from
  `paint_nav_slider`.

**Persistent worker thread (not yet committed)**

A single persistent worker thread design was implemented and saved in
`tmp/cache_with_worker.rs` for reference. It replaces the per-request
`std::thread::spawn` with a long-lived thread that blocks on a channel, drains
to the latest request (discards stale ones), and calls `request_repaint()` when
done. This was identified as a fix for fps drops during hover + keyboard nav
(dozens of thread spawns per second), but needs isolated performance testing
before committing. The preview is hidden during keyboard nav anyway, so the
urgency is low.

### Performance testing summary

All tests on Linux, NVMe SSD, 24-core CPU, RTX 3090, 4K PNG images (~10MB each).

| Scenario | main branch | PR (piggybacked) | PR (fixed) |
|---|---|---|---|
| Keyboard nav only | >60fps | 30-35fps | >60fps |
| Hover only | N/A | ~60fps | ~60fps |
| Keyboard nav + hover | N/A | 30-35fps | 50-60fps |

The remaining 50fps during keyboard nav + hover is from the preview rendering
code itself (layer_painter, galley layout, style lookups). Hiding the preview
during keyboard nav sidesteps this entirely.

### Thumbnail decode timing measurements

Measured with Python PIL as a rough proxy (Rust `image` crate is typically
faster):

| File size | Decode + downscale to 400x300 |
|---|---|
| 10MB 4K PNG | ~150ms |
| 98MB PNG | ~2000ms |

10MB feels instant with sync. 98MB would block the UI for 2 seconds. The 20MB
threshold covers all typical images with sync while pushing giant files to
async.

### Preview FPS benchmarks (using preview-specific counter)

Measured with a dedicated preview FPS counter that only ticks when the hover
cursor moves to a new image index. Tested on 4K PNG images (~10MB each), with
async threshold lowered to 1MB to force all images through the async path.

| Configuration | Preview FPS |
|---|---|
| Sync thumbnail loading | 10-11 |
| Async (per-request thread spawn) | 30-35 |
| Async (persistent worker thread) | 37-42 |

Sync is slower per-frame because it blocks until the thumbnail is ready. Async
is faster because it returns immediately (showing the previous thumbnail) and
decodes in the background. The persistent worker gives a modest improvement
over per-request spawning.

### First-5-seconds sluggishness investigation

On the original async system (`ac85d5e`), the slider preview is noticeably
sluggish for the first ~5 seconds after loading images, then smooths out.

Tested by reproducing the original async code at `ac85d5e` with only the
previous-frame fallback added. The sluggishness was present both WITH and
WITHOUT piggybacked thumbnail generation, ruling that out as the cause.

**Isolation testing to find the root cause:**

Tested on `ac85d5e` base with previous-frame fallback, piggybacked thumbnails
removed, and spinner removed. Applied changes incrementally:

1. Capping `poll()` thumbnail `load_texture` to 1 per frame -- still sluggish.
2. Replacing per-request `std::thread::spawn` with persistent worker -- not
   tested in isolation on this base (was tested on later code where
   sluggishness was already gone).
3. Replacing per-thumbnail `ctx.load_texture()` (creates a new GPU texture each
   time) with a single reusable `TextureHandle` updated via `tex.set()` (swaps
   pixel data in an existing texture) -- **sluggishness gone.**

**Instrumentation results:**

- `ctx.load_texture()` takes 2-9 microseconds per call. Constant, does not
  grow with texture count.
- `device.poll(Maintain::Wait)` takes a consistent 6-7ms regardless of
  thumbnail texture count. The spikes continue even after thumbnail uploads
  stop -- this is baseline GPU sync cost for rendering 4K images, not caused
  by thumbnails.
- Thumbnail uploads are 1 per frame (occasionally 2). Never batched.
- Sliding window cache loads within ~1.5 seconds (cache overlay), but
  sluggishness lasts ~5 seconds.

**What we know:**

- Per-thumbnail `ctx.load_texture()` (new GPU texture each time): sluggish
  for ~5 seconds.
- Single reusable texture via `tex.set()`: no sluggishness.
- Capping `load_texture` to 1 per frame: still sluggish.
- Neither `load_texture` call time nor `device.poll` time increases with
  texture count.

**egui internals (epaint/src/textures.rs):**

`ctx.load_texture()` calls `TextureManager::alloc` which creates a new texture
ID and pushes an `ImageDelta::full` onto `delta.set`. Each new ID creates a
new wgpu texture in the renderer (`device.create_texture()` +
`device.create_bind_group()` + `queue.write_texture()`). These textures are
never freed (stored in HashMap, no eviction).

`tex.set()` calls `TextureManager::set` which reuses the same ID. Line 62:
`self.delta.set.retain(|(x, _)| x != &id)` discards previous enqueued deltas
for this ID. In the renderer, the old wgpu texture is dropped (freed) when
`self.textures.remove(&id)` replaces it. Only 1 wgpu texture exists at a time.

The exact mechanism causing the sluggishness with per-thumbnail texture
allocation is unclear from our measurements. The measurable costs (egui-side
`load_texture`, GPU-side `device.poll`) don't account for it. The fix
(single reusable texture) is confirmed effective.

The thumbnail logic doesn't belong in `SlidingWindowCache` or its `poll()` at
all. `SlidingWindowCache` manages preloaded images for keyboard navigation.
Thumbnails are a separate concern for the slider preview feature. The
contributor put them there because that's where the image decoding code lived,
but they should be independent.

Our fix uses a single reusable `TextureHandle` (`thumb_texture`) updated via
`tex.set()` inside `upload_thumbnail()`. The `poll()` thumbnail drain calls
`upload_thumbnail` instead of `ctx.load_texture`, and `current_thumbnail_for`
returns `thumb_texture` directly. Only one GPU texture exists for the preview
at any time.

### Remaining items

- **Settings toggle**: add a preference to enable/disable the slider preview.
  Default to enabled.
- **Thumbnail cache eviction**: `thumbnail_images` HashMap (CPU side) still
  grows unbounded. The GPU side is now a single texture, but decoded ColorImages
  accumulate. A simple cap or LRU eviction should be added.
- **Future preloading**: could preload thumbnails in the background starting from
  the current position outward, so they're cached before the user hovers. The
  persistent worker thread is already in place for this.
