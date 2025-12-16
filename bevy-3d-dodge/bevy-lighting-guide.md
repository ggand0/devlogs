# Bevy 3D Lighting Best Practices

Guide to making Bevy 3D scenes look modern and realistic instead of like old 3D games.

## Core Improvements (Biggest Impact)

### 1. Environment Maps (HDR)
Essential for realistic PBR rendering - provides ambient lighting reflecting the environment.

```rust
commands.insert_resource(EnvironmentMapLight {
    diffuse_map: asset_server.load("environment_maps/pisa_diffuse_rgb9e5_zstd.ktx2"),
    specular_map: asset_server.load("environment_maps/pisa_specular_rgb9e5_zstd.ktx2"),
    intensity: 1000.0,
});
```

### 2. Material Properties (PBR)
Proper `metallic` and `perceptual_roughness` values are crucial:

```rust
// Metals (shiny to brushed)
StandardMaterial {
    metallic: 1.0,
    perceptual_roughness: 0.2,  // shiny
    // perceptual_roughness: 0.5,  // brushed
    ..default()
}

// Non-metals (plastics, wood, etc.)
StandardMaterial {
    metallic: 0.0,
    perceptual_roughness: 0.3,  // smooth plastic
    // perceptual_roughness: 0.7,  // rough surface
    ..default()
}

// Very rough surfaces (concrete, rough stone)
StandardMaterial {
    metallic: 0.0,
    perceptual_roughness: 0.8,
    ..default()
}
```

### 3. Physical Camera Exposure
Realistic exposure settings using aperture, shutter speed, and ISO:

```rust
Camera3d {
    exposure: Exposure::from_physical_camera(PhysicalCameraParameters {
        aperture_f_stops: 4.0,
        shutter_speed_s: 1.0 / 125.0,
        sensitivity_iso: 100.0,
        sensor_height: 0.01866,
    }),
    ..default()
}
```

## Secondary Improvements

### 4. Shadow Configuration
Proper shadow settings for directional lights:

```rust
DirectionalLight {
    illuminance: 10_000.0,
    shadows_enabled: true,
    shadow_depth_bias: 0.02,
    shadow_normal_bias: 0.6,
    ..default()
}
```

For cascaded shadows (outdoor scenes):
```rust
use bevy::pbr::CascadeShadowConfigBuilder;

DirectionalLight {
    shadows_enabled: true,
    shadow_cascade: CascadeShadowConfigBuilder {
        first_cascade_far_bound: 4.0,
        maximum_distance: 10.0,
        ..default()
    }.build(),
    ..default()
}
```

### 5. Normal Maps
Add surface detail without extra geometry - **requires tangents**:

```rust
// Generate tangents for normal mapping
let mesh_handle = meshes.add(
    mesh.with_generated_tangents().unwrap()
);

// Material with normal map
StandardMaterial {
    normal_map_texture: Some(asset_server.load("textures/normal.png")),
    // flip_normal_map_y: true,  // if authored for DirectX
    ..default()
}
```

### 6. Ambient Light
Set appropriate ambient lighting (measured in candela per meter square):

```rust
commands.insert_resource(AmbientLight {
    color: Color::WHITE,
    brightness: 200.0,  // adjust to taste
});
```

### 7. Point/Spot Lights
Configure intensity properly (measured in lumens):

```rust
// Point light
PointLight {
    intensity: 100_000.0,
    shadows_enabled: true,
    ..default()
}

// Spot light
SpotLight {
    intensity: 100_000.0,
    shadows_enabled: true,
    inner_angle: 0.6,
    outer_angle: 0.8,
    ..default()
}
```

## Advanced Techniques

### Light Textures (Bevy 0.16+)
Modulate light intensity with textures (for effects like window shadows, caustics):

```rust
// Add to a PointLight, SpotLight, or DirectionalLight
PointLight {
    intensity: 100_000.0,
    ..default()
}
// Then add the light texture component
PointLightTexture {
    texture: asset_server.load("textures/light_pattern.png"),
}
```

### Raytraced Lighting (Bevy 0.17+, Experimental)
Cutting-edge physically realistic lighting (requires compatible GPU):
- **Solari**: Experimental raytraced lighting system
- **DLSS**: Support for Nvidia RTX GPUs for upscaling/antialiasing
- Note: Early work, not production-ready yet

### Baked Global Illumination
For static scenes, consider `bevy-baked-gi`:
- Lightmapping
- Irradiance volumes
- Reflection probes
- Works with Blender or Unity for baking

## Full PBR Material Properties

```rust
StandardMaterial {
    // Base properties
    base_color: Color::WHITE,
    base_color_texture: Some(asset_server.load("albedo.png")),
    
    // PBR core
    metallic: 0.0,
    perceptual_roughness: 0.5,
    metallic_roughness_texture: Some(asset_server.load("metallic_roughness.png")),
    
    // Surface detail
    normal_map_texture: Some(asset_server.load("normal.png")),
    occlusion_texture: Some(asset_server.load("ao.png")),
    
    // Emission
    emissive: Color::BLACK,
    emissive_texture: Some(asset_server.load("emissive.png")),
    
    // Advanced
    depth_map: Some(asset_server.load("height.png")),  // parallax mapping
    parallax_depth_scale: 0.1,
    clearcoat: 0.0,  // extra translucent layer
    clearcoat_perceptual_roughness: 0.5,
    
    // Rendering
    double_sided: false,
    cull_mode: Some(Face::Back),
    alpha_mode: AlphaMode::Opaque,
    
    ..default()
}
```

## Quick Wins Checklist

1. ✅ Add HDR environment map
2. ✅ Set proper material metallic/roughness values (not defaults)
3. ✅ Use physical camera exposure
4. ✅ Enable shadows on directional lights
5. ✅ Add normal maps (with generated tangents)
6. ✅ Tune ambient light brightness

## Resources & Examples

**Official Bevy Examples:**
- `examples/3d/lighting.rs` - Various light types
- `examples/3d/pbr.rs` - PBR material showcase
- `examples/3d/atmosphere.rs` - Atmospheric scattering

**External Resources:**
- [Bevy PBR documentation](https://docs.rs/bevy/latest/bevy/pbr/)
- [Google Filament material properties](https://google.github.io/filament/notes/material_properties.html)
- [bevy-baked-gi](https://github.com/pcwalton/bevy-baked-gi) - Baked GI workflow

## Common Mistakes

❌ Using default metallic/roughness (0.5/0.5) for everything
❌ No environment map (flat ambient lighting)
❌ Not generating tangents for normal maps
❌ Light intensities too low or too high
❌ Forgetting to enable shadows
❌ No exposure control (overblown or too dark scenes)

## Version Notes

- **Bevy 0.16+**: Light textures, improved PBR
- **Bevy 0.17+**: Experimental raytracing (Solari), DLSS support, atmospheric scattering improvements
- Most techniques work on Bevy 0.14+
