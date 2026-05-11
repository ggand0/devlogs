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

## Open questions

- Should thumbnails be generated in background threads rather than cloning
  `DynamicImage` (the PR clones the full image before downscaling)?
- Exact positioning math for fixed-offset preview above the slider.
