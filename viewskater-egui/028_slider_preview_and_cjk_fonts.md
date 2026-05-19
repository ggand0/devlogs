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

## Open questions

- Should thumbnails be generated in background threads rather than cloning
  `DynamicImage` (the PR clones the full image before downscaling)?
