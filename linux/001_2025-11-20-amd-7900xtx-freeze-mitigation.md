# AMD 7900 XTX Freeze Issues - Mitigation Approach

**Date:** 2025-11-20
**Issue:** System freezes occurring with varying frequency (few days to weeks) on AMD 7900 XTX GPU

## Problem Analysis

Examined two freeze log instances revealing two distinct failure modes:

### Failure Type 1: IOMMU Page Faults
- `AMD-Vi: Event logged [IO_PAGE_FAULT domain=0x001e address=0x...]`
- `GCVM_L2_PROTECTION_FAULT_STATUS` errors
- Memory access violations through the IOMMU layer

### Failure Type 2: MES Communication Failures
- Repeated `MES failed to response msg=14` errors
- GPU firmware/driver communication breakdown
- Requires GPU reset to recover

## Applied Mitigation

### Primary Fix: Enable IOMMU Passthrough

Added kernel parameter to address IOMMU page faults:

```bash
iommu=pt
```

**What it does:** Sets IOMMU to passthrough mode - enabled but only translating addresses for devices that require it (like VMs), bypassing translation for direct GPU access.

**Configuration location:** `/etc/default/grub`
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash acpi_enforce_resources=lax iommu=pt"
```

## Additional Recommendations (if issues persist)

Try these **one at a time** if freezes continue:

1. **Update kernel and firmware** (current kernel 6.2 is old for RDNA3)
   ```bash
   sudo apt update
   sudo apt install linux-firmware amd64-microcode
   ```

2. **Disable runtime power management**
   ```bash
   amdgpu.runpm=0
   ```

3. **Increase GTT size for GFXOFF issues**
   ```bash
   amdgpu.gttsize=8192
   ```

4. **Disable GPU recovery if causing cascading failures**
   ```bash
   amdgpu.gpu_recovery=0
   ```

## What NOT to Do

**Avoid `amdgpu.ppfeaturemask=0xffffffff`** - This enables experimental PowerPlay features and can cause:
- Stuttering and artifacts
- Screen flicker
- Broken suspend/resume
- Additional instability

This parameter is for unlocking overclocking controls, not fixing stability issues.

## Monitoring

After reboot, verify the parameter is active:
```bash
cat /proc/cmdline | grep iommu
```

Monitor for freezes over the next 10-30 days to assess effectiveness.

## Log References

- [logs/amdgpu_fail0.log](../logs/amdgpu_fail0.log) - IOMMU page fault example
- [logs/amdgpu_fail1.log](../logs/amdgpu_fail1.log) - MES communication failure example
