# GNOME wsmatrix Extension: GPU Usage Fix

**Date:** 2026-04-07
**Hardware:** NVIDIA RTX 3090, dual monitors (2560x1440 + 1080x1920 rotated)
**OS:** Ubuntu 24.04 LTS, GNOME Shell 46.0
**Extension:** Workspace Matrix (wsmatrix) v50

## Problem

Noticed ~22% sustained GPU usage from gnome-shell and moderately noisy fan at idle.
`nvidia-smi pmon` confirmed gnome-shell was using 23% SM (compute) and 13% memory bandwidth continuously.

Disabling the wsmatrix extension dropped GPU usage to 1-3%. Re-enabling brought it back to ~22%.

## Root Cause Analysis

Investigated the extension source at `~/.local/share/gnome-shell/extensions/wsmatrix@martin.zurowietz.de/`.

Three issues identified:

### 1. Live BackgroundManager instances (CRITICAL)

`workspacePopup/workspaceThumbnail.js` created a `BackgroundManager` (live wallpaper texture renderer) for every workspace thumbnail unconditionally in `_init()`. These persisted even when the overview/popup was hidden.

With a 3x3 workspace grid × 2 monitors, that's up to 18 live background textures being rendered by the compositor every frame, even at idle.

### 2. Redundant style-changed → redisplay() cascade

`workspacePopup/workspaceSwitcherPopupList.js` had a `style-changed` signal handler that called the expensive `redisplay()` method unconditionally, even when spacing values hadn't changed. This could cascade with other layout signals.

### 3. Unconditional compositor BEFORE_REDRAW callbacks

`overview/thumbnailsBox.js` scheduled a `Meta.LaterType.BEFORE_REDRAW` compositor callback to hide the drop placeholder on every `vfunc_allocate` pass, even when the placeholder was already hidden. This prevented the GPU from fully going idle between frames.

## Fix

Forked the repo: https://github.com/ggand0/gnome-shell-wsmatrix
Branch: `fix/reduce-gpu-usage`

### Fix 1: Lazy BackgroundManager lifecycle
- BackgroundManagers are now only created when the thumbnail actor is `mapped` (visible on screen)
- Destroyed when `unmapped` (overview closed / popup dismissed)
- Also destroyed on actor `destroy` signal for cleanup safety
- Guard in `_createBackground()` prevents duplicate creation

### Fix 2: Guard style-changed handler
- Only calls `redisplay()` when spacing values actually differ from current values

### Fix 3: Guard BEFORE_REDRAW later
- Only schedules the compositor callback when `_dropPlaceholder.visible` is true

## Testing

- Re-enabled the extension with the patched code
- Extension functions correctly (workspace grid, switcher popup, overview thumbnails)
- GPU usage TBD (monitoring for sustained improvement)

## Files Modified

```
wsmatrix@martin.zurowietz.de/workspacePopup/workspaceThumbnail.js     (Fix 1)
wsmatrix@martin.zurowietz.de/workspacePopup/workspaceSwitcherPopupList.js  (Fix 2)
wsmatrix@martin.zurowietz.de/overview/thumbnailsBox.js                (Fix 3)
```

## Backup

Original extension backed up to:
`~/.local/share/gnome-shell/extensions/wsmatrix@martin.zurowietz.de.bak/`

## Notes

- No update available for GNOME 46 (already on latest v50; v53 is for GNOME 47+)
- Fork cloned to `~/ggando/gnome-shell-wsmatrix/`
- Could submit upstream PR if fix proves stable
