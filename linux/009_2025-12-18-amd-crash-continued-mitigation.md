# AMD 7900 XTX Crash Mitigation - Continued

**Date:** 2025-12-18
**Issue:** GPU crashes persist during PyTorch training + desktop use (crash frequency: once every 1-2 days)
**Related:** [001_2025-11-20-amd-7900xtx-freeze-mitigation.md](001_2025-11-20-amd-7900xtx-freeze-mitigation.md), [resources/amd_crash_context_121825.md](../resources/amd_crash_context_121825.md)

## Current System State

- **Kernel:** 6.8.0-90-generic (HWE upgrade from 6.2)
- **Previous params:** `iommu=pt amdgpu.gfxoff=0`
- **Crash pattern:** CPC page faults → MES timeout → MODE1 reset → Mutter fails to recover

## Crash Log Analysis

Latest crash showed:
```
[gfxhub] page fault (src_id:0 ring:153 vmid:0 pasid:0)
GCVM_L2_PROTECTION_FAULT_STATUS:0x00000B32
Faulty UTCL2 client ID: CPC (0x5)
WALKER_ERROR: 0x1, PERMISSION_FAULTS: 0x3, MAPPING_ERROR: 0x1
...
*ERROR* ring gfx_0.0.0 timeout
*ERROR* Process information: process brave pid 58691
GPU reset begin! → GPU reset(2) succeeded!
```

Key insight: **GPU actually recovers** via MODE1 reset. The freeze is caused by **Mutter (GNOME compositor) failing to handle GPU context invalidation** after reset.

## Root Cause

Multi-context MES (Micro Engine Scheduler) firmware bug on RDNA3:
- Desktop compositor (gnome-shell) + browser (Brave) + PyTorch competing for GPU
- MES cannot handle simultaneous graphics + compute context scheduling
- Known AMD issue, under investigation with no fix timeline

## Actions Taken Today

### 1. Disabled Brave Hardware Acceleration

Settings → System → "Use graphics acceleration when available" → OFF

Verified via `brave://gpu`:
```
Compositing: Software only. Hardware acceleration disabled
Rasterization: Software only. Hardware acceleration disabled
Video Decode: Software only. Hardware acceleration disabled
GPU0: SwiftShader Device (Subzero) *ACTIVE*
```

### 2. Added Kernel Parameters

Updated `/etc/default/grub`:
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash acpi_enforce_resources=lax iommu=pt amdgpu.gfxoff=0 amdgpu.tmz=0 amdgpu.runpm=0"
```

New parameters:
| Parameter | Purpose |
|-----------|---------|
| `amdgpu.tmz=0` | Disable Trusted Memory Zone - has known bugs on RDNA3 |
| `amdgpu.runpm=0` | Disable runtime power management - prevents power state transition crashes |

Ran `sudo update-grub`. Will take effect on next reboot.

### 3. Parameters Considered but Not Added

These are more relevant to display/idle issues, not multi-context scheduling:
- `amdgpu.sg_display=0` - scatter-gather display (not relevant)
- `amdgpu.dcdebugmask=0x10` - Display Core debug (not relevant)
- `pcie_aspm=off` - PCIe power states (not relevant to active-use crashes)

## Other Apps Still Using GPU

```
gnome-shell  - required for desktop
xdg-desktop  - portal, unavoidable
VS Code      - should disable HW acceleration
Anytype      - should disable HW acceleration
```

## Planned Next Steps

1. After next crash/reboot, verify new params are active:
   ```bash
   cat /proc/cmdline
   cat /sys/module/amdgpu/parameters/tmz
   cat /sys/module/amdgpu/parameters/runpm
   ```

2. Disable GPU acceleration in VS Code and Anytype

3. Upgrade to Ubuntu 24.04 during winter vacation (~1 week) for:
   - Newer kernel (6.8+ base with HWE path to 6.11+)
   - Newer linux-firmware package
   - Potentially better Mutter GPU reset handling

## Fundamental Limitation

There is no user-applicable firmware fix. MES firmware is:
- Closed-source AMD proprietary
- Shipped via `linux-firmware` package
- Only updatable to what AMD releases

Current `linux-firmware` version: `20220329.git681281e4-0ubuntu3.40` (Ubuntu 22.04 repos, files updated Sept 2024)

## Nuclear Options if Issues Persist

1. **ppfeaturemask:** `amdgpu.ppfeaturemask=0xfffd3fff` - disables additional power features
2. **Mainline kernel:** Install 6.11+ for latest RDNA3 MES fixes
3. **iGPU for display:** Route monitors through Ryzen iGPU, use 7900 XTX for compute only
4. **Different compositor:** Try KDE Plasma or X11 - may handle GPU resets better than Mutter/Wayland
