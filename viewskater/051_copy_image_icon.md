# Copy Image Icon (E806) - Fix and Reproduction Guide

**Date:** 2026-03-25

## Summary

Fixed the copy-image icon glyph (U+E806) in `viewskater-fonts.ttf` to align with file-copy (E804) and folder-copy (E805).

## What Was Fixed

The copy-image icon had three issues:
- Bottom edge didn't align with the other two icons
- Front icon appeared smaller due to incorrect scaling
- Border gap between back and front was inconsistent

Root cause: the copy-image glyph was built with a separate script using different offset, border, and scaling parameters from the other two icons.

## Design

All three copy icons follow the same visual pattern: two overlapping copies of a base icon (back silhouette + front).

- **file-copy (E804):** Two file shapes. Front has fold cutout.
- **folder-copy (E805):** Two folder shapes. Front is solid.
- **copy-image (E806):** Two file-image shapes. Front is "inverted" — solid document body with fold, mountain, and sun as cutout holes.

The "inverted" front makes copy-image visually distinct from file-copy. Instead of showing internal details as filled shapes inside a hollow content area, the entire body is solid and the image elements (mountain, sun) appear as negative space.

## Glyph Structure (E806)

5 contours total:
1. **Back silhouette** (CW, filled): Outer contour of the file-image shape, clipped against the buffered front using Shapely polygon difference
2. **Front outer** (CW, filled): Document body, solid fill
3. **Front fold** (CCW, hole): Dog-ear cutout in top-right corner
4. **Front mountain** (CCW, hole): Mountain silhouette carved out of body
5. **Front sun** (CCW, hole): Sun circle carved out of body

Winding rule: TrueType non-zero fill. CW contours contribute +1 winding, CCW contribute -1. Holes are created where opposite windings cancel to zero.

## Alignment Parameters

All three icons share:
- `advWidth = 1000`
- `yMin = -196` (back bottom)
- Front yMin = -22
- Front offset: dx=219, dy=174 (relative to back)
- Border = 85 font units (gap between front edge and clipped back)

## How to Reproduce

### Prerequisites
```
pip install fonttools shapely Pillow freetype-py
```

### Source Data
- **IcoMoon font:** `/tmp/viewskater-fonts-v1.0_1/fonts/viewskater-fonts.ttf` — contains the original E806 glyph with 10 contours (back 0-4, front 5-9)
- **Working font:** `assets/fonts/viewskater-fonts.ttf` — E804/E805 used as alignment reference

### Pipeline (Python)

1. **Extract single file-image icon** from IcoMoon E806 contours 5-9 (the front copy), shift to origin.

2. **Compute scale factor:**
   ```
   scale = (1000 - FC_DX) / ICON_W
   ```
   Where FC_DX=219 (file-copy front xMin), ICON_W=675 (source icon width).
   Result: scale ≈ 1.157

3. **Create two copies** in source coordinates:
   - Back at (0, 0)
   - Front at (DX_src, DY_src) where DX_src = 219/scale, DY_src = 174/scale

4. **Clip back** using Shapely:
   ```python
   BORDER_SRC = 85 / scale  # ~73.5 in source coords
   back_clipped = back_poly.difference(front_outer_poly.buffer(BORDER_SRC))
   ```

5. **Build inverted front** (4 contours):
   - Outer: keep original CW winding (filled)
   - Fold: keep original CCW winding (hole)
   - Mountain: reverse from CW to CCW (hole)
   - Sun: reverse from CW to CCW (hole)
   - Content contour: dropped (becomes part of solid body)

6. **Scale and shift** all contours to final font coordinates:
   ```python
   transform_contour(contour, scale=scale, dx=0, dy=-196)
   ```

7. **Write to font** using fontTools GlyphCoordinates.

8. **Render preview** with freetype-py to verify visually.

### Build Script
The full script is at `/tmp/fix_copy_image.py`. The border comparison script that generates variants at different border values is inline in the conversation history.

## Font File Backups

| File | Description |
|------|-------------|
| `assets/fonts/viewskater-fonts.ttf` | Active font (border=85) |
| `assets/fonts/viewskater-fonts-v15.ttf` | First fix (border=60) |
| `/tmp/viewskater-fonts-border60.ttf` | Border=60 variant |
| `/tmp/viewskater-fonts-border75.ttf` | Border=75 variant |
| `/tmp/viewskater-fonts-border85.ttf` | Border=85 variant |
