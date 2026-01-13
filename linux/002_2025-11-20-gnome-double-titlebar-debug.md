# GNOME Double Titlebar Bug - Debug Log

**Date:** 2025-11-20
**System:** Ubuntu 22.04
**Issue:** Duplicate window decorations (two sets of titlebar buttons) on all GNOME applications after switching back from i3

---

## Problem Description

After installing i3 window manager and using it for a session, then logging back into the Ubuntu/GNOME session, all GNOME applications (Nautilus, GNOME Terminal, etc.) displayed two sets of window controls:
1. One titlebar at the top (GNOME/Mutter server-side decoration)
2. Another duplicate set below it (GTK3 client-side decoration)

## Investigation Steps

### 1. Environment Check
```bash
XDG_CURRENT_DESKTOP: Unity
DESKTOP_SESSION: ubuntu
GDMSESSION: ubuntu
```

**Finding:** The "Ubuntu" session at login is actually a customized GNOME session that reports itself as "Unity" for backward compatibility (legacy from when Ubuntu used Unity desktop).

### 2. Window Manager Status
- Confirmed GNOME Shell (Mutter) running correctly
- No i3 processes lingering
- GNOME extensions already disabled by user

### 3. GTK Configuration Discovery

**Critical finding:**
```bash
$ env | grep GTK
GTK_CSD=0
```

**What this means:**
- `GTK_CSD=0` disables Client-Side Decorations
- Forces window manager to draw titlebars (Server-Side Decorations)
- BUT modern GNOME apps are coded to draw their own titlebars
- Result: Both window manager AND application draw titlebars = **double titlebars**

### 4. Root Cause Identification

Located the source in [/etc/X11/Xsession.d/51gtk3-nocsd-detect](/etc/X11/Xsession.d/51gtk3-nocsd-detect):

```bash
case "$XDG_CURRENT_DESKTOP" in
  *GNOME*|*Gnome*)
      # This is GNOME, make sure GTK_CSD is not set to 0
      if [ x"$GTK_CSD"x = x"0"x ] ; then
          unset GTK_CSD
      fi
      ;;
  *)
      # not GNOME, and the user didn't specify GTK_CSD
      if [ -z "$GTK_CSD" ] ; then
          GTK_CSD=0
      fi
      export GTK_CSD
      ;;
esac
```

**The bug:** Script checks if `XDG_CURRENT_DESKTOP` contains "GNOME", but Ubuntu session sets it to "Unity", which doesn't match. Therefore it treats it as a non-GNOME desktop and sets `GTK_CSD=0`.

### 5. Package Investigation

**Installed package causing the issue:**
```bash
ii  gtk3-nocsd                    3-1ubuntu1      all
ii  libgtk3-nocsd0:amd64          3-1ubuntu1      amd64
```

**Important:** `gtk3-nocsd` is NOT a dependency of i3, i3status, i3lock, dmenu, feh, or arandr. It was installed separately (unknown when/why).

**Packages installed for i3 setup:**
```bash
sudo apt install i3
sudo apt install i3status i3lock dmenu feh
sudo apt install arandr
```

None of these depend on gtk3-nocsd.

## Technical Explanation

### What is GTK_CSD?

**Client-Side Decorations (CSD):**
- Modern approach where applications draw their own titlebars
- Used by GNOME apps (Nautilus, GNOME Terminal, etc.)
- Allows apps to customize titlebars (embed buttons, search bars, etc.)
- `GTK_CSD=1` or unset = enabled

**Server-Side Decorations (SSD):**
- Traditional X11 approach
- Window manager (i3, KDE, XFCE) draws uniform titlebars
- `GTK_CSD=0` = forces server-side decorations

### What is LD_PRELOAD?

When `GTK_CSD=0` is set, the system also sets:
```bash
LD_PRELOAD="libgtk3-nocsd.so.0"
```

- `LD_PRELOAD` loads a library before all others when programs start
- `libgtk3-nocsd.so.0` intercepts GTK3 functions
- Forces all GTK apps to NOT use client-side decorations
- Designed for non-GNOME window managers like i3, KDE, XFCE

### Why The Double Titlebar?

1. `gtk3-nocsd` package is installed
2. Login to "Ubuntu" session (which is GNOME)
3. Session sets `XDG_CURRENT_DESKTOP=Unity`
4. Script [51gtk3-nocsd-detect](/etc/X11/Xsession.d/51gtk3-nocsd-detect) sees "Unity" ≠ "GNOME"
5. Script sets `GTK_CSD=0` and `LD_PRELOAD=libgtk3-nocsd.so.0`
6. GNOME apps try to draw CSD titlebars (as coded)
7. Mutter/GNOME Shell also draws SSD titlebars (because CSD disabled)
8. **Result:** Two titlebars!

## Hypothesis: What Happened?

**Timeline reconstruction (FINAL - CORRECTED AGAIN):**

1. **December 21, 2020**: `gtk3-nocsd` package was installed (stat shows modification date 2020-12-21)
   - NOT installed today with i3
   - Installed when experimenting with Unity or another window manager back in 2020

2. **2020-2025**: Used Ubuntu/GNOME session successfully without issues
   - User confirmed using "Ubuntu" session before (not "GNOME Classic")
   - `gtk3-nocsd` package was present but NOT triggering the bug
   - `GTK_CSD` was NOT set in systemd/DBus environment

3. **Today (November 20, 2025)**:

   **02:22:43** - First login to Ubuntu session
   - `XDG_CURRENT_DESKTOP=ubuntu:GNOME` set correctly
   - No `GTK_CSD` in environment → apps work normally

   **02:28** - Installed i3 and related packages
   - `sudo apt install i3` (installed i3-wm, i3status, i3lock, and dependencies)
   - `sudo apt install i3status i3lock dmenu feh`
   - `sudo apt install arandr`
   - **gtk3-nocsd was NOT installed today** - it was already present since 2020

   **02:31:43** - Logged into i3 session
   - Boot log shows: `dbus-update-activation-environment: setting GTK_CSD=0`
   - **`GTK_CSD=0` is now stored in systemd user environment (persists across sessions!)**

   **03:22:37** - Logged back into Ubuntu session
   - `XDG_CURRENT_DESKTOP=ubuntu:GNOME` set correctly
   - **BUT**: `GTK_CSD=0` from i3 session is STILL in systemd environment
   - All GNOME apps inherit `GTK_CSD=0` on startup
   - **First time experiencing double titlebars**

**Root Cause (CONFIRMED):**

The i3 session set `GTK_CSD=0` in the **systemd user environment** (via `dbus-update-activation-environment`), which persists across login sessions until logout/reboot. This environment is inherited by all user applications, including GNOME apps.

Before trying i3, `GTK_CSD` was simply not set at all in the systemd environment, so GNOME apps used their default behavior (CSD enabled).

## Solution

### Immediate Fix (Current Session)

**Step 1:** Clear the environment variables from systemd
```bash
systemctl --user unset-environment GTK_CSD LD_PRELOAD
```

**Step 2:** Kill and restart GNOME apps (they cached the old environment)
```bash
killall nautilus gnome-terminal-server
# Then reopen the apps - they'll start with clean environment
```

**Alternative:** Just restart GNOME Shell (Alt+F2 → r) and close/reopen affected apps.

### Permanent Fix (Prevent Recurrence)

Since i3 will be used again and will keep setting `GTK_CSD=0` in systemd environment, add this to `~/.profile`:

```bash
# Disable gtk3-nocsd when using GNOME/Unity sessions
# This prevents GTK_CSD from being set even if it persists in systemd environment
if [[ "$XDG_CURRENT_DESKTOP" == *"Unity"* ]] || [[ "$XDG_CURRENT_DESKTOP" == *"GNOME"* ]]; then
    export GTK3_NOCSD_IGNORE=1
    # Also unset if already set by session scripts
    systemctl --user unset-environment GTK_CSD LD_PRELOAD 2>/dev/null
fi
```

This will:
1. Tell gtk3-nocsd to ignore GNOME sessions
2. Clean up any lingering environment variables from previous i3 sessions

### Alternative Solutions:

**Option 1:** Remove gtk3-nocsd entirely (if i3 won't be used much)
```bash
sudo apt remove gtk3-nocsd libgtk3-nocsd0
```

**Option 2:** Just reboot after switching from i3 to GNOME
- Simplest but requires reboot each time you switch

## Files Examined

- [/etc/X11/Xsession.d/51gtk3-nocsd-detect](/etc/X11/Xsession.d/51gtk3-nocsd-detect) - Main script setting GTK_CSD
- [/etc/X11/Xsession.d/01gtk3-nocsd](/etc/X11/Xsession.d/01gtk3-nocsd) - gtk3-nocsd initialization
- [/usr/share/xsessions/ubuntu.desktop](/usr/share/xsessions/ubuntu.desktop) - Ubuntu session definition
- [~/.config/gtk-3.0/settings.ini](/home/gota/.config/gtk-3.0/settings.ini) - GTK3 user settings
- [~/.config/gtk-4.0/settings.ini](/home/gota/.config/gtk-4.0/settings.ini) - GTK4 user settings

## References

- [AskUbuntu: Double title bars in Ubuntu 20.04](https://askubuntu.com/questions/1252897/double-title-bars-two-title-bars-appear-in-ubuntu-20-04)
- Cause identified as conflict between multiple desktop environments (KDE + GNOME in that case, i3 + GNOME in ours)

---

## Status: RESOLVED ✓

**Resolution timeline:**
1. **Diagnosis confirmed**: Rebooted after running `systemctl --user unset-environment GTK_CSD LD_PRELOAD` - double titlebars were gone, confirming the root cause
2. **Permanent fix implemented**: Added GTK3_NOCSD_IGNORE block to [~/.profile:22-28](/home/gota/.profile#L22-L28) to prevent recurrence
3. **Testing**: Will be tested on next i3 → GNOME session switch

**What was changed:**
- Modified [~/.profile](/home/gota/.profile) to disable gtk3-nocsd for GNOME/Unity sessions and auto-clean systemd environment variables