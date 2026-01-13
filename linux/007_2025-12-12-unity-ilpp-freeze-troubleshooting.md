# Unity Editor Freeze on Ubuntu 22.04 - ILPP Issue

**Date:** 2025-12-12
**System:** Ubuntu 22.04, Linux 6.2.0-39-generic
**Unity Version:** 2022.3.62f3
**Project:** WFXDebugProject/New Unity Project

## Problem

Unity Editor gets stuck at "Initial Database Asset Refresh" when opening projects on Ubuntu 22.04.

## Root Cause Analysis

Through verbose logging (`./Unity -logFile -`), identified the issue:

```
Connectivity with IL Post Processor runner cannot be established yet. Retrying.
Unhandled Exception: System.InvalidOperationException: Can't find file /tmp/ilpp.sock-3f627ccefaf5eab27728a83f4b098708
```

**The Issue:**
- IL Post Processor (ILPP) runner fails to create Unix socket file
- ILPP Trigger continuously retries connection, causing infinite loop
- Additional error: `inotify_init() failed: Too many open files (24)`

## Troubleshooting Steps Attempted

### 1. Cleared Unity Temporary Files
```bash
rm -f /tmp/ilpp.sock-* /tmp/Unity-*
```
**Result:** Did not resolve the issue.

### 2. Deleted Library Folder
```bash
rm -rf "/path/to/project/Library"
```
Forces Unity to do clean asset import.
**Result:** Did not resolve the issue.

### 3. Checked System Limits
```bash
cat /proc/sys/fs/inotify/max_user_watches  # 524288 (OK)
cat /proc/sys/fs/inotify/max_user_instances  # 128 (OK)
ulimit -n  # 524288 (OK)
```
**Result:** System limits were adequate.

### 4. Tested ILPP Runner Directly
```bash
~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Runner/Unity.ILPP.Runner --help
```
**Result:** Error: "Unix socket path must be absolute" - confirms ILPP runner is fundamentally broken.

### 5. Attempted Environment Variable Workaround
```bash
UNITY_MIXED_CALLSTACK=0 ./Unity -projectpath "..." -logFile ~/unity_no_ilpp.log
```
**Result:** Unity started but still encountered ILPP issues when launched via Hub.

## Solution

Disabled ILPP components by replacing executables with placeholder scripts:

```bash
# Backup originals
mkdir -p /home/gota/work/freelance/dataviz/unity_ilpp_backup
cp ~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Runner/Unity.ILPP.Runner \
   /home/gota/work/freelance/dataviz/unity_ilpp_backup/
cp ~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Trigger/Unity.ILPP.Trigger \
   /home/gota/work/freelance/dataviz/unity_ilpp_backup/

# Replace with placeholder scripts
# For Runner:
mv ~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Runner/Unity.ILPP.Runner \
   ~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Runner/Unity.ILPP.Runner.broken
echo -e '#!/bin/bash\nexit 0' > ~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Runner/Unity.ILPP.Runner
chmod +x ~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Runner/Unity.ILPP.Runner

# For Trigger:
mv ~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Trigger/Unity.ILPP.Trigger \
   ~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Trigger/Unity.ILPP.Trigger.broken
echo -e '#!/bin/bash\nexit 0' > ~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Trigger/Unity.ILPP.Trigger
chmod +x ~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Trigger/Unity.ILPP.Trigger
```

## Restoration Instructions

To restore ILPP functionality if needed:

```bash
# Restore Runner
cp /home/gota/work/freelance/dataviz/unity_ilpp_backup/Unity.ILPP.Runner \
   ~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Runner/Unity.ILPP.Runner
rm ~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Runner/Unity.ILPP.Runner.broken

# Restore Trigger
cp /home/gota/work/freelance/dataviz/unity_ilpp_backup/Unity.ILPP.Trigger \
   ~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Trigger/Unity.ILPP.Trigger
rm ~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/Unity.ILPP.Trigger/Unity.ILPP.Trigger.broken
```

## Technical Notes

- **ILPP (IL Post Processor):** Component for processing IL code during compilation, primarily for IL2CPP builds
- **Impact of Disabling:** Projects using Mono scripting backend (default) are unaffected. IL2CPP builds may fail.
- **Known Issue:** Unity 2022.3.x ILPP has documented Linux compatibility issues
- **Log Location:** `~/.config/unity3d/Editor.log`

## Status

**RESOLVED** ✓

Disabling both ILPP Runner and ILPP Trigger components successfully resolved the freeze issue. Unity Editor now opens projects without getting stuck at "Initial Database Asset Refresh".

## Outcome

- Unity 2022.3.62f3 successfully opens projects on Ubuntu 22.04
- "Initial Database Asset Refresh" completes without hanging
- Editor is fully functional for projects using Mono scripting backend
- Original ILPP executables backed up at: `/home/gota/work/freelance/dataviz/unity_ilpp_backup/`

## Limitations

- IL2CPP builds will not work with ILPP disabled
- For IL2CPP support, consider:
  - Upgrading to Unity 2023 LTS (improved Linux support)
  - Using a different Unity version without ILPP issues
  - Restoring ILPP components and building on a different platform

## References

- Unity Editor logs: `~/.config/unity3d/Editor.log`
- Unity process: `/home/gota/Unity/Hub/Editor/2022.3.62f3/Editor/Unity`
- ILPP tools location: `~/Unity/Hub/Editor/2022.3.62f3/Editor/Data/Tools/ilpp/`
