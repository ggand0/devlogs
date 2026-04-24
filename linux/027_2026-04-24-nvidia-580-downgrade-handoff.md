# HANDOFF: NVIDIA Driver Downgrade 595 → 580 for Isaac Sim

**Date:** 2026-04-24
**Author:** linux-agent (gaming/driver context)
**For:** Isaac Sim installation agent
**Hardware:** NVIDIA RTX 3090, Ubuntu 24.04 LTS
**Current state:** Driver 595.58.03, CUDA 13.2, kernel 6.8.0-110-generic

## Goal

Downgrade the NVIDIA driver from 595.58.03 to 580.x for Isaac Sim compatibility.
The latest available 580 candidate in apt is **580.126.20-1ubuntu1**.

## Critical Concerns (ranked by risk)

### 1. Kernel / DKMS — HIGH RISK

The current kernel is **6.8.0-110-generic**. DKMS currently has 595.58.03 built for it.
Driver 580 has **never been built on this machine for any kernel**. During install, DKMS
will attempt to compile the 580 kernel module against 6.8.0-110.

**If DKMS fails:** The system reboots with no GPU driver loaded — no desktop, no CUDA,
recovery-mode-only. This has happened before on this machine during driver transitions
(see devlog 024).

**Mitigation:**
- Check DKMS build result during install — do NOT reboot if it reports errors.
- Keep a fallback kernel with a known-good driver. Currently installed kernels:
  - `6.8.0-110-generic` (running, 595.58.03 built)
  - `6.8.0-107-generic` (595.58.03 built)
  - `6.8.0-90-generic` (595.58.03 built)
  - `6.2.0-39-generic` (595.58.03 built)
- None of these have ever had a 580 module built. There is **no safe fallback kernel
  for 580**. If DKMS fails on 6.8.0-110, boot back into 6.8.0-110 with the old 595
  module still cached (it won't be removed until overwritten).
- If DKMS fails, abort the downgrade and re-install 595:
  ```bash
  sudo apt install nvidia-driver-open
  ```

### 2. CUDA Compatibility — HIGH RISK

Current CUDA stack:
- **System CUDA toolkit:** 13.1 and 13.2 (both installed side-by-side)
- **PyTorch:** 2.9.1+cu128 (CUDA 12.8 runtime bundled)
- **Driver CUDA version:** 595.58.03 reports CUDA 13.2 capability

Driver 580 is from an older branch and will report a **lower CUDA version** than 13.2.
This means:
- The system CUDA 13.2 toolkit **may not work** — the driver may not support all 13.2
  APIs. CUDA has a forward-compatibility model (new driver supports old toolkit) but
  NOT backward (old driver does NOT support new toolkit).
- PyTorch's bundled CUDA 12.8 runtime **should work** — 580 almost certainly supports
  CUDA 12.8, but verify with `python3 -c "import torch; print(torch.cuda.is_available())"`.
- Isaac Sim will bring its own CUDA requirements — check what version it expects and
  whether 580's reported CUDA version covers it.

**Key question for Isaac Sim agent:** What CUDA version does Isaac Sim require? If it
needs CUDA 13.x, the 580 driver may not provide it, creating a catch-22 (need 580 for
Isaac Sim compatibility but 580 can't run Isaac Sim's CUDA).

**After downgrade, verify:**
```bash
nvidia-smi  # check reported CUDA version
python3 -c "import torch; print(torch.cuda.is_available())"
```

**DO NOT run `apt autoremove`** after the downgrade — it will remove CUDA 13.1/13.2
toolkit packages that lost their `cuda` metapackage dependency.

### 3. Flatpak / Steam Gaming — MEDIUM RISK

Steam is installed as a **Flatpak**. Flatpak sandboxes its own GL/Vulkan runtime matched
to the host driver version. After any driver change, a matching
`org.freedesktop.Platform.GL.nvidia-XXX` runtime must exist.

**Currently installed Flatpak GL runtimes:**
- `nvidia-515-48-07` (ancient, unused)
- `nvidia-590-48-01` (from previous driver)
- `nvidia-595-58-03` (current)

There is **no 580.x runtime installed**. After downgrade:
```bash
flatpak update
```
This should pull `org.freedesktop.Platform.GL.nvidia-580-126-20` (and GL32 variant).
If it doesn't auto-pull:
```bash
flatpak install flathub org.freedesktop.Platform.GL.nvidia-580-126-20
flatpak install flathub org.freedesktop.Platform.GL32.nvidia-580-126-20
```

**If no matching runtime exists on Flathub:** All Flatpak Steam games fall back to CPU
software rendering (llvmpipe). This is the exact failure documented in devlog 024 Issue 2.
The game process will show ~1300% CPU and 0% GPU in nvidia-smi.

**After Flatpak update, fully restart Steam** (not just the game) — Flatpak loads the GL
runtime at Steam startup.

### 4. Proton / Game-Specific Notes — LOW RISK

- Currently using **Proton-GE** (installed via Flatpak compatibility tool). This resolved
  prior stuttering issues on 595. Should work on 580 with no changes.
- DXVK/VKD3D shader caches will be invalidated by the driver change. Expect a full
  shader recompilation stutter pass on first launch of each game.
- **Do NOT use these launch options** — they cause CPU rendering fallback on this Flatpak
  setup regardless of driver version:
  - `PROTON_ENABLE_NVAPI=1`
  - `PROTON_USE_NTSYNC=1`
- Working launch options (The Finals): `LD_PRELOAD="" PROTON_HIDE_NVIDIA_GPU=0 PROTON_LOCAL_SHADER_CACHE=1 %command%`

## Downgrade Procedure

The 580 branch uses **versioned package names** (`nvidia-driver-580-open`) rather than
the unversioned names used by 590/595 (`nvidia-driver-open`). The install command is:

```bash
sudo apt install nvidia-driver-580-open
```

This will:
- Install the 580.126.20 driver packages
- Trigger DKMS build for kernel 6.8.0-110-generic
- The old 595 packages (`nvidia-driver-open`) should be auto-removed or will coexist

**Watch the DKMS output carefully.** If it fails, do not reboot.

### Post-downgrade checklist

1. Verify DKMS: `dkms status` — should show `nvidia/580.126.20` for 6.8.0-110
2. Reboot
3. Verify: `nvidia-smi` — should show 580.126.20
4. Verify CUDA: `python3 -c "import torch; print(torch.cuda.is_available())"`
5. Run `flatpak update` for Steam GL runtime
6. Verify: `flatpak list --runtime | grep nvidia-580`
7. Hold packages to prevent re-upgrade:
   ```bash
   sudo apt-mark hold nvidia-driver-580-open nvidia-dkms-580-open
   ```
8. Test Isaac Sim
9. Test a Steam game (The Finals) to confirm GPU rendering

## Rollback

If anything breaks, boot into recovery and re-install the current driver:

```bash
sudo apt install nvidia-driver-open=595.58.03-1ubuntu1
sudo reboot
```

The 595.58.03 DKMS modules are already built for all installed kernels.

## Related Devlogs

- `024_2026-03-31-nvidia-driver-upgrade-mismatch-fix.md` — full incident report for
  590→595 upgrade breaking gaming (driver mismatch, Flatpak GL, stuttering)
- `025_2026-04-05-nvidia-driver-downgrade-plan.md` — planned 595→590 downgrade (never
  executed, stutters resolved with Proton-GE instead)
- `026_2026-04-07-wsmatrix-gpu-usage-fix.md` — GNOME extension GPU waste fix (unrelated
  to driver version but relevant to GPU baseline usage)
