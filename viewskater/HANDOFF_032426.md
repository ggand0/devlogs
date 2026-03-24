# Handoff: Dual-Pane Drag-and-Drop Fix

**Date:** 2026-03-24
**Status:** Fix confirmed working, needs cleanup and commit

## What Was Done

Fixed a bug where drag-and-dropping files in dual-pane mode on Linux always loaded into the right pane. The fix has two parts:

### 1. winit X11 DnD position fix
The iced-compatible winit fork (rev `47974007`) emitted `WindowEvent::DragDrop` with hardcoded `(0.0, 0.0)` positions on X11. The egui-compatible fork (branch `custom-dnd-0.30.13`) had this fixed in commit `f94ea9ea`. That commit was cherry-picked onto `47974007` as branch `fix/x11-dnd-position` (commit `7dea6466`), pushed to GitHub.

### 2. Split widget handlers updated
Both `src/widgets/split.rs` and `src/widgets/synced_image_split.rs` had Linux `FileDropped` handlers that discarded the event position (`_`) and used `cursor.position().unwrap_or_default()`. Changed to use the event position directly: `Point::new(position.x as f32, position.y as f32)`.

## Current State of Each Repo

### data-viewer (this repo)
- **Branch:** `main`
- **Uncommitted changes:**
  - `Cargo.toml` — switched from git iced deps to LOCAL path deps (`path = "../iced/*"`). The git deps are commented out. **Must switch back to git deps after iced's custom-0.13 branch is updated with the new winit rev.**
  - `Cargo.lock` — reflects local deps
  - `src/widgets/split.rs` — Linux FileDropped handler uses event position instead of cursor.position()
  - `src/widgets/synced_image_split.rs` — same fix
- **Devlog:** `devlogs/052_dual_pane_drag_drop_bug.md` — full investigation writeup
- **User wants:** a feature branch (`fix/...` naming) for these changes

### iced (`../iced/`)
- **Uncommitted change in `Cargo.toml`:** winit dep changed from `git rev 47974007` to `path = "../winit"`. **This needs to be updated to `git rev 7dea6466` and pushed to the `custom-0.13` branch.**
- Original line was: `winit = { git = "https://github.com/ggand0/winit.git", rev = "47974007b84eb488a20d0a2eca5a2aa9b5ee66b2" }`

### winit (`../winit/`)
- **Current branch:** `fix/x11-dnd-position` (commit `7dea6466`)
- **Pushed to GitHub:** yes
- **Original branch was:** `custom-dnd-0.30.13` (commit `ea442bd8`)
- The fix cherry-picks `f94ea9ea` onto `47974007`

## What Still Needs to Be Done

1. **iced repo:** Update `Cargo.toml` winit dep to `rev = "7dea6466a4b5e174932c493ec65919afb2b9d036"`, commit, push to `custom-0.13` branch
2. **data-viewer repo:** Switch `Cargo.toml` back to git iced deps (uncomment git lines, comment out path lines), create feature branch, commit the split widget fixes + Cargo changes
3. **winit repo:** Switch back to `custom-dnd-0.30.13` branch (`git checkout custom-dnd-0.30.13`) — the `fix/x11-dnd-position` branch is already pushed and referenced by rev

## Key Commits / Refs

| Repo | Ref | Description |
|------|-----|-------------|
| winit | `47974007` | Old iced winit rev (placeholder DnD positions) |
| winit | `7dea6466` | New rev with X11 DnD fix (branch `fix/x11-dnd-position`) |
| winit | `f94ea9ea` | Original fix on `custom-dnd-0.30.13` that was cherry-picked |
| winit | `ea442bd8` | HEAD of `custom-dnd-0.30.13` (egui's branch) |
| data-viewer | `8243cd6` | Commit that introduced bug in split.rs (PR #33) |
| data-viewer | `8d99560` | Commit that introduced bug in synced_image_split.rs (PR #44) |

## Bug Root Cause Summary

Two layers:
1. **winit:** X11 XdndPosition handler had position extraction commented out. DragDrop events were emitted with `PhysicalPosition::new(0.0, 0.0)`.
2. **split widgets:** Linux handlers discarded the (always-zero) event position and used `cursor.position()` as a workaround. This returned stale positions during external drag operations, typically matching the right pane.

Wayland has no DnD support in this winit fork at all — only X11.
