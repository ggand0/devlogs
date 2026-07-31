# 0011 — Hold-the-line fix, craters off, combat/setting decision (2026-07-09)

## Shipped (`bf3363f`)

- **Hold the line**: armies interpenetrated on contact because the phi=0
  contour only existed after densities overlapped. Blur widened (3 passes)
  + CONTACT_T 0.35 → 0.12: the line now forms while armies are ~25 m apart
  and approach units lock onto it. Verified: razor-sharp interface, zero
  mixing.
- **Death craters disabled** (CRATER_CHANCE = 0). Mechanism kept for future
  explosives/artillery.

## Owner verdict on combat

The statistical pair-damage model is out ("pseudo logic"). Wants M2TW-style
actual attacks: each soldier swings/shoots with a timer at a chosen target;
being surrounded means more incoming swings → dying faster emerges. Also:
if guns, projectiles must be individually simulated. Setting undecided:
medieval / pre-modern / modern / futuristic — decide by scalability.

## Setting scalability analysis (see chat 2026-07-09)

Cost drivers at 100k units: (a) active combatants (contact band vs
everyone), (b) live projectile count = fire rate × flight time, (c) VFX.

- **Medieval melee**: combat confined to the contact band (~1-2k active),
  no projectiles. Cheapest; preserves the line-pushing core. M2TW swing
  model drops into the existing neighbor loop.
- **Napoleonic**: volleys = projectile bursts (10-50k peak, short-lived);
  line warfare fits the frontline solver perfectly. Smoke VFX cheap.
- **Modern**: sustained autofire = 100-300k live projectiles (still fine as
  a SoA pool) BUT 300 m engagement ranges dissolve the physical battle
  line — the core mechanic — into cover/suppression gameplay. Needs
  "somewhat realistic" explosion VFX, which fights the flat-shaded art.
- **Futuristic**: fire rate / range / projectile speed are free design
  dials (no realism constraint) → tune for both perf AND readable fronts;
  stylized energy VFX matches the chunky art direction.

**Recommendation**: medieval/pre-gun melee as the core now (it IS the
WoD × M2TW hybrid; scales best; keeps the line the star), architected so
projectiles are a SoA pool + instanced draw ("individually simulated"
≠ one Bevy entity per bullet — that's the per-unit-entity mistake again).
Futuristic is the best choice IF ranged is wanted as the fantasy.

## Confidence notes (Bevy)

- Swing-based melee at 100k: high — same neighbor data, add target pick +
  cooldown column, apply damage via per-chunk event buffers (no races).
- Projectile pool (up to ~300k live): high — same SoA + instanced pattern
  as units, grid collision.
- Stylized chunky explosions (flash, shockwave ring, debris cubes, crater):
  high — custom instanced pipelines, matches art.
- "Somewhat realistic" modern VFX (volumetric smoke/fire): moderate — needs
  bevy_hanabi (new dependency) or custom GPU particles; real time sink and
  clashes with the flat-shaded look. Would advise stylized instead.
