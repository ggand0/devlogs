# Environment Map Conversion Guide

**Date:** 2025-11-16
**Topic:** Converting HDR equirectangular panoramas to Bevy-compatible KTX2 cubemaps

## Background

Bevy 0.15 supports environment maps for image-based lighting (IBL) through the `Skybox` and `EnvironmentMapLight` components. However, these require cubemap textures in KTX2 format with specific encoding (rgb9e5 with zstd compression), not the common equirectangular HDR format.

## Problem

We had a bright HDR environment map (`autumn_field_puresky_2k.hdr`) that needed to be converted to proper cubemap format for use in Bevy. Initial attempts with online converters failed because they produced 2D textures instead of cubemaps, resulting in the error:

```
Texture binding 0 expects dimension = Cube, but given a view with dimension = D2
```

## Solution: bevy_skybox_cli

### Installation

1. **Clone the repository:**
```bash
cd /tmp
git clone https://github.com/bytestring-net/bevy_skybox_cli.git
```

2. **Build and install:**
```bash
cd bevy_skybox_cli
cargo install --path .
```

This installs the `bevy_skybox_cli` binary to `~/.cargo/bin/`.

### Tool Overview

`bevy_skybox_cli` is a specialized converter that:
- Takes equirectangular HDR files as input
- Generates THREE KTX2 cubemap files optimized for Bevy:
  - `skybox.ktx2` - High-quality cubemap for visible skybox
  - `diffuse_map.ktx2` - Pre-convolved diffuse irradiance map for ambient lighting
  - `specular_map.ktx2` - Pre-filtered specular map for reflections

### Usage

Convert an HDR file with a single command:

```bash
bevy_skybox_cli /path/to/your/environment.hdr
```

**Example:**
```bash
bevy_skybox_cli /home/gota/ggando/gamedev/bevy_3d_dodge/assets/textures/autumn_field_puresky_2k.hdr
```

**Output:**
The tool generates three files in the same directory as the input:
- `skybox.ktx2` (65 MB)
- `diffuse_map.ktx2` (49 MB)
- `specular_map.ktx2` (65 MB)

### Processing Details

The tool automatically:
1. Converts equirectangular panorama to cubemap (6 faces)
2. Generates mipmaps for proper filtering
3. Pre-convolves the diffuse map using spherical harmonics
4. Pre-filters the specular map for various roughness levels
5. Encodes using rgb9e5 HDR format
6. Compresses with zstd

## Integration in Bevy

Update your camera spawn code to use the generated files:

```rust
fn spawn_camera(mut commands: Commands, asset_server: Res<AssetServer>) {
    let skybox_handle = asset_server.load("textures/skybox.ktx2");
    let env_rotation = Quat::from_rotation_x(std::f32::consts::FRAC_PI_2);

    commands.spawn((
        Camera3d::default(),
        Camera {
            hdr: true,
            ..default()
        },
        Transform::from_xyz(0.0, -15.0, 10.0)
            .looking_at(Vec3::new(0.0, 0.0, 1.0), Vec3::Z),
        Skybox {
            image: skybox_handle.clone(),
            brightness: 500.0,
            rotation: env_rotation,
            ..default()
        },
        EnvironmentMapLight {
            diffuse_map: asset_server.load("textures/diffuse_map.ktx2"),
            specular_map: asset_server.load("textures/specular_map.ktx2"),
            intensity: 1500.0,
            rotation: env_rotation,
        },
        DebugCamera,
    ));
}
```

## Key Points

### Coordinate System
- Environment maps are typically Y-up (standard graphics convention)
- Our game uses Z-up (Isaac Sim convention)
- Apply rotation: `Quat::from_rotation_x(FRAC_PI_2)` to align properly

### Intensity Tuning
- `Skybox::brightness`: Controls visual brightness of the background (try 300-1000)
- `EnvironmentMapLight::intensity`: Controls lighting contribution to objects (try 1000-3000)
- Balance with existing artificial lights (point lights, directional lights)

### File Sizes
The generated KTX2 files are large (50-65 MB each) because they include:
- 6 cubemap faces at high resolution
- Complete mipmap chains
- Pre-computed lighting data

For production, consider:
- Lower resolution source HDRs (1k instead of 2k)
- Separate diffuse/specular quality levels
- Asset streaming for large environments

## Alternative Methods (Not Recommended)

### Manual Conversion with KTX-Software
Requires multiple steps and manual cubemap generation:
1. Convert equirectangular to 6 separate face images
2. Generate mipmaps manually
3. Convert to KTX2 with proper flags
4. Pre-convolve for diffuse/specular separately

**Verdict:** Too complex. Use `bevy_skybox_cli` instead.

### Online Converters
Most online converters produce 2D textures, not cubemaps, making them incompatible with Bevy's requirements.

**Verdict:** Unreliable. Use `bevy_skybox_cli` instead.

## Resources

- **bevy_skybox_cli**: https://github.com/bytestring-net/bevy_skybox_cli
- **HDR Sources**:
  - Poly Haven: https://polyhaven.com/hdris (free, high quality)
  - Poliigon: https://www.poliigon.com/hdrs/free
- **Bevy Documentation**: https://docs.rs/bevy/latest/bevy/pbr/struct.EnvironmentMapLight.html

## Results

After conversion and integration:
- **Bright outdoor skybox** visible as background
- **Realistic ambient lighting** from environment on all objects
- **Natural reflections** on glossy materials (player capsule, projectiles)
- **Better visual quality** compared to pure artificial lighting
- **5x intensity increase** (from 300 to 1500) to make environment lighting visible against strong point lights
