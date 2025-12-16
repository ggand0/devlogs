# RT-DETR Setup and Triton Compatibility Resolution

**Date:** 091225 
**Hardware:** AMD Radeon RX 7900 XTX (24GB VRAM)  
**Goal:** Set up explore-cv project with RT-DETR model for video inference

## Initial Setup Success

### Project Structure Created ✅
- Modern uv-based Python package management
- Extensible detector architecture with base classes
- RT-DETR implementation using transformers/torch.hub
- Video processing pipeline with OpenCV
- CLI interface for easy video inference

### Components Implemented
- `BaseDetector` abstract class for consistent model interfaces
- `RTDETRDetector` with proper preprocessing/postprocessing
- `VideoReader`/`VideoWriter` utilities with progress tracking
- Command-line interface with configurable parameters

## The Critical Compatibility Issue

### Error Encountered
```bash
AttributeError: module 'triton' has no attribute 'language'
```

**Error Location:** `torch/_dynamo/utils.py:1066`
```python
common_constant_types.add(triton.language.dtype)  # <- Failed here
```

### Root Cause Analysis

**Initial Hypothesis:** Transformers version incompatibility
- ❌ **Reality:** Transformers 4.55.4 was compatible (requires PyTorch >2.2 ✅)

**Actual Root Cause:** ROCm-PyTorch-Triton version mismatch
- **Issue:** `pytorch-triton-rocm 3.0.0` (July 2024) lacked `triton.language` module
- **Context:** PyTorch 2.4.1+rocm6.0 uses `torch._dynamo` which expects modern Triton API
- **Gap:** ~1 year between our ROCm version and latest releases

## Version Timeline & Attempts

### Initial Setup (ROCm 6.0.0)
```toml
# What we had
torch = "2.4.1+rocm6.0"
torchvision = "0.19.1+rocm6.0" 
pytorch-triton-rocm = "3.0.0"  # <- Missing triton.language
```

**Result:** ❌ Import failures on basic torchvision operations

### Failed Workarounds Attempted
1. **Transformers downgrade** - Wouldn't help since issue was in PyTorch/TorchVision
2. **Direct triton fixes** - PyTorch 2.4.1+rocm6.0 locked to triton 3.0.0
3. **Alternative imports** - Core API missing in ROCm triton build

### Research Findings
- **GitHub Issues:** pytorch/pytorch #143718 - ROCm triton issues marked "STILL NOT FIXED"
- **Version Gap:** pytorch-triton-rocm 3.0.0 (July 2024) vs triton 3.4.0 (July 2025)
- **Upstream Status:** triton-lang/triton main branch ✅ HAS `triton.language` module

## The Solution: ROCm Upgrade

### Version Discovery
- **Current ROCm:** 6.0.0 (found via `rocm-smi --version`)
- **Latest Stable:** ROCm 6.4.3 (August 7, 2025)
- **Gap:** ~1.5 years of fixes and compatibility improvements

### ROCm 6.4.3 Upgrade Process
```bash
# Repository setup
sudo mkdir --parents --mode=0755 /etc/apt/keyrings
wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | \
    gpg --dearmor | sudo tee /etc/apt/keyrings/rocm.gpg > /dev/null

# Add ROCm 6.4.3 repository (Ubuntu 22.04)
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocm/apt/6.4.3 jammy main" | \
    sudo tee /etc/apt/sources.list.d/rocm.list

echo -e 'Package: *\nPin: release o=repo.radeon.com\nPin-Priority: 600' | \
    sudo tee /etc/apt/preferences.d/rocm-pin-600

# Upgrade ROCm packages
sudo apt update
sudo apt upgrade rocm rocm-core rocm-dev rocm-libs
sudo reboot
```

## Final Working Configuration

### Package Versions (ROCm 6.4.3)
```toml
# Working setup
torch = "2.6.0+rocm6.4.3"
torchvision = "0.21.0+rocm6.4.3"
torchaudio = "2.6.0+rocm6.4.3"
pytorch-triton-rocm = "3.2.0+rocm6.4.3"  # <- Has triton.language ✅
transformers = "4.55.4"  # Works perfectly now
```

### uv Configuration
```toml
[tool.uv.sources]
torch = [{ index = "pytorch-rocm" }]
torchvision = [{ index = "pytorch-rocm" }]
torchaudio = [{ index = "pytorch-rocm" }]
pytorch-triton-rocm = [{ index = "pytorch-rocm" }]

[[tool.uv.index]]
name = "pytorch-rocm"
url = "https://download.pytorch.org/whl/rocm6.4"
explicit = true
```

## Key Learnings

### The Real Issue
- **Not a transformers compatibility problem**
- **Not a PyTorch version mismatch**  
- **ROCm-specific triton implementation** was outdated and missing critical APIs

### Critical Insight
The error `triton.language` attribute missing was specific to **ROCm's triton fork**, not the main triton repository. The main triton-lang/triton had the language module, but ROCm's pytorch-triton-rocm 3.0.0 didn't.

### Version Dependencies
- ROCm releases drive PyTorch+triton compatibility
- Major version gaps (6.0 → 6.4.3) contain critical fixes
- Always check ROCm release dates vs package install dates

## Current Status ✅

### Working Components
- **GPU Detection:** ✅ AMD RX 7900 XTX properly detected
- **RT-DETR Model:** ✅ Loading and running inference  
- **Video Processing:** ✅ Processing frames with progress tracking
- **CLI Interface:** ✅ `uv run explore-cv rtdetr video.mp4` working

### Performance Notes
- First inference: ~6.3s (model loading + GPU warmup)
- Subsequent inference: Much faster GPU-accelerated processing
- Video processing: Real-time capable on 4K footage

## Next Steps

1. **Expand Model Support:** Add RF-DETR, YOLOv11, Florence-2
2. **Performance Optimization:** Batch processing, model optimization
3. **Results Analysis:** Failure case analysis, accuracy comparisons
4. **Documentation:** Technical blog post on findings

## Resources Used

- **ROCm Installation Guide:** [Ubuntu Native Installation](https://rocm.docs.amd.com/projects/install-on-linux/en/latest/install/install-methods/package-manager/package-manager-ubuntu.html)
- **GitHub Issues:** pytorch/pytorch #143718, astral-sh/uv #10712
- **ROCm Releases:** [ROCm 6.4.3 Release Notes](https://rocm.docs.amd.com/en/latest/about/release-notes.html)
- **Triton Repository:** [triton-lang/triton](https://github.com/triton-lang/triton) (confirmed language module presence)

---

**Conclusion:** The issue was a classic case of dependency version skew in the ROCm ecosystem. The ~1.5 year gap between our installed ROCm 6.0.0 and the latest ROCm 6.4.3 contained critical triton compatibility fixes. The solution wasn't downgrading or workarounds, but upgrading to get the properly implemented triton.language module in pytorch-triton-rocm 3.2.0.