# PR #20: Window State Persistence via eframe Built-in (viewskater-egui)

Scope: this devlog covers PR #20 only (eframe's built-in `persistence` feature, branch `feat/window-state`). PR #29 takes a different code path (custom YAML save/load) and its issues are tracked separately.

**Date**: May 31, 2026
**Repo**: `ggand0/viewskater-egui`
**Related PRs**:
- [#20 "Egui persistence test"](https://github.com/ggand0/viewskater-egui/pull/20) — branch `hml-pip:feat/window-state`, DRAFT
- [#29 "Implement window persistence feature"](https://github.com/ggand0/viewskater-egui/pull/29) — branch `hml-pip:feat/window-presistence`, DRAFT
- [#61 (iced version) "Automatic window state save and restore"](https://github.com/ggand0/viewskater/pull/61) — MERGED, used as reference by contributors
**Pinned versions on `feat/window-state`**: `eframe 0.31.1`, `egui 0.31`, `winit` via `ggand0/winit.git` branch `custom-dnd-0.30.13`
**Test machine**: Windows 10 Home Single Language 22H2, build 19045.6466
**Monitor layout**:
| Role | Windows name | Bounds | Notes |
|---|---|---|---|
| Primary | `\\.\DISPLAY2` | x:0..2560, y:0..1440 | landscape 2560×1440 at origin |
| Secondary | `\\.\DISPLAY1` | x:3840..4920, y:228..2148 | portrait 1080×1920, right of primary |

> Windows' `\\.\DISPLAYn` numbering is registration order, not "which is primary." On this PC the secondary monitor is named `DISPLAY1`.

## Overview

The egui-based viewskater fork ships with no window state persistence by default. Two community PRs attempt to add it:

- **#20** uses eframe's built-in persistence: enabling it causes eframe to write `app.ron` under `eframe::storage_dir("viewskater-egui")` (on Windows: `%APPDATA%\viewskater-egui\data\app.ron`) and reload window position/size on startup. The PR also adjusts `main.rs` to skip the hardcoded `with_inner_size([1280.0, 720.0])` when a persisted state file exists, and gates the DPI physical-pixel correction in `app.rs` to only fire on first launch.
- **#29** is a more custom implementation by the same contributor (hml-pip), intended to address gaps in #20 for situations egui's built-in persistence does not handle well.

Both PRs are explicitly marked draft. The PR author noted in #29: *"this PR is not really meaningful right now, I'll just use this for my personal experiments. If there's no further improvement or interest, I'll just close this later."*

## Issues reported in PR thread

Numbered by order of discussion. Status reflects local reproduction on the test machine above.

### #1 — White flash on restore of maximized window (YelovSK)
Briefly shows a white rectangle when launching with persisted `maximized:true`.
- **Upstream cause** per YelovSK: winit calls the Win32 maximize API before hiding the window. Upstream PR: [winit#4575](https://github.com/rust-windowing/winit/pull/4575).
- **Status: REPRODUCED** ✅

### #2 — Unmaximize forgets prior normal size (YelovSK)
After relaunching into a maximized state, clicking the unmaximize button does not return the window to its previous normal size.
- **Cause** per YelovSK: neither egui nor winit exposes Win32's `WINDOWPLACEMENT.rcNormalPosition` (the "restore rect"). Would require calling `GetWindowPlacement` from inside egui.
- **Status: REPRODUCED** ✅

### #3 — Saved position on secondary monitor is not restored (YelovSK + hml-pip + local)
A window placed on the secondary monitor before quit does not return there on relaunch. Reproduces across all window-state variants:
- **Maximized variant** (YelovSK): maximize on secondary, quit, relaunch — comes up maximized on primary.
- **Fullscreen variant** (hml-pip, PR description): fullscreen on secondary, quit, relaunch — comes up on primary.
- **Normal-windowed variant** (local): drag windowed (not maximized/fullscreen) to secondary, quit, relaunch — appears at a small size in the top-left of primary. Observed: persisted `inner_size_points` (~751×589) was applied, but the persisted position was not.
- **Upstream fix candidate**: [egui#8191](https://github.com/emilk/egui/pull/8191) ("choose monitor by overlap instead"). YelovSK reports merged on May 25, 2026 — not yet in the pinned `egui 0.31`.
- **Status: REPRODUCED** ✅ (all three variants confirmed locally)

### #4 — Window size after exiting fullscreen isn't retained (hml-pip, PR description)
Enter fullscreen, quit while in fullscreen, relaunch (comes up fullscreen), exit fullscreen — window does not return to the pre-fullscreen size.
- **Status: REPRODUCED** ✅

## Reproduction summary table

| # | Issue | Reporter | Reproduced |
|---|---|---|---|
| 1 | Maximize white flash | YelovSK | ✅ |
| 2 | Unmaximize loses normal size | YelovSK | ✅ |
| 3 | Saved position on secondary monitor not restored (maximized / fullscreen / windowed variants) | YelovSK + hml-pip + local | ✅ (all variants) |
| 4 | Fullscreen exit loses windowed size | hml-pip | ✅ |

## Testing notes

- **"Quit normally"** = File→Quit, window close button, or `Alt+F4`. Killing the process skips eframe's `save()`, which bypasses the persistence path under test.
- **Reset persisted state between tests**:
  ```powershell
  Remove-Item $env:APPDATA\viewskater-egui\data\app.ron -ErrorAction SilentlyContinue
  ```
- **Inspect persisted state**:
  ```powershell
  Get-Content $env:APPDATA\viewskater-egui\data\app.ron | Select-String 'window'
  ```
  Relevant fields in the `window` line: `inner_position_pixels`, `outer_position_pixels`, `fullscreen`, `maximized`, `inner_size_points`.

## Path forward / ship decision

The reproduced issues split cleanly by platform:

- **macOS + Ubuntu**: window restore works correctly across all variants (windowed / maximized / fullscreen / multi-monitor). Verified previously by ggand0 ("I verified it working on macOS and Ubuntu") at the time the fix commit `ad70399` was pushed.
- **Windows**: #1 (white flash on maximize restore) and #3 (saved position on secondary monitor not restored, in all three variants) and #4 (fullscreen exit loses windowed size) and #2 (unmaximize loses normal size) all reproduce.

### Why "just fix winit/egui upstream" is not viable short-term

- **#1 white flash** has an upstream fix proposed in [winit#4575](https://github.com/rust-windowing/winit/pull/4575), but winit PR throughput is slow.
- **#3 multi-monitor restore** has an upstream fix merged in [egui#8191](https://github.com/emilk/egui/pull/8191) ("choose monitor by overlap") on 2026-05-25 — but it's only available in egui ≥ 0.32, and this branch is pinned to `egui 0.31` / `eframe 0.31.1`. A version bump is its own task.
- **#2 unmaximize size** requires querying Win32's `GetWindowPlacement.rcNormalPosition` ("restore rect"), which neither egui nor winit currently exposes. No upstream fix in sight.

### Why forking winit is undesirable

- The viewskater iced version's experience maintaining a winit fork was painful (per ggand0).
- The egui branch currently does depend on a custom winit (`ggand0/winit.git#custom-dnd-0.30.13`) for an unrelated DnD fix, so technically "one more cherry-pick" is possible — but compounding fork patches increases the maintenance burden.

### Options considered

1. **Cargo `[patch.crates-io]` winit** pointing at the winit#4575 contributor's branch — low commitment, but breaks if the contributor force-pushes or deletes their branch.
2. **Cherry-pick winit#4575** onto the existing `custom-dnd-0.30.13` fork — adds to fork burden but is durable.
3. **Workaround in viewskater-egui**: don't restore maximized/fullscreen state on Windows only — keep restore for macOS/Ubuntu. Windows users would have to re-maximize manually on relaunch. Avoids the flash and sidesteps fullscreen-related issues (#4) on Windows.
4. **Document as known limitations and wait for upstream.**
5. **Try YelovSK's "create hidden, maximize after show" approach** — already tested by them and reported to still show the maximize animation, so not a clean workaround.

### Tentative direction

Option 3 — ship #20 with maximize/fullscreen restore disabled on Windows only — gives a working feature on macOS/Ubuntu and a degraded-but-flash-free experience on Windows, with no new fork dependencies.

**Open implementation question**: #20 relies on eframe's built-in `persistence` feature, which handles position + size + maximized + fullscreen together via `app.ron`. Selectively opting out of just the maximize/fullscreen restore on Windows is not obviously a one-line change. Possible implementation paths (not yet verified):
- find an eframe `NativeOptions` knob to disable maximize-state restore selectively
- post-process `app.ron` on Windows at startup to strip the maximize/fullscreen flags before eframe reads it
- switch to a hybrid where size+position come from eframe but maximize/fullscreen come from custom logic

If none of these are clean, the cost of the Windows-only opt-out increases and PR #29's custom persistence (where every field is controlled explicitly in user code) becomes comparatively more attractive. eframe 0.31's persistence API needs to be checked before committing to this direction.

## Draft reply for PR #20 thread

Not yet posted. To be reviewed and posted by ggand0 manually.

> Confirming I reproduced these on Windows 10:
>
> - white flash on maximize restore
> - unmaximize losing the prior normal size
> - wrong-monitor restore — for maximized, fullscreen, **and** also for normal windowed mode (window comes back small at the top-left of the primary regardless of where on the secondary it was saved)
> - fullscreen exit losing the previous windowed size
