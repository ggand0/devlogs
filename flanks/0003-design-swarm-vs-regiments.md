# 0003 — Design note: swarm vs Total War-style regiments (2026-07-08)

## The question

Can this engine pivot from "Creeper World-like continuous swarm" to a
Total War-like game — discrete regiments of units instead of one flowing
mass? (The 2D cylinder collision already feels like M2TW's barrel collision.)
Undecided for now; either way the engine core is the deliverable.

## Assessment: yes, the pivot is cheap. The core is genre-agnostic.

What we've built so far assumes only "very many units, grouped, not
individually entities". Nothing in it is swarm-specific:

| Engine piece | Swarm game uses it as | TW-like game uses it as |
|---|---|---|
| SoA unit buffers + instanced draw | 100k blob members | 5–20k soldiers (trivial by comparison) |
| Spatial hash grid | separation, combat range | collision, charge impacts, melee pairing |
| Separation + positional correction | crowd pressure physics | exactly M2TW barrel collision (2D cylinders) |
| Group entities + neighbor links | blobs with connected fronts | regiments in a battle line |
| M5 slot assignment along a curve | organic front line slots | rank-and-file formation slots (grid instead of density curve) |
| Group orders (M4 attack/cut/stance) | salients, blob splitting | regiment orders, formation width/depth |

The M5 frontline solver is the fork point: slots sampled along a
density-derived contact curve (swarm) vs slots from an authored formation
grid with facing (regiments). Same slot-steering code underneath — a
regiment is a group whose slot layout is rigid and whose cohesion is strong.

## What a TW pivot would add (none of it invalidates current work)

- Per-unit yaw/facing (instance data grows by a rotation; shader lerps fine)
- Formation-keeping as primary steering (slot attraction dominates goal drive)
- Morale/rout state per group; charge momentum (mass × velocity impulse
  through the same positional-correction channel)
- Fewer units, richer per-unit state — SoA layout welcomes extra columns

## Decision for now

Keep executing the swarm milestones (M3 terrain → M6 combat). Design rule to
protect the pivot: **group logic stays behind the group abstraction** — sim
systems read per-group parameters (slot layout, cohesion, stance), never
hardcode "the swarm". If we later want regiment mode, it's a different slot
generator + tuning profile, not a rewrite.
