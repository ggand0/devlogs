# AMD GPU Periodic Lag Troubleshooting (wgpu/Bevy apps)

**Date:** 2025-12-20
**GPU:** Radeon RX 7900 XTX (Navi 31)
**OS:** Ubuntu 22.04, X11/GNOME, Kernel 6.8.0-90
**Issue:** Periodic stuttering every 10-30 seconds in wgpu/Vulkan apps

## What We Tried (all failed to fix)

### 1. Kernel Parameters (already set)
- `amdgpu.gfxoff=0` - Disables GFXOFF power saving
- `amdgpu.runpm=0` - Disables runtime power management

### 2. Power Profile Mode
```bash
echo manual | sudo tee /sys/class/drm/card1/device/power_dpm_force_performance_level
echo 5 | sudo tee /sys/class/drm/card1/device/pp_power_profile_mode
```
Set to COMPUTE profile (5) - **did not fix**

### 3. Memory Clock
Checked `/sys/class/drm/card1/device/pp_dpm_mclk` - already at max (1249MHz)

### 4. fTPM
Checked `dmesg | grep -i tpm` - no output, likely not the cause

### 5. Blueman dbus spam
Killed blueman-applet (was spamming dbus errors) - **did not fix**

### 6. Compositor
Using X11 + GNOME (Mutter). App doesn't have fullscreen mode so can't test unredirected mode.

### 7. Switched to RADV (Mesa open-source driver)
```bash
AMD_VULKAN_ICD=RADV cargo run
```
Was using AMDVLK (proprietary). Switched to RADV - **did not fix**

### 8. Present Mode
Changed Bevy to use `PresentMode::AutoNoVsync` instead of default Fifo - **did not fix**

### 9. GNOME Night Light
Disabled via:
```bash
gsettings set org.gnome.settings-daemon.plugins.color night-light-enabled false
```
Was enabled and can cause gamma ramp stutters - **did not fix**

### 10. PCIe/ASPM
Checked settings - ASPM already disabled (`aspm=0`), PCIe running at full x16 16GT/s

### 11. Mesa Version
Mesa 23.2.1 - tried Kisak PPA but no newer version available for 22.04

## Key Finding: Outdated Firmware

```
linux-firmware: 20220329 (March 2022)
GPU release: December 2022
```

**The firmware predates the GPU by 9 months.** This is likely the root cause.

## Next Step: Update Firmware Manually

```bash
# Backup current firmware
sudo cp -r /lib/firmware/amdgpu /lib/firmware/amdgpu.bak

# Clone latest firmware
cd /tmp
git clone --depth 1 https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git

# Copy AMD firmware
sudo cp -r linux-firmware/amdgpu/* /lib/firmware/amdgpu/

# Rebuild initramfs
sudo update-initramfs -u

# Reboot
sudo reboot
```

## Fallback Options (if firmware doesn't fix)

### 1. ppfeaturemask (safe to try now) - PARTIALLY WORKED

Disables PP_GFXOFF_MASK and PP_STUTTER_MODE. Side effect: higher idle power draw.

**Result (2025-12-22):** Stutter duration reduced by ~50%, but still present.

**To enable:**
```bash
sudo nano /etc/default/grub
# Add to GRUB_CMDLINE_LINUX_DEFAULT:
# amdgpu.ppfeaturemask=0xfffd7fff

sudo update-grub
sudo reboot
```

**To revert:**
```bash
# Edit /etc/default/grub, remove the parameter
sudo update-grub
sudo reboot
```

**If display breaks on boot:**
1. Hold Shift during boot to get GRUB menu
2. Press `e` to edit
3. Remove `amdgpu.ppfeaturemask=0xfffd7fff` from the linux line
4. Press Ctrl+X to boot

### 2. Additional kernel parameters
```
amdgpu.aspm=0 amdgpu.bapm=0 amdgpu.vm_update_mode=0 amdgpu.dcdebugmask=0x10
```

## References
- https://community.amd.com/t5/pc-graphics/linux-rx-7900-xtx-performance-drops-after-a-few-minutes/m-p/611199
- https://bbs.archlinux.org/viewtopic.php?id=273933
- https://wiki.archlinux.org/title/AMDGPU
- https://www.phoronix.com/news/AMD-RDNA3-Plus-Firmware-Files
- https://docs.kernel.org/gpu/amdgpu/thermal.html
