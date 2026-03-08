# GTA San Andreas — Modding Setup on Linux (Ubuntu 24.04)

**Date:** 2026-03-09

## Context

Bought the original GTA San Andreas (classic, not Definitive Edition) as part of the GTA Trilogy from the Rockstar Store. Downloaded via Lutris using the Rockstar Games Launcher under Wine. Goal: mod the game for a proper modern experience, running via Proton through Steam.

## Game Files Location

- **Original backup:** `/data2/SteamLibraryFlatpak/gta_sa_org/Grand Theft Auto San Andreas_org`
- **Modded copy:** `/data2/SteamLibraryFlatpak/gta_sa_org/gta_sa`
- **Post-ModLoader backup:** `/data2/SteamLibraryFlatpak/gta_sa_org/gta_sa_aftermodloader`

## Step 1: Downgrade to v1.0

The Rockstar Store ships version v1.0.0.22 (5.4 MB exe) which is not mod-compatible. Replaced with the v1.0 US Hoodlum + LargeAddressAware exe.

**Source:** [Codeberg xls69/gta_sa_largeaddress](https://codeberg.org/xls69/gta_sa_largeaddress) — sourced from MixMods.com.br, also used by the official [Lutris installer](https://lutris.net/games/install/8721/view).

```bash
cd /data2/SteamLibraryFlatpak/gta_sa_org/gta_sa
mv gta_sa.exe gta_sa_rockstar.exe.bak
wget -O gta_sa.exe "https://codeberg.org/xls69/gta_sa_largeaddress/raw/branch/main/gta_sa.exe"
```

**Verification:**
- File size: 14,383,616 bytes (v1.0 US Hoodlum standard)
- SHA256: `f01a00ce950fa40ca1ed59df0e789848c6edcf6405456274965885d0929343ac`
- LargeAddressAware already applied (no separate 4GB patch needed)

Also deleted `MTLX.dll` (Rockstar Launcher DRM, not needed for standalone).

**Reference checksums for v1.0 US Hoodlum** (from [GTAG Forum](https://gtag.sannybuilder.com/forums/index.php?showtopic=171)):
- MD5: `170b3a9108687b26da2d8901c6948a18` or `4c6fa3b270e7028b31381761d08656d9` (unpatched)
- The Codeberg exe has a different hash due to the LAA patch.

## Step 2: Add to Steam as Non-Steam Game

1. Steam → Library → Add a Game → Add a Non-Steam Game → Browse → select `gta_sa.exe`
2. Right-click → Properties → Compatibility → Force Proton Experimental (or GE-Proton)
3. Flatpak Steam required filesystem access:
   ```bash
   flatpak override --user --filesystem=/data2 com.valvesoftware.Steam
   ```

Tested: game launches and runs (audio works, driving works).

## Step 3: Essential Mod Infrastructure

Installed in this order, following the [Proper Definitive Edition for SA guide](https://steamcommunity.com/sharedfiles/filedetails/?id=3543921363) and the [TJGM YouTube guide](https://www.youtube.com/watch?v=AYmrSyqyppc).

### 3a. Ultimate ASI Loader (v9.7.0)

Loads all `.asi` mod plugins. Must be named `vorbisFile.dll` to hook into the game's DLL loading.

```bash
mv vorbisFile.dll vorbisFile.dll.original.bak
wget -O vorbisFile.zip "https://github.com/ThirteenAG/Ultimate-ASI-Loader/releases/download/Win32-latest/vorbisFile-Win32.zip"
unzip vorbisFile.zip vorbisFile.dll && rm vorbisFile.zip
```

**Source:** [GitHub ThirteenAG/Ultimate-ASI-Loader](https://github.com/ThirteenAG/Ultimate-ASI-Loader)

### 3b. CLEO 5 (v5.3.0)

Script mod support — enables `.cs` CLEO scripts and `.cleo` plugins.

```bash
wget -O cleo5.zip "https://github.com/cleolibrary/CLEO5/releases/download/v5.3.0/SA.CLEO-v5.3.0.zip"
unzip -o cleo5.zip -x "cleo_readme/*" && rm cleo5.zip
```

Installs: `CLEO.asi`, `bass.dll`, and `cleo/` directory with plugins.

**Source:** [GitHub cleolibrary/CLEO5](https://github.com/cleolibrary/CLEO5)

### 3c. CLEO+ (v1.2.0)

Additional CLEO opcodes/commands used by many mods.

```bash
wget -O "cleo/cleo_plugins/CLEO+.cleo" "https://github.com/JuniorDjjr/CLEOPlus/releases/download/v1.2.0/CLEO%2B.cleo"
```

**Source:** [GitHub JuniorDjjr/CLEOPlus](https://github.com/JuniorDjjr/CLEOPlus)

**Note:** NewOpcodes (2012-2014, now dead) was skipped — CLEO 5 + CLEO+ covers its functionality. Only needed if a specific legacy mod requires it.

### 3d. ModLoader (v0.3.7)

Allows installing mods in subfolders under `modloader/` without replacing original game files. Each mod gets its own folder — disable by renaming/deleting the folder.

```bash
wget -O modloader.zip "https://github.com/thelink2012/modloader/releases/download/v0.3.7/modloader.zip"
unzip -o modloader.zip && rm modloader.zip
```

**Source:** [GitHub thelink2012/modloader](https://github.com/thelink2012/modloader)

**Tested:** Game launches with all four essentials loaded.

## Step 4: Fixes and Adjustments

### 4a. SilentPatch SA (BUILD 33.1)

Critical bug fixes for v1.0 (which has notorious bugs). Installed into `modloader/SilentPatch/`.

```bash
wget -O SilentPatchSA.zip "https://github.com/CookiePLMonster/SilentPatch/releases/download/1.1-BUILD33.1-SA/SilentPatchSA.zip"
mkdir -p modloader/SilentPatch
unzip -o SilentPatchSA.zip -d modloader/SilentPatch/ -x ReadMe.txt && rm SilentPatchSA.zip
```

**Source:** [GitHub CookiePLMonster/SilentPatch](https://github.com/CookiePLMonster/SilentPatch)

### 4b. Open Limit Adjuster (v1.5.9)

Removes hardcoded engine limits (object counts, model counts, etc.). Installed into `modloader/OpenLimitAdjuster/`.

```bash
wget -O ola.zip "https://github.com/GTAmodding/III.VC.SA.LimitAdjuster/releases/download/v1.5.9/III.VC.SA.LimitAdjuster.zip"
mkdir -p modloader/OpenLimitAdjuster
unzip -o ola.zip -d modloader/OpenLimitAdjuster/ && rm ola.zip
```

**Config change:** Set `MemoryAvailable = 1024` (1GB) in `[SALIMITS]` section of `III.VC.SA.LimitAdjuster.ini` (was `30%` = 30% of system RAM). Per the YT guide's updated recommendation.

**Source:** [GitHub GTAmodding/III.VC.SA.LimitAdjuster](https://github.com/GTAmodding/III.VC.SA.LimitAdjuster)

### 4c. Widescreen Fix (Unofficial 2024 Version)

Fixes aspect ratio, HUD, and FOV for widescreen/ultrawide displays. Installed into `modloader/WidescreenFix/`.

Downloaded manually from [MixMods Widescreen Fix page](https://www.mixmods.com.br/2021/05/widescreen-fix/) (ShareMods link, requires browser). Extracted `GTASA.WidescreenFix.asi` and `GTASA.WidescreenFix.ini` from the `SA_-_Widescreen_Fix.7z` archive.

The YT guide author updated his recommendation from the older GitHub version to this unofficial 2024 version, which includes additional fixes and is compatible with older mods.

## Final Directory Structure

```
gta_sa/
├── gta_sa.exe                    # v1.0 Hoodlum + LargeAddressAware
├── gta_sa_rockstar.exe.bak       # Original Rockstar Store exe (backup)
├── vorbisFile.dll                # Ultimate ASI Loader (renamed)
├── vorbisFile.dll.original.bak   # Original vorbisFile.dll (backup)
├── CLEO.asi                      # CLEO 5
├── bass.dll                      # CLEO dependency
├── modloader.asi                 # ModLoader
├── cleo/
│   ├── .cleo_config.ini
│   ├── .config/
│   ├── cleo_plugins/
│   │   ├── CLEO+.cleo
│   │   ├── SA.Audio.cleo
│   │   ├── SA.DebugUtils.cleo
│   │   ├── SA.FileSystemOperations.cleo
│   │   ├── SA.GameEntities.cleo
│   │   ├── SA.IniFiles.cleo
│   │   ├── SA.Input.cleo
│   │   ├── SA.Math.cleo
│   │   ├── SA.MemoryOperations.cleo
│   │   └── SA.Text.cleo
│   ├── cleo_modules/
│   ├── cleo_saves/
│   └── cleo_text/
└── modloader/
    ├── SilentPatch/
    │   ├── SilentPatchSA.asi
    │   └── SilentPatchSA.ini
    ├── OpenLimitAdjuster/
    │   ├── III.VC.SA.LimitAdjuster.asi
    │   └── III.VC.SA.LimitAdjuster.ini  (1024MB streaming)
    └── WidescreenFix/
        ├── GTASA.WidescreenFix.asi
        └── GTASA.WidescreenFix.ini
```

## Key References

- [Proper Definitive Edition for SA guide](https://steamcommunity.com/sharedfiles/filedetails/?id=3543921363) — curated modlist with install order
- [TJGM YouTube guide (Episode 2)](https://www.youtube.com/watch?v=AYmrSyqyppc) — video walkthrough of essentials
- [Linux modding guide](https://steamcommunity.com/sharedfiles/filedetails/?id=2553842834) — Linux-specific setup
- [Downgrade every version guide](https://steamcommunity.com/sharedfiles/filedetails/?id=3036381821) — exe downgrade methods
- [gta_sa.exe version checksums](https://gtag.sannybuilder.com/forums/index.php?showtopic=171) — MD5 reference

## Notes

- **FusionFix** is for GTA IV / Definitive Edition, NOT original SA. SilentPatch is the equivalent for original SA.
- **NewOpcodes** was skipped — CLEO 5 + CLEO+ covers its functionality.
- **Proton version matters** — if mods cause issues, try switching between Proton Experimental and GE-Proton.
- To add more mods: create a subfolder in `modloader/`, drop files in. To disable: rename or delete the folder.

## Next Steps

- Test-launch with all mods loaded
- Episode 3+ of the YT guide: visual/graphics mods (Project2DFX, SkyGFX, etc.)
- Follow the curated modlist for Fixes → Interface → Map and Graphics → QoL
