# Increased Swap to 32GB

**Date:** 2025-12-23

## Problem

System running critically low on memory with swap exhausted. Training script (`train_sac.py`) consuming ~32GB RAM, combined with 72 Claude Code extension processes across 8 concurrent projects.

**Before:**
- RAM: 62GB total, 58GB used
- Swap: 15GB total, 15GB used (0B free)

## Root Cause

Running multiple VSCode windows with Claude Code extension spawns many processes (~72), each using 250-300MB. Combined with ML training workloads, the existing 15GB swap was insufficient.

## Solution

Added 16GB swap file to supplement existing swap:

```bash
sudo fallocate -l 16G /swapfile2
sudo chmod 600 /swapfile2
sudo mkswap /swapfile2
sudo swapon /swapfile2

# Made permanent
echo '/swapfile2 none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Result

```
NAME           TYPE      SIZE USED PRIO
/swapfile      file        8G   8G   -2
/dev/nvme0n1p2 partition 7.5G 7.4G   -3
/swapfile2     file       16G   0B   -4

Swap: 31Gi total, 15Gi used, 16Gi free
```

Total swap now ~32GB with 16GB headroom.
