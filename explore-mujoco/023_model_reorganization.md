# Devlog 023: Model Reorganization

## Summary

Moved robot models and scene files from the SO-ARM100 git submodule to a local `models/so101/` directory. Mesh assets are now tracked via Git LFS instead of being pulled from a submodule.

## Motivation

The SO-ARM100 submodule caused several issues:

1. **MuJoCo path resolution** - Scene files using `<include>` had to live in the same directory as robot XMLs due to relative mesh path resolution
2. **Commit complexity** - Changes to robot XMLs required commits in both the submodule and main repo
3. **Clone overhead** - Full submodule was 251MB but only 16MB was needed for simulation

## Changes

### Before
```
SO-ARM100/                    # Git submodule (251MB)
└── Simulation/SO101/
    ├── so101_new_calib.xml
    ├── lift_cube_scene.xml
    └── assets/
```

### After
```
models/so101/                 # Local directory (16MB)
├── so101_new_calib.xml       # Robot models
├── so101_ik_grasp.xml
├── so101_horizontal_grasp.xml
├── lift_cube.xml             # Scene files
├── lift_cube_ik_grasp.xml
├── lift_cube_horizontal.xml
└── assets/                   # STL meshes (Git LFS)
```

## Implementation

1. **Copy models and assets**
   ```bash
   mkdir -p models/so101
   cp SO-ARM100/Simulation/SO101/so101_*.xml models/so101/
   cp -r SO-ARM100/Simulation/SO101/assets models/so101/
   ```

2. **Setup Git LFS for mesh files**
   ```bash
   git lfs install
   git lfs track "*.stl" "*.part"
   ```

3. **Update scene file includes**
   Scene files now use local includes since they're in the same directory:
   ```xml
   <include file="so101_new_calib.xml" />
   ```

4. **Update test script paths**
   ```python
   # Before
   scene_path = Path("SO-ARM100/Simulation/SO101/lift_cube_scene.xml")

   # After
   scene_path = Path("models/so101/lift_cube.xml")
   ```

5. **Remove submodule**
   ```bash
   git rm SO-ARM100
   rm -rf .git/modules/SO-ARM100
   ```

## File Naming

Scene files were renamed for consistency:
- `lift_cube_scene.xml` → `lift_cube.xml`
- `lift_cube_scene_ik_grasp.xml` → `lift_cube_ik_grasp.xml`
- `lift_cube_scene_horizontal.xml` → `lift_cube_horizontal.xml`

## Installation

Assets are pulled automatically via Git LFS on clone. Alternative manual method:
```bash
# From SO-ARM100 repo
cp -r SO-ARM100/Simulation/SO101/assets models/so101/
```

## Benefits

- Simpler project structure
- Single repo for all changes
- Faster clones (16MB vs 251MB)
- No submodule sync issues
- Scene files can include robot models with simple relative paths
