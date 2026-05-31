# War of Dots (Steam 3902430) — Proton Launch Failure on Flatpak Steam

**Date:** 2026-05-31
**OS:** Ubuntu 24.04 LTS, Kernel 6.8.0-111-generic
**GPU:** NVIDIA GeForce RTX 3090 (driver 580.65.6)
**Steam:** Flatpak (`com.valvesoftware.Steam`)
**Game:** War of Dots (AppID 3902430) — PyInstaller-bundled Python 3.12/pygame game

## Symptom

Game refuses to launch from Steam. Clicks Play, button briefly shows "Running", then immediately returns to "Play". No error dialog, no crash dump, no Wine output. Happens with every Proton version tried: 9.0 (Beta), Experimental, 11.0 (Beta).

## Troubleshooting Timeline

### 1. ProtonDB Suggestions — vcrun2022/ucrtbase (failed)

ProtonDB reports pointed to a known Wine bug: missing `crealf()` in Wine's `ucrtbase.dll` ([ValveSoftware/Proton#9494](https://github.com/ValveSoftware/Proton/issues/9494)). Installed via protontricks:

```bash
flatpak run com.github.Matoking.protontricks 3902430 vcrun2022 ucrtbase2019
```

DLL overrides were set in the registry (`native,builtin`). The native ucrtbase.dll was confirmed to export `crealf`. **Did not help.**

### 2. Proton Version Sweep (all failed)

| Proton Version | Runtime | Exit Code |
|---|---|---|
| 9.0 (Beta) | SteamLinuxRuntime_sniper | 255 |
| Experimental | SteamLinuxRuntime_4 | 255 |
| 11.0 (Beta) | SteamLinuxRuntime_4 | 255 |

All die within <1 second. Exit code 255 = the pressure-vessel wrapper itself fails, not Wine.

### 3. Capturing Logs — The Flatpak Challenge

Steam's console-linux.txt and gameprocess_log.txt only showed the outer wrapper dying. Wine/Proton output was trapped inside the pressure-vessel container. Multiple log capture attempts:

- `PROTON_LOG=1` via Steam launch options: no log written (container dies before Proton initializes logging)
- Stderr redirect via launch options: only captured pre-container output (the `ld.so` gameoverlay warning)
- Launch wrapper script with redirects: same — redirects apply before container entry
- `user_settings.py` with `PROTON_LOG=1`: failed for the same reason
- Flatpak sandbox `/tmp` is at `/run/user/1000/.flatpak/com.valvesoftware.Steam/tmp/`

### 4. Direct Wine Test — Game Works

Running the game directly with Proton 11's Wine binary (no container, no Steam):

```bash
WINEPREFIX=".../compatdata/3902430/pfx" WINEDEBUG=-all \
  .../Proton\ 11.0/files/bin/wine .../War\ of\ Dots/game.exe
```

**Result: Game starts successfully**, loads pygame 2.6.1, prints "Connection started" (server connection). One non-fatal warning:

```
ModuleNotFoundError: No module named 'scipy.optimize._cobyla'
```

This confirmed the game itself is compatible with Wine 11.0/Proton 11.

### 5. Proton Script Test — lsteamclient Assertion

Running the Proton wrapper script directly (outside pressure-vessel):

```bash
STEAM_COMPAT_DATA_PATH=... SteamAppId=3902430 \
  python3 .../Proton\ 11.0/proton run .../game.exe
```

Revealed:
```
err:steamclient:steamclient_init unable to load native steamclient library
err:msvcrt:_wassert (L"!status",L"../src-lsteamclient/steamclient_main.c",375)
```

This is the MSVC assertion dialog the user saw. It's from Proton's lsteamclient failing to load `steamclient.so` — expected when running outside Steam's process tree. **Not the root cause for in-Steam launches.**

### 6. Manual pressure-vessel Test — Root Cause Found

Running the exact SteamLinuxRuntime entry point manually:

```bash
.../SteamLinuxRuntime_4/_v2-entry-point --verb=waitforexitandrun -- \
  .../Proton\ 11.0/proton waitforexitandrun .../game.exe
```

**Error:**
```
pressure-vessel-wrap[2294149]: E: Child process exited with code 1: bwrap: setting up uid map: Permission denied
```

## Root Cause

Ubuntu 24.04 enables `kernel.apparmor_restrict_unprivileged_userns=1`, which blocks unprivileged processes from creating user namespaces. Bubblewrap (`bwrap`), used by pressure-vessel to containerize Proton, requires user namespaces.

```bash
$ cat /proc/sys/kernel/apparmor_restrict_unprivileged_userns
1
```

An AppArmor profile exists for native Steam (`/etc/apparmor.d/steam`) which grants `userns` permission, but it only covers `/usr/{lib/steam/bin_steam.sh,games/steam}` — NOT the `bwrap` binary that pressure-vessel uses. **This affects ALL Proton games through Flatpak Steam**, not just War of Dots.

## Fix

### Recommended: Per-Binary AppArmor Profile (targeted)

Create `/etc/apparmor.d/bwrap`:

```
abi <abi/4.0>,
include <tunables/global>

profile bwrap /usr/bin/bwrap flags=(unconfined) {
  userns,

  include if exists <local/bwrap>
}
```

Then:
```bash
sudo apparmor_parser -r /etc/apparmor.d/bwrap
```

This grants user namespace permission ONLY to bwrap, not system-wide.

### NOT Recommended: System-Wide Disable

```bash
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
```

This disables the protection for ALL applications. Real CVEs have exploited unprivileged user namespaces (CVE-2022-0185, CVE-2023-2640, CVE-2023-32629). Other distros (Arch, Fedora) don't restrict this at all, but Ubuntu's approach of per-profile exceptions is strictly safer.

## Prefix Corruption Note

During troubleshooting, the Wine prefix was corrupted by:
1. Multiple Proton version upgrades (9.0 → Experimental → 11.0) on the same prefix
2. protontricks installs of vcrun2022/ucrtbase2019 adding DLL overrides
3. Direct `wine` invocation creating a non-Proton prefix (missing `version`/`config_info` files)

After applying the fix, the prefix should be deleted and recreated cleanly by Proton:

```bash
rm -rf /data2/SteamLibraryFlatpak/SteamLibrary/steamapps/compatdata/3902430
```

Then launch the game from Steam with the desired Proton version.

### 7. AppArmor Profile for bwrap — Applied, Host Fixed

Created `/etc/apparmor.d/bwrap` with `userns` permission and loaded it via `apparmor_parser -r`. Host bwrap test confirmed working:

```bash
$ /usr/bin/bwrap --ro-bind / / --dev /dev --proc /proc echo "bwrap works"
bwrap works
```

**Steam still exits code 255.** The AppArmor fix only covers host `/usr/bin/bwrap`. Inside the Flatpak sandbox, there is no `/usr/bin/bwrap` — pressure-vessel falls back to `bwrap-no-perms` mode (runs without a container).

### 8. Manual Launch Inside Flatpak — Game Works

Running the full pipeline inside the Flatpak sandbox manually:

```bash
flatpak run --command=bash com.valvesoftware.Steam -c '
STEAM_COMPAT_DATA_PATH="/data2/SteamLibraryFlatpak/SteamLibrary/steamapps/compatdata/3902430" \
STEAM_COMPAT_CLIENT_INSTALL_PATH="$HOME/.local/share/Steam" \
SteamAppId=3902430 SteamGameId=3902430 \
~/.local/share/Steam/steamapps/common/SteamLinuxRuntime_4/_v2-entry-point \
--verb=waitforexitandrun -- \
"/data2/SteamLibraryFlatpak/SteamLibrary/steamapps/common/Proton 11.0/proton" \
waitforexitandrun \
"/data2/SteamLibraryFlatpak/SteamLibrary/steamapps/common/War of Dots/game.exe"'
```

**Result: Game launches successfully.** Pressure-vessel uses `bwrap-no-perms` mode (no nested container). Proton 11 creates a valid prefix. Game window appears.

### 9. Steam Launch — Still Fails

Launching from Steam's Play button still exits code 255 instantly. The difference between the working manual launch and Steam's launch:

- **Manual:** `flatpak run --command=bash` → `_v2-entry-point` → `proton` → `game.exe`
- **Steam:** `steam-launch-wrapper` → `reaper` → `_v2-entry-point` → `proton` → `game.exe`

Steam wraps the command with `steam-launch-wrapper` and `reaper SteamLaunch AppId=3902430`. One of these wrappers appears to be the source of exit code 255. The game itself, Wine, Proton, and pressure-vessel all work correctly.

## Current State

| Component | Status |
|---|---|
| Game + Wine 11.0 | Works (direct wine test) |
| Game + Proton 11 script | Works (assertion from lsteamclient is non-fatal outside Steam) |
| Game + pressure-vessel + Proton 11 (manual Flatpak) | Works |
| Game + Steam Play button | **Broken** — exit code 255 from wrapper chain |
| AppArmor bwrap profile | Applied, host bwrap works, not relevant inside Flatpak |
| Wine prefix | Valid (created by manual Flatpak launch, version 11.0-100) |
| Old prefixes | Backed up as `3902430.bak` and `3902430.bak2` |

## Remaining Issue

The exit code 255 comes from Steam's `steam-launch-wrapper` or `reaper`, not from Proton or the game. Possible causes:

1. **reaper PID tracking** — the reaper may misinterpret pressure-vessel's `bwrap-no-perms` process tree as a failed launch
2. **steam-launch-wrapper environment** — may set up the environment differently than a direct `flatpak run --command=bash` invocation
3. **SteamTinkerLaunch conflict** — compat_log showed STL was briefly mapped to this game before being switched to Proton; residual config may interfere

## Next Steps

- Verify no SteamTinkerLaunch override remains for this game
- Try `STEAM_LINUX_RUNTIME_LOG=1` or `STEAM_LINUX_RUNTIME_VERBOSE=1` in launch options to capture wrapper-level errors
- Try launch option: `steam-launch-wrapper -- %command%` bypass
- Check if other Proton games also fail from Steam (The Finals last worked April 18, before kernel update on May 6)
