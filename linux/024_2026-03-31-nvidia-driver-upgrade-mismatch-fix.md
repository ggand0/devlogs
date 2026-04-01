# NVIDIA Driver Upgrade Mismatch & Vulkan ICD Fix

**Date:** 2026-03-31
**Hardware:** NVIDIA RTX 3090
**OS:** Ubuntu 24.04 LTS, kernel 6.8.0-94 → 6.8.0-106
**Driver:** 590.48.01 → 595.58.03

## Issue 1: nvidia-smi driver/library version mismatch

### Symptom

```
$ nvidia-smi
Failed to initialize NVML: Driver/library version mismatch
NVML library version: 595.58
```

### Cause

`apt upgrade` on 2026-03-30 upgraded `nvidia-driver-open` from 590.48.01 to 595.58.03,
updating all userspace libraries (libnvidia-gl, libnvidia-compute, etc.) to 595.58.03.
The kernel module still loaded in memory was the old 590.48.01. NVML detects the mismatch
and refuses to initialize.

This happens silently — the software updater gives no warning that a reboot is required.

### Fix

Reboot. The new kernel module (595.58.03, already built by DKMS) loads on boot.

### Prevention

**Option A: Hold nvidia packages** so they don't auto-upgrade:

```bash
sudo apt-mark hold nvidia-driver-open nvidia-dkms-open nvidia-open
```

Upgrade manually when ready to reboot:

```bash
sudo apt-mark unhold nvidia-driver-open nvidia-dkms-open nvidia-open
sudo apt upgrade
sudo reboot
```

**Option B: apt post-hook warning** — prints a message after any apt operation if the
loaded nvidia kernel module doesn't match the installed package:

```bash
sudo tee /etc/apt/apt.conf.d/99nvidia-reboot-warning << 'EOF'
DPkg::Post-Invoke { "if dpkg-query -W -f='${Status}' nvidia-driver-open 2>/dev/null | grep -q 'install ok installed' && [ -f /proc/driver/nvidia/version ]; then LOADED=$(sed -n 's/.*Module  *\([0-9.]*\).*/\1/p' /proc/driver/nvidia/version); INSTALLED=$(dpkg-query -W -f='${Version}' nvidia-driver-open | cut -d- -f1); if [ \"$LOADED\" != \"$INSTALLED\" ]; then echo ''; echo '*** NVIDIA DRIVER UPDATED — REBOOT REQUIRED ***'; echo \"    Loaded: $LOADED  Installed: $INSTALLED\"; echo ''; fi; fi"; };
EOF
```

**General rule:** Don't run the software updater unless ready to reboot. NVIDIA kernel
modules cannot be hot-swapped.

---

## Issue 2: Steam game (The Finals) rendering on CPU after reboot

### Symptom

After reboot, The Finals ran but rendered entirely on CPU. `nvidia-smi` showed no game
process using the GPU. The game process (`Discovery-d.exe`) consumed ~1286% CPU and 18GB
RAM — clearly software rendering.

### Cause

Steam is installed as a **Flatpak**. Flatpak uses its own GL/Vulkan runtime extensions
matched to the host nvidia driver version. The installed Flatpak GL runtimes were:

- `org.freedesktop.Platform.GL.nvidia-590-48-01` (old driver)
- `org.freedesktop.Platform.GL.nvidia-515-48-07` (ancient)

After `apt upgrade` bumped the host driver to **595.58.03**, there was no matching Flatpak
GL runtime. Flatpak Steam couldn't load nvidia GL/Vulkan libraries, so Proton/DXVK fell
back to software rendering (llvmpipe).

This was NOT a host Vulkan ICD issue — `vulkaninfo` on the host correctly showed the
NVIDIA GPU. The problem was entirely inside the Flatpak sandbox.

### Fix

Update Flatpak runtimes to pull in the matching nvidia GL extension:

```bash
flatpak update
```

Verify the correct extensions are installed:

```bash
flatpak list --runtime | grep nvidia-595
```

Should show both:
- `org.freedesktop.Platform.GL.nvidia-595-58-03`
- `org.freedesktop.Platform.GL32.nvidia-595-58-03`

If `flatpak update` doesn't pull them automatically:

```bash
flatpak install flathub org.freedesktop.Platform.GL.nvidia-595-58-03
flatpak install flathub org.freedesktop.Platform.GL32.nvidia-595-58-03
```

After installing, **fully restart Steam** (not just the game) — Flatpak loads the GL
runtime at Steam startup.

### Dead ends tried

- **Disabling Lavapipe ICD** (`lvp_icd.json` → `.disabled`): This only affects the host
  Vulkan loader, not the Flatpak sandbox. Was a red herring.
- **`VK_ICD_FILENAMES` Steam launch option**: Overrides ICD discovery too aggressively,
  crashed the game (exit code 3).

### Note

- First launch after fix froze on GNOME workspace switch, worked on second launch.
- After any nvidia driver upgrade via apt, `flatpak update` must also be run to pull
  the matching GL runtime. This is easy to forget.
- Old nvidia GL runtimes (515, 590) can be cleaned up:
  ```bash
  flatpak uninstall --unused
  ```

---

## Summary of packages upgraded (2026-03-30)

Key packages that caused the issues:

| Package | Old | New |
|---|---|---|
| nvidia-driver-open | 590.48.01 | 595.58.03 |
| nvidia-dkms-open | 590.48.01 | 595.58.03 |
| mesa-libgallium | 25.0.7 | 25.2.8 |
| mesa-vulkan-drivers | 25.0.7 | 25.2.8 |
| libglx-mesa0 | 25.0.7 | 25.2.8 |
| cuda | 13.1.1 | 13.2.0 |
| linux-image-generic | 6.8.0-90 | 6.8.0-106 |

## Lesson

On a desktop with an NVIDIA GPU and Flatpak Steam, `apt upgrade` can silently break things
in two ways:

1. **Driver/library mismatch** — requires reboot, no way around it.
2. **Flatpak GL runtime mismatch** — `flatpak update` must be run after any nvidia driver
   upgrade to pull matching GL extensions. Without this, Flatpak-sandboxed apps (Steam/games)
   fall back to CPU software rendering.

Don't upgrade casually without being ready to reboot, run `flatpak update`, and verify.
