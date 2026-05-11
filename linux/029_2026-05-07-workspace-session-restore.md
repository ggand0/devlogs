# Workspace Session Save/Restore Across Reboots

**Date:** 2026-05-07
**OS:** Ubuntu 24.04 LTS, GNOME Shell 46.0, X11
**Extensions:** Workspace Matrix (wsmatrix) — 3x3 grid (9 workspaces, linear index 0–8)
**Hardware:** Dual monitors (2560x1440 + 1080x1920 rotated)

## Problem

Dual-booting Windows (gaming at night) and Linux (work in the morning) means the desktop session is lost on every reboot. Need to restore:

- **Terminal windows** attached to specific tmux sessions, on the correct workspace
- **Nautilus windows** opened to specific directories, on the correct workspace
- **Brave browser** tabs and multi-monitor window placement

Brave's "Continue where you left off" setting doesn't auto-restore after an unclean shutdown (reboot) — it shows a "Restore" prompt requiring manual click. Windows also don't restore to their original monitor.

## Solution Architecture

Two scripts + systemd timer + GNOME autostart:

```
Save (periodic, every 5 min)           Restore (on login)
─────────────────────────               ──────────────────
wmctrl -l -x -G                        1. Patch Brave exit_type
  → parse window class,                2. Start tmux server
    workspace, geometry                    → triggers tmux-continuum restore
  → terminals: extract tmux            3. Open gnome-terminal per session
    session from [tmux:#S] title           → tmux attach-session -t $name
  → nautilus: resolve dir path          4. Open nautilus --new-window $path
    from window title                   5. Launch brave-browser --restore-last-session
  → brave: save workspace + pos         6. wmctrl -i -r $wid -t $ws -e geometry
  → write ~/.cache/workspace-             for each opened window
    session/state.json
```

### Files Created

| File | Purpose |
|------|---------|
| `~/.local/bin/workspace-save.sh` | Captures window layout to JSON |
| `~/.local/bin/workspace-restore.sh` | Reopens everything on login |
| `~/.config/systemd/user/workspace-save.timer` | Triggers save every 5 min |
| `~/.config/systemd/user/workspace-save.service` | Oneshot service for save |
| `~/.config/autostart/workspace-restore.desktop` | Runs restore on GNOME login |

### Files Modified

| File | Change |
|------|--------|
| `~/.tmux.conf` | Added `set -g set-titles on` / `set -g set-titles-string '[tmux:#S] #W'` |
| `~/.config/autostart/gnome-terminal.desktop` | Disabled (restore script handles it) |
| `~/.config/autostart/nautilus.desktop` | Disabled (restore script handles it) |

## Implementation Details

### Save Script — Window Detection

Uses `wmctrl -l -x -G` which outputs per-window: ID, workspace index, x, y, w, h, WM class, hostname, title.

**Terminal detection:** Tmux sets the terminal title via `set-titles-string '[tmux:#S] #W'`, producing titles like `[tmux:ml] python`. The save script matches `^\[tmux:([^\]]+)\]` to extract the session name.

**Nautilus path resolution:** Maps the window title (folder name only — Nautilus doesn't show full paths) back to a filesystem path:
- Hardcoded XDG mappings: Home→$HOME, Documents→$XDG_DOCUMENTS_DIR, etc.
- Fallback: `find` search under $HOME, /data, /data2, /tmp, /media, /mnt (maxdepth 4, 3s timeout)
- Known limitation: ambiguous names like "tmp" may resolve to the wrong directory

**Brave:** Saves workspace, position, geometry, and title for each window.

### State File Format

```json
{
  "saved_at": "2026-05-07T12:11:57",
  "terminals": [
    {"workspace": 4, "tmux_session": "ml", "x": 570, "y": 670, "w": 2214, "h": 1100}
  ],
  "nautilus": [
    {"workspace": 0, "path": "/home/gota", "x": 10, "y": -10, "w": 1404, "h": 824}
  ],
  "brave": [
    {"workspace": 4, "title": "...", "x": 130, "y": 242, "w": 1015, "h": 1013}
  ]
}
```

### Restore Script — Window Placement

**tmux startup sequence:**
1. Creates a temporary `__restore__` tmux session to start the server
2. This triggers tmux-continuum's auto-restore (all saved sessions are recreated)
3. Polls until the first expected session exists (up to 15s)
4. Kills the temporary session
5. Opens `gnome-terminal -- tmux attach-session -t $session` for each saved terminal

**Window-to-workspace assignment:**
For each opened window, the script:
1. Records existing window IDs (`wmctrl -l -x | grep $class`)
2. Launches the app
3. Polls for a new window ID via `comm -13` (set difference) — up to 8s timeout
4. Runs `wmctrl -i -r $wid -t $ws` and `wmctrl -i -r $wid -e 0,$x,$y,$w,$h`

Sticky windows (workspace -1) get `wmctrl -i -r $wid -b add,sticky` instead.

**Brave restore prompt fix:**
Before launching Brave, patches `~/.config/BraveSoftware/Brave-Browser/Default/Preferences`:
```
sed -i 's/"exit_type":"Crashed"/"exit_type":"Normal"/;s/"exited_cleanly":false/"exited_cleanly":true/'
```
This prevents the "Restore tabs?" prompt that appears after unclean shutdown. Combined with `--restore-last-session` flag for extra safety.

### Workspace Matrix Compatibility

Workspace Matrix maps a visual grid to linear workspace indices (e.g., 3x3 grid → indices 0–8). `wmctrl -t` operates on linear indices and works correctly with the extension.

The restore script does NOT use `wmctrl -n` (set workspace count) because it desyncs with the extension's grid config. The 3x3 grid (9 workspaces) is managed entirely by the extension via dconf at `/org/gnome/shell/extensions/wsmatrix/`.

### Systemd Timer

`workspace-save.timer` fires every 5 minutes (after initial 2-minute boot delay). The service sets `Environment=DISPLAY=:1` since this system uses display :1.

Logs rotate automatically (truncated to 200 lines when exceeding 1000).

## Alternatives Considered

- **"Another Window Session Manager" GNOME extension** — Can save/restore window positions but doesn't handle tmux session attachment or Nautilus directory paths reliably.
- **Linux Window Session Manager (lwsm)** — Node.js CLI tool, good for X11 but same tmux limitation.
- **devilspie2** — Lua-based window matching rules, better for static placement rules than dynamic session restore.

## Bug Fix: Silent Crash on First Boot (2026-05-07)

Restore script failed to open any windows after reboot. Log showed tmux started but nothing after.

**Root cause:** `set -euo pipefail` + `grep` returning exit code 1 when no matching windows exist yet. At boot, `get_wids("gnome-terminal")` calls `wmctrl | grep "gnome-terminal" | ...` — grep finds no matches, returns 1, pipefail propagates it, `set -e` kills the script silently.

**Fix:**
- Changed `set -euo pipefail` → `set -uo pipefail` (removed `-e`, keep nounset + pipefail)
- Added `|| true` to the grep pipeline in `get_wids()` and `comm` in `wait_for_new_window()`
- Added `trap ERR` handler that logs the failing line number and content
- Added verbose logging at each stage (desktop ready, tmux session availability, each window open/place result)

**Note on "missing" autostart apps:** The old `gnome-terminal.desktop` and `nautilus.desktop` autostart entries were intentionally disabled since the restore script handles them. They appeared "gone" because the restore script crashed before opening any windows.

## Known Limitations

1. **Nautilus paths with non-unique names** may resolve incorrectly (e.g., "tmp" matching wrong directory). Can be manually corrected in state.json.
2. **Brave window-to-workspace matching** is order-based, not title-based, since titles may change between sessions.
3. **Non-tmux terminal windows** are not saved (no reliable way to detect working directory from gnome-terminal-server's shared PID).
4. **Gnome-terminal tabs** are not individually tracked — only the top-level window with its active tmux session.

## Manual Usage

```bash
# Save current state manually (e.g., right before rebooting to Windows)
~/.local/bin/workspace-save.sh

# Check saved state
cat ~/.cache/workspace-session/state.json | jq .

# Check logs
cat ~/.cache/workspace-session/save.log
cat ~/.cache/workspace-session/restore.log

# Timer status
systemctl --user status workspace-save.timer
```
