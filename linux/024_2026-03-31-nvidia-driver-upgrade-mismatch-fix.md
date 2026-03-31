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

After reboot, The Finals ran but appeared to render on CPU. `nvidia-smi` showed no game
process using the GPU. `vulkaninfo --summary` showed two Vulkan devices:

1. NVIDIA GeForce RTX 3090 (real GPU)
2. llvmpipe (CPU software renderer)

### Cause

The same apt upgrade also updated **Mesa from 25.0.7 to 25.2.8**. The new Mesa version
installed/enabled `lvp_icd.json` — the Vulkan ICD for Lavapipe (llvmpipe's Vulkan frontend),
a software Vulkan implementation that runs entirely on CPU.

Before the reboot, old Mesa libraries were still in memory so llvmpipe wasn't visible as a
Vulkan device. After reboot, the new Mesa loaded, llvmpipe appeared as a Vulkan device, and
the game (via Proton/DXVK) picked it over the NVIDIA GPU.

### Fix

Disable the Lavapipe ICD by renaming it:

```bash
sudo mv /usr/share/vulkan/icd.d/lvp_icd.json /usr/share/vulkan/icd.d/lvp_icd.json.disabled
```

Verify only the NVIDIA device remains:

```bash
vulkaninfo --summary 2>&1 | grep deviceName
```

To undo:

```bash
sudo mv /usr/share/vulkan/icd.d/lvp_icd.json.disabled /usr/share/vulkan/icd.d/lvp_icd.json
```

### Note

- `VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json %command%` as a Steam launch
  option does NOT work — it overrides all ICD discovery too aggressively and crashes the game
  (exit code 3).
- Lavapipe (lvp) is only useful for Vulkan development/testing on headless machines or VMs.
  Useless on a system with a real GPU.
- A future Mesa update may recreate `lvp_icd.json`. If games break again after an update,
  rename it again.

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

On a desktop with an NVIDIA GPU, `apt upgrade` can silently break things in two ways:
the driver/library mismatch (requires reboot), and Mesa updates adding software renderers
that steal Vulkan device selection from the real GPU. Don't upgrade casually without being
ready to reboot and verify.
