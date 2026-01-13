# X11 Crash: DRI Driver Symlinks Point to Old ROCm

**Date:** 2025-01-05
**Status:** FIXED
**Issue:** X11 session crashes, GDM silently falls back to Wayland

## Symptom

After selecting "Ubuntu on Xorg" at GDM login screen, `$XDG_SESSION_TYPE` still shows `wayland`. The session selection appeared to work but X11 never actually started.

```bash
echo $XDG_SESSION_TYPE
# wayland (expected: x11)

loginctl show-session $(loginctl | grep $(whoami) | awk '{print $1}' | head -1) -p Type
# Type=wayland
```

## Investigation

Xorg crash reports existed in `/var/crash/`:
```
_usr_lib_xorg_Xorg.1000.crash
_usr_lib_xorg_Xorg.127.crash
```

Examining the crash report stacktrace:
```bash
grep -a "Stacktrace\|Signal\|opt/amdgpu" /var/crash/_usr_lib_xorg_Xorg.1000.crash
```

Showed the crash occurred in:
```
/opt/amdgpu/lib/x86_64-linux-gnu/dri/radeonsi_dri.so
```

## Root Cause

The system Mesa DRI drivers were **symlinked** to old ROCm drivers:

```bash
ls -la /usr/lib/x86_64-linux-gnu/dri/radeonsi_dri.so
# /usr/lib/x86_64-linux-gnu/dri/radeonsi_dri.so -> /opt/amdgpu/lib/x86_64-linux-gnu/dri/radeonsi_dri.so
```

The old amdgpu-pro/ROCm installation from Ubuntu 22.04 replaced system DRI drivers with symlinks. These symlinks survived the upgrade but point to incompatible drivers.

This is different from the ld.so.conf issue fixed in devlog 018 - that affected shared library loading, but DRI drivers use a separate search mechanism (`LIBGL_DRIVERS_PATH` or built-in paths).

## Fix

**Note:** Simply reinstalling or deleting symlinks doesn't work - the old amdgpu packages have dpkg triggers that recreate the symlinks.

```bash
# These approaches do NOT work:
sudo apt install --reinstall libgl1-mesa-dri
# Symlinks remain - dpkg won't overwrite symlinks from other packages

sudo find /usr/lib/x86_64-linux-gnu/dri/ -type l -lname '/opt/amdgpu/*' -delete
sudo apt install --reinstall libgl1-mesa-dri
# Symlinks get recreated by amdgpu package triggers
```

**Actual fix:** Remove the old amdgpu mesa packages that create the symlinks:

```bash
# Remove old amdgpu display packages (does NOT affect ROCm compute)
sudo apt remove libgl1-amdgpu-mesa-dri
sudo apt remove mesa-amdgpu-va-drivers mesa-amdgpu-omx-drivers mesa-amdgpu-vdpau-drivers

# Reinstall proper system Mesa drivers
sudo apt install --reinstall libgl1-mesa-dri mesa-va-drivers mesa-vdpau-drivers
```

The old packages were from Ubuntu 22.04 ROCm install:
- `libgl1-amdgpu-mesa-dri` (version 23.3.0.60000-1697589.22.04)
- `mesa-amdgpu-va-drivers`
- `mesa-amdgpu-omx-drivers`
- `mesa-amdgpu-vdpau-drivers`

These are display/rendering libraries, separate from the ROCm compute stack (hip, rocblas, etc.).

## Verification

After fix:
```bash
# Should show no symlinks to /opt/amdgpu
find /usr/lib/x86_64-linux-gnu/dri/ -type l -lname '/opt/amdgpu/*' | wc -l
# 0

# DRI driver should point to system Mesa
ls -la /usr/lib/x86_64-linux-gnu/dri/radeonsi_dri.so
# -> libdril_dri.so (NOT /opt/amdgpu/...)

# VA-API driver should point to system Mesa
ls -la /usr/lib/x86_64-linux-gnu/dri/radeonsi_drv_video.so
# -> ../libgallium-*.so (NOT /opt/amdgpu/...)

# Reboot and select "Ubuntu on Xorg"
echo $XDG_SESSION_TYPE
# Should show: x11
```

## Related

- Devlog 018: Initial libdrm and ld.so.conf fixes
- Devlog 019: Incorrectly stated X11 was working (updated with correction)

## Symlinks That Were Affected

9 symlinks in `/usr/lib/x86_64-linux-gnu/dri/` were pointing to `/opt/amdgpu/lib/x86_64-linux-gnu/dri/`:
- `radeonsi_dri.so`
- `radeonsi_drv_video.so`
- `r600_dri.so`
- `r600_drv_video.so`
- `kms_swrast_dri.so`
- `swrast_dri.so`
- `virtio_gpu_dri.so`
- `virtio_gpu_drv_video.so`
- `vmwgfx_dri.so`
