# 0045 — FL_MAP=river: classic/new map swap (2026-07-20)

Implemented by Kimi K3 (OpenCode), directed and reviewed by the owner.

`FL_MAP=river` runs the map-art generation; anything else (default) is
the classic pre-map-art map. (First landed inverted — `FL_MAP=old`
opted OUT — the owner preferred classic as the default while the new
map cooks.) The flag is read once (`map_is_classic`, OnceLock) and
carried on the `Terrain` resource (`classic` field) so every consumer
branches locally:

- `generate_terrain`: classic skips terraces + river carve and leaves
  the blocked mask all-false; crater carves skip mask refresh too.
- `band_color`: wheat patches / tone wobble only on the new map.
- `height_at` / `wade_mult`: deck override and wade slow are no-ops on
  classic.
- `water.rs` (water strip + bridge) and `vegetation.rs` spawn systems
  early-return on classic.

## Verification

- FL_MAP=old + FL_TEST_DIR: 511/310/594, per-hit 22.6/30.0/49.1,
  yaw dev 0.00 rad — byte-identical to the devlog-0042 baseline. This
  also cleared the earlier 2.94 rad yaw reading on the new map: it was
  a different lone survivor from shifted terrain heights, not a facing
  regression.
- New map default path re-smoked after the flag landed (vegetation
  spawns, zero render errors).
