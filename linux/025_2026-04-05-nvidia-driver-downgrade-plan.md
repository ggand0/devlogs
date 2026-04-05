# NVIDIA Driver Downgrade Plan: 595.58.03 → 590.48.01

**Date:** 2026-04-05
**Hardware:** NVIDIA RTX 3090
**OS:** Ubuntu 24.04 LTS, kernel 6.8.0-106-generic
**Reason:** Game stutters in The Finals started after driver upgrade from 590.48.01 to 595.58.03. Game worked fine on 590.48.01.

## Status: NOT YET EXECUTED

## What the dry run showed

`apt install --simulate` with all 25 nvidia packages pinned to 590.48.01-0ubuntu1:

- **25 packages downgraded** — all nvidia driver/lib packages, consistent at 590.48.01
- **1 package removed** — `cuda` metapackage (just the pointer, not the actual toolkit)
- **No dependency conflicts reported**
- CUDA 13.1 and 13.2 toolkit packages listed as "no longer required" but NOT removed
  unless `apt autoremove` is run (DO NOT run autoremove)

## Verification

**DKMS (no risk if booting 6.8.0-94):** 590.48.01 has never been built for kernel
6.8.0-106-generic — only 595.58.03 has. But 6.8.0-94-generic is still installed
and 590.48.01 was already built for it. Boot into 6.8.0-94 from GRUB after the
downgrade to get the exact previous working combo. DKMS will also attempt to build
590 for 6.8.0-106 during install — if that succeeds, both kernels work.

**CUDA 13.2 (low risk):** `cuda-runtime-13-2` → `cuda-libraries-13-2` → individual
toolkit libs (cublas, cufft, etc.). None of these have an apt dependency on a minimum
nvidia driver version. The `cuda` metapackage is removed but the actual toolkit
packages stay. Runtime compatibility is not guaranteed by apt metadata alone, but
590.48.01 is in the same 590 driver branch and was packaged alongside CUDA in
NVIDIA's repo.

**Flatpak GL runtime (no risk):** Both `org.freedesktop.Platform.GL.nvidia-590-48-01`
and `GL32.nvidia-590-48-01` are already installed from when the system was on 590.
No Flatpak changes needed — just restart Steam after reboot.

**Root cause uncertainty:** The stutter may not be the driver. Mesa was also upgraded
(25.0.7 → 25.2.8) in the same apt upgrade. If stutters persist after downgrade, Mesa
would be the next suspect.

## Downgrade command

```bash
sudo apt install \
  nvidia-driver-open=590.48.01-0ubuntu1 \
  nvidia-dkms-open=590.48.01-0ubuntu1 \
  nvidia-open=590.48.01-0ubuntu1 \
  nvidia-kernel-common=590.48.01-0ubuntu1 \
  nvidia-kernel-source-open=590.48.01-0ubuntu1 \
  nvidia-firmware=590.48.01-0ubuntu1 \
  nvidia-modprobe=590.48.01-0ubuntu1 \
  nvidia-persistenced=590.48.01-0ubuntu1 \
  nvidia-settings=590.48.01-0ubuntu1 \
  libnvidia-gl=590.48.01-0ubuntu1 \
  libnvidia-gl:i386=590.48.01-0ubuntu1 \
  libnvidia-compute=590.48.01-0ubuntu1 \
  libnvidia-compute:i386=590.48.01-0ubuntu1 \
  libnvidia-decode=590.48.01-0ubuntu1 \
  libnvidia-decode:i386=590.48.01-0ubuntu1 \
  libnvidia-encode=590.48.01-0ubuntu1 \
  libnvidia-encode:i386=590.48.01-0ubuntu1 \
  libnvidia-fbc1=590.48.01-0ubuntu1 \
  libnvidia-fbc1:i386=590.48.01-0ubuntu1 \
  libnvidia-extra=590.48.01-0ubuntu1 \
  libnvidia-cfg1=590.48.01-0ubuntu1 \
  libnvidia-gpucomp=590.48.01-0ubuntu1 \
  libnvidia-gpucomp:i386=590.48.01-0ubuntu1 \
  xserver-xorg-video-nvidia=590.48.01-0ubuntu1 \
  libnvidia-common=590.48.01-0ubuntu1
```

## Post-downgrade steps

1. Reboot — hold **Shift** at GRUB → **Advanced options** → select **6.8.0-94-generic**
   (this is the exact kernel that was running with 590.48.01 before the upgrade)
2. Verify: `nvidia-smi` should show 590.48.01
3. Flatpak GL runtime for 590 is already installed — just restart Steam
4. Verify: `flatpak list --runtime | grep nvidia-590`
5. Hold packages to prevent re-upgrade:
   ```bash
   sudo apt-mark hold nvidia-driver-open nvidia-dkms-open nvidia-open
   ```
6. **DO NOT run `apt autoremove`** — it would remove CUDA 13.1/13.2 toolkit packages
7. Test The Finals for stutters
8. Test CUDA: `python3 -c "import torch; print(torch.cuda.is_available())"`

## If it breaks

- Boot into recovery mode (hold Shift at GRUB)
- Re-upgrade back to 595.58.03:
  ```bash
  sudo apt install nvidia-driver-open
  ```
  (apt will pull the latest candidate, 595.58.03)
- Reboot
