# ROCm to CUDA Migration Plan

**Date:** 2026-02-06
**Hardware:** AMD RX 7900 XTX → NVIDIA RTX 3090
**OS:** Ubuntu 24.04 LTS, kernel 6.8.0-90-generic

## Context

Replacing the 7900 XTX with an RTX 3090. This requires full removal of ROCm and AMD GPU
driver stack, followed by NVIDIA driver + CUDA toolkit installation, and updating all ML
project configurations from ROCm PyTorch wheels to CUDA wheels.

Lessons from previous AMD setup work (see blog posts rocm0, rocm1, 008-24-04-upgrade):
- Stale config files (`ld.so.conf.d`, `xorg.conf`, symlinks) caused cascading GNOME crashes
  after the 24.04 upgrade. Must do a thorough cleanup this time.
- When originally switching to AMD, `sudo apt purge nvidia*` was done before the physical
  swap. Mirror that approach in reverse.
- ROCm consumed ~23.8 GB. Reclaim that space.

---

## Phase 1: Pre-Swap (while still on AMD GPU)

### 1.1 Remove ROCm packages

```bash
sudo apt purge amdgpu-dkms rocm
sudo apt autoremove
sudo apt clean
```

### 1.2 Remove ROCm apt repo and keyring

```bash
sudo rm /etc/apt/sources.list.d/rocm.list
sudo rm /etc/apt/keyrings/rocm.gpg
sudo apt update
```

### 1.3 Remove ROCm config files

```bash
# linker config
sudo rm /etc/ld.so.conf.d/rocm.conf
sudo rm -f /etc/ld.so.conf.d/15-amdgpu-pro.conf.bak
sudo rm -f /etc/ld.so.conf.d/20-amdgpu.conf.bak
sudo rm -f /etc/ld.so.conf.d/10-rocm-opencl.conf.bak

# udev rules
sudo rm /etc/udev/rules.d/70-amdgpu.rules

# rebuild linker cache
sudo ldconfig
```

### 1.4 Check for stale xorg.conf

```bash
ls -la /etc/X11/xorg.conf
# If it exists and contains AMD/amdgpu references, remove it:
# sudo rm /etc/X11/xorg.conf
```

This file caused X11 segfaults during the 24.04 upgrade. NVIDIA may write its own; a stale
AMD one would conflict.

### 1.5 Clean up GRUB

Edit `/etc/default/grub`:

```
# FROM:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash acpi_enforce_resources=lax iommu=pt amdgpu.gfxoff=0 amdgpu.tmz=0 amdgpu.runpm=0"

# TO:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```

All the extra parameters were AMD-specific workarounds (RDNA3 crash mitigations, IOMMU page
fault fixes). None are needed for NVIDIA.

```bash
sudo update-grub
```

### 1.6 Clean up ~/.bashrc

Remove these lines:

```bash
# ROCm paths and aliases
export PATH=$PATH:/opt/rocm-6.0/bin
export LD_LIBRARY_PATH=/opt/rocm/lib
export HIP_VISIBLE_DEVICES=0
export HSA_OVERRIDE_GFX_VERSION=11.0.0
alias rocm-smi='/opt/rocm/bin/rocm-smi'
alias watch-rocm='watch -n 1 /opt/rocm/bin/rocm-smi'

# Obsolete CUDA 11.2 paths (from before the AMD era)
export PATH=/usr/local/cuda-11.2/bin
export LD_LIBRARY_PATH=/usr/local/cuda-11.2/lib64
```

### 1.7 Remove leftover ROCm directories

```bash
sudo rm -rf /opt/rocm /opt/rocm-*
```

### 1.8 Verify cleanup

```bash
# Should return nothing:
dpkg -l | grep -i rocm
dpkg -l | grep -i amdgpu
ls /opt/rocm* 2>/dev/null
cat /etc/ld.so.conf.d/*rocm* 2>/dev/null
cat /etc/ld.so.conf.d/*amdgpu* 2>/dev/null
```

### 1.9 Shut down

```bash
sudo shutdown now
```

---

## Phase 2: Physical GPU Swap

1. Unplug power cable from PSU
2. Remove 7900 XTX, install RTX 3090
3. Connect display cable to RTX 3090
4. Boot — system will come up with the `nouveau` open-source driver (basic display, low res)

---

## Phase 3: Post-Swap — Install NVIDIA Driver + CUDA

### 3.1 Install CUDA toolkit (includes driver)

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
sudo apt install cuda
```

This installs the latest compatible NVIDIA driver + full CUDA toolkit. The RTX 3090 (Ampere,
compute capability 8.6) is well supported.

### 3.2 Reboot and verify

```bash
sudo reboot
```

After reboot:

```bash
nvidia-smi                     # Should show RTX 3090, 24GB VRAM
nvcc --version                 # CUDA compiler version
cat /proc/driver/nvidia/version
```

### 3.3 Add CUDA to ~/.bashrc

```bash
export PATH=$PATH:/usr/local/cuda/bin
export LD_LIBRARY_PATH=/usr/local/cuda/lib64
alias watch-gpu='watch -n 1 nvidia-smi'
```

### 3.4 Wayland (optional)

`/etc/gdm3/custom.conf` currently has `WaylandEnable=false`. NVIDIA now supports Wayland
well on recent drivers. Can re-enable if desired, or keep X11.

---

## Phase 4: Update ML Projects

### 4.1 Update pyproject.toml in all projects

**Projects to update:**
- `/home/gota/ggando/ml/lerobot/pyproject.toml`
- `/home/gota/ggando/ml/lerobot-org/pyproject.toml`
- `/home/gota/ggando/ml/pick-101/pyproject.toml`
- `/home/gota/ggando/ml/salt/pyproject.toml`
- `/home/gota/ggando/ml/explore-cv/pyproject.toml`
- `/home/gota/ggando/ml/pick-101-genesis/pyproject.toml`

**Changes for each:**

1. Remove `pytorch-triton-rocm` from dependencies (CUDA PyTorch bundles triton natively)

2. Change the uv index:
```toml
# FROM:
[[tool.uv.index]]
name = "pytorch-rocm"
url = "https://download.pytorch.org/whl/rocm6.4"
explicit = true

# TO:
[[tool.uv.index]]
name = "pytorch-cuda"
url = "https://download.pytorch.org/whl/cu124"
explicit = true
```

3. Update `[tool.uv.sources]` — change `pytorch-rocm` references to `pytorch-cuda` and
   remove any `pytorch-triton-rocm` source entries.

### 4.2 Recreate virtual environments

For each project:

```bash
rm -rf .venv uv.lock
uv sync
```

### 4.3 Verify PyTorch CUDA

```python
import torch
print(torch.cuda.is_available())        # True
print(torch.cuda.get_device_name(0))    # NVIDIA GeForce RTX 3090
print(torch.version.cuda)               # e.g. 12.4
```

---

## Summary

| Removed (ROCm)                         | Added (CUDA)                          |
|-----------------------------------------|---------------------------------------|
| `amdgpu-dkms`, `rocm` packages         | `cuda` package (driver + toolkit)     |
| `/etc/apt/sources.list.d/rocm.list`     | NVIDIA CUDA apt repo                  |
| `/etc/ld.so.conf.d/rocm.conf`           | (handled by cuda package)             |
| `/etc/udev/rules.d/70-amdgpu.rules`    | (not needed for NVIDIA)               |
| GRUB `amdgpu.*` params                 | (none needed)                         |
| bashrc ROCm exports                    | bashrc CUDA exports                   |
| `pytorch-triton-rocm` + rocm6.4 wheels | cu124 wheels (triton bundled)         |
| ~23.8 GB in /opt/rocm                  | ~5-8 GB CUDA toolkit                  |
