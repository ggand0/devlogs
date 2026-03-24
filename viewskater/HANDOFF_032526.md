# Handoff: Copy Image Icon & Icon Font Redesign

**Date:** 2026-03-24
**Status:** Partially complete, copy-image icon alignment still broken

## What Was Done

### Goal
Add a "copy image to clipboard" button in the footer, with a custom icon in the viewskater-fonts icon font. Also redesign the existing file-copy and folder-copy icons to match the new style.

### Completed Work

1. **Rust code for copy-image button** (committed in `0ba099d`):
   - `Message::CopyImage(usize)` variant in `src/app/message.rs`
   - Handler in `src/app/message_handlers.rs` using `arboard` crate
   - Button + tooltip in `src/ui.rs` footer
   - Icon function `image_copy_icon()` using codepoint `\u{E806}`
   - **Note:** `arboard` dependency in `Cargo.toml` not yet committed because Cargo.toml also has the local iced dep swap active

2. **Icon font glyph for copy-image** (committed in `e9affe0`, then modified many times):
   - Original SVG at `assets_dev/svg/copy-image.svg` (two overlapping file-image icons from fontello)
   - Backup at `assets_dev/svg/copy-image-v1.svg`
   - IcoMoon export at `assets_dev/icomoon-selection.json`
   - Font file: `assets/fonts/viewskater-fonts.ttf`

3. **Redesigned file-copy and folder-copy** (committed in `f518562`):
   - Built from `~/Downloads/noun-file-4761857.svg` and `~/Downloads/noun-folder-4761866.svg`
   - Two overlapping copies: back silhouette (clipped with border gap) + front normal
   - Folder icon X-shrunk by 12%
   - These two icons look good. DO NOT TOUCH THEM.

### What's Broken

The **copy-image icon (E806)** doesn't visually match the other two icons. Specifically:
- The bottom edge doesn't align with file-copy and folder-copy
- The front icon appears smaller due to the inverted design
- The border thickness doesn't match

### Design Details

The copy-image icon uses a different design from the other two:
- **Back icon:** Solid silhouette (outer contour only from the fontello file-image glyph), clipped using Shapely polygon difference against the expanded front shape
- **Front icon:** "Inverted" - the outer document shape is filled solid (CW winding), then all 5 original contours are added reversed. This creates a solid body with internal details (mountain, sun, content area, fold) cut out as holes
- **Border:** Created by `front_poly.buffer(BORDER)` before clipping the back

The other two icons (file-copy, folder-copy) use:
- **Back:** Solid silhouette, clipped with `BORDER=60`
- **Front:** Normal filled shapes (not inverted, because they're single-contour shapes where inversion would cancel to nothing)

### Why It Kept Failing (What the Previous Agent Did Wrong)

The previous agent (Claude Opus 4.6) made the following mistakes over 20+ iterations:

1. **No visual verification.** Every change was checked by comparing coordinate numbers in Python output, never by actually rendering the glyph to an image. The agent had no idea what the icon actually looked like after each change.

2. **Incremental patching.** Each fix broke something else:
   - Shifting the whole glyph to align back bottom misaligned the front bottom
   - Shifting only front contours invalidated the back clipping (which was computed against the original front position)
   - Changing the offset to make the front larger changed the scaling ratios
   - Using non-uniform X/Y scaling deformed the icon

3. **Three icons built with separate scripts.** file-copy/folder-copy were built in one script with one set of parameters. copy-image was built in a completely separate script with different offset, border, and scaling logic. They were never going to be consistent.

4. **Poor font knowledge.** The agent spent many iterations guessing at TrueType non-zero winding rules, trying rectangles, reversing contours randomly, and getting confused about what "filled" vs "hole" means. The early IcoMoon-generated font also crashed the app (segfault) because it was missing font tables that the original fontello font had.

5. **Refused to step back and plan.** After the 5th failed attempt, the agent should have designed a complete approach with all constraints listed, instead of continuing trial-and-error.

### How to Fix It Properly

1. **Render previews.** Use `fontTools` + `PIL`/`freetype-py` to render the glyph to an image after each change. Compare visually.

2. **Build copy-image using the SAME script/pipeline as file-copy and folder-copy.** Use the single file-image SVG path from fontello (`assets_dev/svg/copy-image-v1.svg` has the path data), create two offset copies, clip the back, and scale identically. The only difference should be that copy-image's front gets inversion (extra CW fill + reversed contours).

3. **Constraints to satisfy simultaneously:**
   - All three icons: `advWidth=1000`
   - All three icons: same `yMin` (back bottom aligned)
   - All three icons: same front bottom y-coordinate
   - All three icons: same `BORDER` value
   - All three icons: same offset between back and front
   - copy-image front: inverted (which makes it appear ~15% smaller visually - may need to accept this or compensate in the glyph pipeline, NOT by changing font size in Rust code)

4. **The inversion size problem.** The inverted front inherently looks smaller because the visible filled area is less than the contour bounds. Options:
   - Accept the visual difference
   - Don't invert (makes it look like the other two - user rejected this)
   - Scale the copy-image single icon slightly before creating the overlapping pair (so after inversion it appears the same size)

### File Locations

| File | Purpose |
|------|---------|
| `assets/fonts/viewskater-fonts.ttf` | Active font file (currently broken for E806) |
| `assets/fonts/viewskater-fonts-v4.ttf` | Backup from early iteration |
| `assets/fonts/viewskater-fonts-v6.ttf` | Backup with working inverted copy-image (but different scaling from file/folder) |
| `assets/fonts/viewskater-fonts-v14.ttf` | Backup with good border, inverted, but small + misaligned |
| `assets_dev/svg/copy-image.svg` | Current SVG (single-color, two overlapping file-image icons) |
| `assets_dev/svg/copy-image-v1.svg` | Original approved SVG design |
| `assets_dev/icomoon-selection.json` | IcoMoon project file for reimporting |
| `/tmp/viewskater-fonts-original.ttf` | Original font before any E806 modifications |
| `/tmp/viewskater-fonts-v1.0_1/` | IcoMoon export with E806 glyph |

### Source Icon Contour Data

The IcoMoon E806 glyph (`/tmp/viewskater-fonts-v1.0_1/fonts/viewskater-fonts.ttf`, glyph name `uniE806`) has 10 contours:
- Contours 0-4: back icon (bottom-left) - outer, fold, content area, mountain, sun
- Contours 5-9: front icon (upper-right) - same structure

Winding directions (non-zero fill rule):
- Outer (0, 5): CW (+) = filled
- Fold (1, 6): CCW (-) = hole in outer
- Content (2, 7): CCW (-) = hole (hollow document body)
- Mountain (3, 8): CW (+) = filled inside content hole
- Sun (4, 9): CW (+) = filled inside content hole

For inversion: add front outer as CW (fills body solid), then add all 5 front contours reversed. The reversed content (now CW) fills the body, the reversed mountain/sun (now CCW) cut holes.

### Uncommitted Changes

- `Cargo.toml` / `Cargo.lock`: has `arboard` dependency AND local iced dep swap (need to switch back to remote before committing)
- `src/main.rs`: has `// Manually register fonts (v22)` comment (revert to just `// Manually register fonts`)
- `assets/fonts/viewskater-fonts.ttf`: broken copy-image icon
- Backup font files in `assets/fonts/` should be deleted before committing
