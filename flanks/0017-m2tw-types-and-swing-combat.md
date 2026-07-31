# 0017 — m2tw branch: unit types, low-poly meshes, swing combat (2026-07-09)

First two milestones of the M2TW-lite MVP (plan: `docs/plans/m2tw-mvp.md`,
branch `feat/m2tw`, commits `61d52a3` + `8566bd5`).

## Unit types + render path (`61d52a3`)

- `unit_types.rs`: data table indexed by the new `kind` column. HEAVY
  knights (slow, 160 hp, hard swings, mass 2.0) and LIGHT men-at-arms
  (fast, 90 hp, quick swings, spear reach). FL_HEAVY_FRAC (default 0.4)
  spawns heavies in the front rows.
- `unit_meshes.rs`: code-built merged-cuboid meshes, flat per-face
  normals — knight 9 cuboids/108 tris (wide torso, pauldrons, helmet brim,
  shield slab), man-at-arms 7/84 (slim, vertical spear + tip). Render
  buckets now key by KIND (one instanced draw per mesh); team is
  per-instance color, as always.
- InstanceData v2 (48 B): anim vec4 at shader location 5 = yaw, move
  amount, lunge, fx. Color alpha repurposed as a stable per-unit anim
  seed. Shader rotates position AND normal by yaw, walk bob + lean by
  move amount, hit-flash lerps to white, death topples about the feet.
- Sim owns `yaw`: smoothed atan2 of the steering output (faces the swing
  target during wind-up).
- Costs at 200k: step +~1 ms (yaw atan2), sync 1.5–3 ms at 48 B, GPU
  transparent pass 3.0 ms at ~178k drawn (~17M tris) — the 10× triangle
  jump cost only 2×, far under the 4–8 ms projection. FL_NO_CULL draws
  exactly 200000 [80000/120000].

## Swing-timer combat (`8566bd5`) — the 1:1-trade artifact is dead

Design followed devlog 0011's verdict: actual attacks, not statistics.

- Per-unit state machine in the parallel integrate (all writes row-local):
  Ready → nearest living enemy in reach from the fused scan (sticky to the
  previous target → duels) → WindUp (feet planted, faces target) → strike
  emits `DamageEvent{victim, attacker, dmg}` → Recover (±25% jittered
  cooldown). Spawn staggers first cooldowns: no metronome wave on contact.
- Damage: per-chunk event buffers, then ONE serial apply pass —
  deterministic, race-free, <0.1 ms at 30–50 events/tick. Hit-time
  validation there (dead / out-of-reach / swap-remove-drifted index =
  whiff, which is correct game feel). Kills are counted at the hp≤0
  transition; `death_t` starts an 18-tick corpse (no combat, still
  collides, topple anim) before the sweep removes it.
- NO attacker cap. Surrounded = more independent swing streams = dies
  faster, emergently.
- Mass-weighted separation: knights bulldoze light infantry aside.
- `SortedUnit.team` → `meta` bits (team|kind|dying): the scan touches no
  SoA arrays and stays branch-light — the SIMD kernel (stretch PR) needs
  exactly this shape. Corpses excluded from targeting for free.
- CombatTuning/FL_DPS deleted → FL_COMBAT_SCALE multiplier.

### Acceptance (FL_TEST_SURROUND, new script)

Two equal 500-unit blue detachments vs the same 4:1 odds, one encircled,
one in line: **pocket annihilated in 48 s, line fight lost only 265**
(~3× per-capita death rate). With no orders the fight stalls at the
separation standoff — correct per "orders are sacred"; the test gives the
orange groups press orders.

### 200k battle (FL_TEST_FRONT, FL_COMBAT_SCALE=4)

step 6.3–8.0 ms (gate ≤8; QUERY_RADIUS grew 1.7→2.0 for spear reach),
grid ~1.9, sync 2.6–4.0, hits/tick 28–49. Casualty curve monotone, front
sharp, hit flashes visible along the contact band. Kill RATE is
structurally slower than the gather model (front-pairs × swing cadence,
not band-wide DPS) — that's the M2TW grind; battles resolve via morale
(next milestone), not annihilation.

## Notes for the next milestones

- Regiment cohesion (PR3) will end the "no orders = fight stalls at
  standoff" behavior for held ground: hold-at-anchor is a standing order.
- step's headroom shrank; if PR3 pushes it past ~8 ms, the SIMD kernel
  (PR6) moves up.
