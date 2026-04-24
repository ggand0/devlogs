# RETURN HANDOFF: NVIDIA Driver Downgrade Complete

**Date:** 2026-04-24
**Author:** real2sim agent (Isaac Sim installation context)
**For:** linux-agent (gaming/driver context)
**Reference:** `027_2026-04-24-nvidia-580-downgrade-handoff.md` (your original handoff)

## What was done

Driver downgraded from 595.58.03 to **580.65.06** for Isaac Sim 5.1.0 compatibility. Isaac Sim now launches without the RTX renderer segfault.

### Packages installed (all pinned to 580.65.06-0ubuntu1)

26 packages explicitly pinned:
- **Driver core**: nvidia-driver-580-open, nvidia-dkms-580-open, nvidia-kernel-common-580, nvidia-kernel-source-580-open, nvidia-firmware-580
- **Userspace libs (amd64)**: libnvidia-gl-580, libnvidia-compute-580, libnvidia-decode-580, libnvidia-encode-580, libnvidia-fbc1-580, libnvidia-cfg1-580, libnvidia-gpucomp-580, libnvidia-common-580, libnvidia-extra-580
- **Utilities**: nvidia-compute-utils-580, nvidia-utils-580, xserver-xorg-video-nvidia-580
- **Downgraded from 595**: nvidia-persistenced, nvidia-modprobe, nvidia-settings
- **i386 libs**: libnvidia-compute-580:i386, libnvidia-decode-580:i386, libnvidia-encode-580:i386, libnvidia-fbc1-580:i386, libnvidia-gl-580:i386, libnvidia-gpucomp-580:i386

### Side effects

- 62 CUDA 13.2 toolkit packages were upgraded to newer point releases (e.g. 13.2.51→13.2.78). Toolkit still works, just newer patch versions.
- `nvidia-smi` now reports CUDA 13.0 (down from 13.2). System CUDA 13.1/13.2 toolkits are still installed on disk but runtime support is limited to 13.0.
- The `cuda` metapackage was removed (dependency chain broke). CUDA toolkits themselves are intact.

## Tasks for you

### 1. Flatpak GL runtime (HIGH — Steam won't GPU-render without this)

No 580.65.06 Flatpak GL runtime is installed. Currently installed:
```
nvidia-515-48-07  (ancient)
nvidia-590-48-01  (previous driver)
nvidia-595-58-03  (just replaced)
```

Run:
```bash
flatpak update
```

If it doesn't auto-pull the 580 runtime:
```bash
flatpak install flathub org.freedesktop.Platform.GL.nvidia-580-65-06
flatpak install flathub org.freedesktop.Platform.GL32.nvidia-580-65-06
```

Restart Steam fully after (not just the game).

### 2. Hold packages (HIGH — prevent apt from upgrading back to 595)

```bash
sudo apt-mark hold nvidia-driver-580-open nvidia-dkms-580-open nvidia-persistenced nvidia-modprobe nvidia-settings
```

### 3. apt autoremove protection (HIGH — CUDA toolkits at risk)

apt now lists ALL CUDA 13.1 and 13.2 toolkit packages as "no longer required." **DO NOT run `apt autoremove`** until you've verified which packages are safe to remove. At minimum, `cuda-toolkit-13-2` should be kept.

### 4. Test Steam/Proton (MEDIUM)

- Expect DXVK/VKD3D shader cache invalidation — full recompilation stutter on first launch
- Test The Finals with existing launch options: `LD_PRELOAD="" PROTON_HIDE_NVIDIA_GPU=0 PROTON_LOCAL_SHADER_CACHE=1 %command%`
- Confirm GPU rendering in `nvidia-smi` (should show game process, not llvmpipe)

### 5. Clean up old Flatpak runtimes (LOW)

After confirming 580 runtime works, the old 595 and 590 runtimes can be removed:
```bash
flatpak uninstall org.freedesktop.Platform.GL.nvidia-595-58-03
flatpak uninstall org.freedesktop.Platform.GL32.nvidia-595-58-03
```

## Current driver state

```
Driver: 580.65.06
CUDA:   13.0
DKMS:   nvidia/580.65.06 built for 4 kernels (6.2.0-39, 6.8.0-90, 6.8.0-107, 6.8.0-110)
```

## Rollback (if needed)

```bash
sudo apt install --allow-downgrades nvidia-driver-open=595.58.03-1ubuntu1
sudo reboot
```

595 DKMS modules are no longer pre-built (they were removed during the swap), so this would trigger a DKMS rebuild.

## Related files

- Swap script: `~/ggando/ml/real2sim-so101/tmp/swap_nvidia_driver.sh`
- Swap log: `~/ggando/ml/real2sim-so101/logs/driver_swap0.log`
- Pre-downgrade snapshot: `~/ggando/ml/real2sim-so101/devlogs/001a_pre-downgrade-snapshot.md`
- Your original handoff: `/data/devlogs/linux/027_2026-04-24-nvidia-580-downgrade-handoff.md`
