# 0080: the Ubuntu desktop stutter, root-caused (appindicator leak fed by MEGAsync)

**Date:** 2026-09-20
**Branch:** none (nothing in the game changed). Follow-up to 0051.
**Claude Code conversation:** `ubuntu-stutter-appindicator-leak`

## Summary

0051 proved the felt stutter is gnome-shell freezing the compositor on a ~11 s metronome and stopped at "trigger unknown, comes and goes". This session found the trigger and the reason it comes and goes. It is a memory leak in Ubuntu's AppIndicator shell extension, fed by MEGAsync re-announcing its tray icon every 10 s. The leak does not cause the GC, it makes each GC long. A local patch is written and verified in isolation. Installing it needs the owner's go (see "Install").

## Status (read this first)

**Root-caused, NOT yet fixed on this box.** As of 2026-09-20 20:50 the patch is staged but not installed (`~/.local/share/gnome-shell/extensions/ubuntu-appindicators@ubuntu.com` does not exist, shell still pid 10905). The stutter is currently gone only because MEGAsync was quit, which freed the leaked pile. MEGAsync autostarts at login (`~/.config/autostart/megasync.desktop`), so after a reboot or re-login the leak restarts from zero and the stutter returns after a few days of uptime. Installing the patch (see "Install") is what makes it permanent.

| state | stutter |
|---|---|
| now: unpatched, MEGAsync off | gone, stays gone while MEGAsync stays off |
| unpatched, MEGAsync on | clean at first, back within days |
| patched + shell restarted | gone for good on this OS release, survives reboots and package updates |

## Measurements (desktop near idle, game not running)

New probe, `work/scripts/shell-stall-probe.py <shell pid> <seconds>`: samples per-thread on-CPU nanoseconds from `/proc/PID/task/TID/schedstat` every 4 ms and reports contiguous runs where the shell main thread is pegged. Passive, no root, replaces pidstat's 1 s resolution with real stall lengths.

| state | main-thread stall | cadence |
|---|---|---|
| shell up 6 days, MEGAsync up 7 days | 172 to 222 ms (10 to 13 frames at 60 Hz) | every 10.00 s exactly |
| MEGAsync frozen with SIGSTOP | 180 to 190 ms | one cycle skipped (21 s), then 11 s |
| MEGAsync quit | 19 s of continuous stall during teardown, then nothing over 25 ms | none |
| after teardown, fine probe (2 ms) | 17 to 21 ms (about one frame) | same timer grid, 12 s |

Shell RSS 1319 MB before, 875 MB after MEGAsync quit: 444 MB released by destroying one tray icon.

## Mechanism

1. **The cadence is GJS, and is normal.** GJS arms a 10 s timer for a full, synchronous, non-incremental GC whenever a GObject wrapper is released (the "big hammer", `g_timeout_add_seconds(10)`). Any churn in the shell re-arms it, so on a live desktop it fires every 10 to 12 s forever. The stalls sit on GLib's seconds-timer grid (always at xx.777) and start 7.56 s after each MEGAsync signal, which rules out the icon update itself being the expensive part. A healthy shell pays about 20 ms for this GC.
2. **The length is the leak.** Session bus census (`dbus-monitor --session --profile`, 25 s): the only recurring traffic is MEGAsync 6.2.1 emitting `NewIcon` x2 + `NewToolTip` every 10.00 s whether or not anything changed, and the extension answering with 3 `Properties.Get`.
3. **The bug** (`ubuntu-appindicators@ubuntu.com` v58, identical in upstream master): `Util.CancellableChild` connects a closure to its parent's `cancelled` signal in the constructor and disconnects only when the child itself is cancelled. On the success path `AppIndicatorProxy.refreshProperty` does `this._cancellables.delete(propertyName)` without cancelling, so every completed property refresh leaves one child permanently connected to the indicator's lifetime cancellable. 3 per MEGAsync cycle is about 26k a day, about 185k after a week. Each is a GObject wrapper plus closure that every full GC must trace. Same pattern in `refreshAllProperties`, both icon-loading paths (`_cleanupIconLoadingCancellable` deletes without cancelling), and the two dbusMenu update paths.
4. **Why it comes and goes.** GC cost grows with MEGAsync uptime (estimate from 200 ms over 7 days: 25 to 30 ms per day) and resets when MEGAsync or the shell restarts. 0051's "quiet" windows and the dock/appindicator toggle round that killed the storm both fit: toggling the extension destroys the indicators and frees the pile.

## Reproduction outside the shell

`work/scripts/appindicator-leak-fix/repro.js` (plain `gjs -m`, the class extracted verbatim): N completed operations against one parent, no references kept.

| | N | steady full GC | RSS | parent teardown |
|---|---|---|---|---|
| original | 50,000 | 14.7 ms | +59 MB | 1.5 s |
| original | 185,000 | 58.7 ms | +211 MB | 17.8 s |
| patched | 185,000 | 0.4 ms | flat | 0 ms |

Teardown is quadratic (each of N handlers schedules an O(N) disconnect), and 17.8 s at the one-week count matches the 19 s desktop freeze observed when MEGAsync quit. The in-shell GC cost is higher than the bare repro (170 to 220 ms vs 59 ms) because each leaked child in the shell also pins its promise chain and variants.

A first liveness check using `WeakRef` reported the patched children as still alive even though their handlers were provably gone (parent cancel took 0.0 ms). The GC time and RSS numbers above show they are freed, so the WeakRef check was the faulty part. Probable reason, not investigated: WeakRef targets stay pinned as kept objects in that script shape.

## The fix

`work/scripts/appindicator-leak-fix/cancellable-leak.patch` (28 changed lines, 3 files). Same pattern GLib uses: connect, run the operation, disconnect when it is over.

- `util.js`: `CancellableChild.release()` unlinks from the parent without cancelling.
- `appIndicator.js`: `release()` in a `finally` in `refreshProperty`, `refreshAllProperties` and both icon-loading paths.
- `dbusMenu.js`: `release()` when a properties update settles. The layout update starts property requests un-awaited on the same cancellable, so it collects them and releases only after `Promise.allSettled`, keeping cancel-on-destroy intact.

Syntax-checked with `node --check`. Not yet run inside the shell.

## Install (needs the owner)

A patched full copy of the extension is staged at `work/scripts/appindicator-leak-fix/ubuntu-appindicators@ubuntu.com/`. The shell scans the user data dir first and skips a system copy with the same UUID (checked in the shell's own `extensionSystem.js` and `fileUtils.js`), so a per-user copy overrides the packaged one with no sudo:

```
cp -r work/scripts/appindicator-leak-fix/ubuntu-appindicators@ubuntu.com ~/.local/share/gnome-shell/extensions/
```

then Alt+F2, `r`, Enter (X11 restart, windows survive), then start MEGAsync. Revert by moving that directory away and restarting the shell again. The agent's attempt to install it was blocked as a persistent change to the desktop, which is the right call for the owner to make.

## Persistence and OS upgrades

- **Reboots:** a copy under `~/.local/share/gnome-shell/extensions/` is a normal per-user extension install. It survives reboots, re-logins and `apt upgrade` (the packaged copy in `/usr/share` keeps getting updated but stays shadowed).
- **Release upgrade (24.04 to 26.04):** the override must be REMOVED BEFORE upgrading. The v58 copy declares `shell-version: ["45", "46"]`. It shadows the system copy by UUID, so under GNOME 50 the shell would find the user copy first, mark it out of date, and skip the packaged one too: no tray icons at all. After the upgrade, re-stage the patch from the new release's own files (26.04 ships them inside `gnome-shell-ubuntu-extensions`, `subprojects/appindicators/`) and re-apply the same three edits. Not checked: whether 26.04 keeps the same UUID.
- **Stale-override risk on 24.04:** if Ubuntu ever ships a fixed or changed v58+ through updates, the override keeps the old patched code. Low risk on an LTS, but if tray icons misbehave after an update, move the override away first.
- The manual step disappears only if upstream merges a fix. Until then it is once per OS release.

## Verification after install

Run `python3 work/scripts/shell-stall-probe.py $(pgrep -x gnome-shell) 60` on day 0 and again after 3+ days of MEGAsync uptime. Pass: stalls stay in the 20 ms class and shell RSS stays flat. Fail: stalls grow by tens of ms per day, meaning another leaking path exists.

## Corrections to 0051

- MPRIS / media tabs: not involved. The metronome fired with zero media players on the bus.
- ubuntu-dock: not the driver. The storm died in that session because the toggle round also cycled appindicator.
- gvfs-udisks2-volume-monitor showed 1.1% lifetime CPU but was idle during every sample. Historical, likely the /data_hdd2 mount flapping.
- "11 to 12 s" period vs today's 10.00 s: the period is 10 s plus the delay until the next wrapper release re-arms the timer. MEGAsync re-arms it within the same second.

## Does upgrading Ubuntu fix it? No (checked 2026-09-20)

The extension is part of stock Ubuntu: in the `main` archive, enabled by default in the Ubuntu session. The leaking `CancellableChild` class is byte-identical (md5 of the class body `25052bf0`) and `refreshProperty` has no unlink on its success path in every version checked:

| where | version | leak present |
|---|---|---|
| 24.04 (installed) | 58-1ubuntu24.04.1 | yes |
| 25.10 | 61-2 | yes |
| 26.04 release / proposed | `gnome-shell-ubuntu-extensions` 50.26.04.7 / .8 (the standalone package was folded into this bundle, lp #2142566, code under `subprojects/appindicators/`) | yes |
| 26.10 dev | bundle 51.26.10.2 | yes |
| upstream | v65 and master | yes |

No distro patch touches it. Checked by downloading the source tarballs from Launchpad. The local patch carries forward unchanged in shape, but on 26.04 the UUID/paths come from the bundle, so re-stage from that release's files instead of copying the v58 directory.

Owner feel check after MEGAsync quit (same day, 1 to 2 min of play): stutter gone.

## Not done

- Upstream report. Issues #321 and Launchpad #2043538 describe the symptom, nobody has tied it to `CancellableChild`. Filing is the owner's call.
- The fullscreen unredirect A/B from 0051 is moot now.
