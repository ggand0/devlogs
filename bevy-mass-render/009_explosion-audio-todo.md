# TODO: Explosion Audio (2025-11-14)

## Next Steps

Add explosion sound effects to complement the visual explosion system.

### Asset Source

**Freesound.org** - Community-uploaded sound effects with Creative Commons licenses

### Requirements

- Explosion sound effects (short, punchy, varied)
- Multiple variations for randomization (similar to existing laser sounds)
- WAV format (Bevy supports `.wav` files)
- Appropriate licensing (CC0 or CC-BY preferred)

### Implementation Plan

1. **Download Assets** from Freesound.org
   - Search for "explosion" or "blast" sound effects
   - Download 3-5 variations for randomization
   - Save to `assets/audio/sfx/explosion0.wav`, `explosion1.wav`, etc.

2. **Add to Git LFS**
   - Audio files should be tracked by LFS (like existing laser sounds)
   - Already configured in `.gitattributes`

3. **Integrate into Explosion System**
   - Add to `ExplosionAssets` resource in `src/explosion_shader.rs`
   - Play sound in `spawn_custom_shader_explosion()` function
   - Use spatial audio with distance falloff (like laser sounds)
   - Randomize between variations
   - Consider audio throttling for mass explosions (similar to laser system)

4. **Performance Considerations**
   - With quantized bursts (12-67 explosions/frame), audio throttling is critical
   - Limit max explosion sounds per frame to prevent audio saturation
   - Tower explosions should always play sound (high priority)
   - Unit explosions could use probability gating (similar to particles)

### Reference Implementation

Existing laser audio system in the codebase provides a good template for:
- Multiple sound variations
- Audio throttling (max 5 sounds per frame)
- Spatial audio setup

### Current Status

- Visual explosions: ✅ Complete (sprite sheet + GPU particles)
- Audio: ⏳ Pending (assets needed from Freesound.org)

---

**Note**: This is a quick TODO note for future audio implementation.
