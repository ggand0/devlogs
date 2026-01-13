# AMD 7900 XTX Driver Stack Analysis

**Date:** 2025-12-09
**Trigger:** UE 5.7 warning about outdated drivers

## Current Driver Stack

| Component | Installed | Recommended |
|-----------|-----------|-------------|
| Kernel | 6.2.0 | 6.8+ |
| Mesa | 23.3.0-devel | 24.2+ |
| amdgpu-dkms | 6.3.6 | Current |
| AMDGPU-PRO | 23.30 | Remove or update |
| ROCm | 6.4.3 | Current |

## Component Relationships

```
┌─────────────────────────────────────────────┐
│  Applications (UE5, PyTorch, Games)         │
├─────────────────┬───────────────────────────┤
│  Mesa 23.3      │  ROCm 6.4.3               │
│  (Graphics)     │  (Compute/ML)             │
├─────────────────┴───────────────────────────┤
│  amdgpu-dkms 6.3.6 (Kernel Driver)          │
├─────────────────────────────────────────────┤
│  Linux Kernel 6.2                           │
└─────────────────────────────────────────────┘
```

- **amdgpu-dkms**: Kernel module that communicates with GPU hardware
- **Mesa**: Userspace driver for OpenGL/Vulkan (graphics rendering)
- **AMDGPU-PRO**: AMD proprietary userspace (OpenCL, AMF encoding)
- **ROCm**: Compute stack for ML/HIP workloads

## Issues Identified

1. **UE5 warning**: Wants Mesa/Vulkan 24.2+, we have 23.3
2. **Version mismatch**: ROCm 6.4.3 is current but Mesa/kernel are old
3. **Mixed driver state**: Both open-source and proprietary components installed
4. **GPU freezes**: Likely caused by kernel 6.2 + driver mismatches

## Relation to GPU Freezes

The MES firmware timeouts and IOMMU page faults documented in [002_2025-11-20-amd-7900xtx-freeze-mitigation.md](./2025-11-20-amd-7900xtx-freeze-mitigation.md) are likely exacerbated by:

- Kernel 6.2 having immature RDNA3 MES support
- Mismatched driver component versions
- Old Mesa not handling memory management optimally

## Resolution Plan (EOY Ubuntu 24.04 Upgrade)

### Pre-upgrade
```bash
# Remove old AMDGPU-PRO stack
amdgpu-install --uninstall

# Document ROCm setup for reinstall
```

### Post-upgrade (Ubuntu 24.04)
- Kernel 6.8+ (automatic)
- Mesa 24.x (automatic)
- Reinstall ROCm 6.4.3+
- Keep `iommu=pt` in GRUB config

### Expected Outcome
- All components in sync
- UE5 driver warning resolved
- GPU stability improved
