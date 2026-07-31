# 0032 — Phase A: M2TW stat model + directional defense (2026-07-14)

Formation combat v2 (docs/plans/formation-combat-v2.md, research 0031),
phase A of four. Owner answered the plan's open questions before work
started: YES to the M2TW TTK texture (elite frontal fights get grindy,
rear kills get lethal), NO levy kind for now (keeps the last 2-bit grid
kind slot free for archers).

## What changed

### unit_types.rs — stats are split, not merged

`damage` is gone. Each kind now carries an M2TW-style row:

| kind  | atk | chg | skill | armour | shield | weapon | frontal def |
|-------|-----|-----|-------|--------|--------|--------|-------------|
| heavy | 13  | 5   | 6     | 5      | 5      | Sword  | 16          |
| light | 9   | 3   | 4     | 3      | 4      | Sword  | 11          |
| spear | 7   | 2   | 3     | 4      | 6      | Spear  | 13          |

Scaled from the vanilla EDU (0031): heavies ~ Dismounted Feudal
Knights, spears ~ upper Spear Militia with the anti-infantry penalty
folded into attack. `ap: bool` (halves armour) is implemented but no
current kind carries it.

Damage per hit = `BASE_DMG[weapon] * 1.13^clamp(factor, -12, 12)`,
factor = attack + situational − (skill' + armour' + shield'). We keep
hp/damage instead of M2TW kill rolls so pacing anchors survive.
BASE_DMG (Sword 23, Spear 32.5) anchors factor parity: light-vs-light
frontal = 18 dmg and spear-vs-light frontal = 20 dmg, exactly the old
flat values.

### movement.rs — the damage apply pass resolves direction

DamageEvent slimmed to (victim, attacker, jit, charge); ALL modifiers
moved from swing-emit time to the serial apply pass where both sides'
full state is known. Per event: sector of the blow against the victim's
yaw (front = within 60°, rear = beyond 120°, else side; victim's left
found via the facing cross product — facing +Z means left = +X):

- defence skill counts vs front/side, NOT rear
- shield counts front + LEFT side only (shield arm), never rear
- armour counts everywhere (halved by ap)

Wall stances became stat points instead of multipliers, so they obey
directionality for free: shieldwall = +2 shield (only where the shield
covers — a shieldwall still has no back; old model absorbed 0.7 from
EVERY direction) and −1 attack; braced spearwall = +2 attack.
CHARGE_MULT (flat 1.75) replaced by the attacker's charge_bonus in
attack points (heavy 1.85×, light 1.44×, spear 1.28×); a braced
spearwall victim still nullifies it (0030 rule, unchanged).

Cost: one dot/cross + one powf per damage EVENT (few hundred/tick
peak) in the serial pass — noise. The parallel integrate is untouched.

### Frontal TTK matrix (computed, 1v1, mean cycle)

| att\vic  | heavy          | light        | spear         |
|----------|----------------|--------------|---------------|
| heavy    | 20.0s (was 9.4)| 6.1s (5.3)   | 8.7s (5.9)    |
| light    | 22.9s (12.4)   | 7.0s (7.0) ✓ | 9.9s (7.8)    |
| spear    | 23.7s (12.8)   | 7.2s (7.2) ✓ | 10.2s (8.0)   |

Rear multipliers (skill+shield gone): ~2.4–2.7× vs same-kind frontal,
3.5× light-vs-heavy-rear (armour is all that's left, and knights carry
the most of it). Heavies are now genuinely hard to kill from the front
and just as soft as anyone from behind — facing is the defensive
resource. This is the owner-approved texture shift; morale (0025's
flanking ring) still decides fights long before annihilation.

## FL_TEST_DIR — the new acceptance

Two hammer-and-anvil pairs of 500 lights: victim on HOLD facing +Z,
pinned frontally at 35 m; second attacker from 55 m arrives ~2.5 s
later — from the LEFT side in the control pair (shield arm covers:
factor-identical to frontal by design) and from the REAR in the test
pair. The apply pass buckets every hit on a blue victim by its actual
sector at hit time (DirTestStats: hits/kills/dmg per sector).

Measured (deterministic across runs):

- dmg/hit by sector: front 18.5 / side 24.1 / rear 48.0 — theory says
  18 (factor −2), left-right mix ~24, rear 47.9 (factor +6).
  Rear/front per hit = 2.6× ≥ 2 ✓. Rear kills take exactly 2 hits
  (94.8 dmg/kill), frontal ~6.
- Kill rate, concurrent-engagement window (t=16–22): rear 22.3/s vs
  frontal 5.6/s per regiment = 4.0× ≥ 2 ✓ (aggregate totals dilute to
  1.1× only because the anvil fights alone for the first ~14 s).
- Outcome texture: test pair ANNIHILATED by t=26 (routers flee into
  the rear attacker and die); control pair breaks ~70% casualties and
  149/500 escape the field. The hammer-anvil reads exactly like M2TW.
- Left-side control tracks frontal (198 vs 210 kills/regiment):
  sector + shield-arm code confirmed against its own design.

## Acceptance sweep (vs pre-phase baselines)

Baselines: FL_TEST_SURROUND run on this branch pre-change;
FL_TEST_FRONT/ROUT from a clean HEAD worktree at `../cascade_base`
(left in place — phases B/C/D will A/B against it too). No stat
retuning was needed; the computed anchors landed on first measure.

- SURROUND: line control 115/500 vs baseline 114/500 — frontal TTK
  parity at population level. Pocket dies FASTER (0/500 by t=38 vs
  baseline 36/500 still standing): directional defense makes real
  encirclement per-soldier lethal, which is the point. Both blue
  detachments still end by BREAK ✓.
- ROUT: breaks in the ~50% band both models (first ROUTING at 508
  alive baseline / 484 new); 465 flee vs 489 (the envelopment cuts a
  few more down). Band intact ✓.
- ORDERS (200k): long-march spread ratio 1.29 (< 1.5, 0030 had 1.31),
  ZERO engage events ✓. Stage-3 straggler quirk unchanged (known).
- FORM: slot err 0.68–0.91 m, facing err ≤ 0.02 rad, nn wall 1.06 <
  normal 1.36 < loose 1.88 — identical to 0030 ✓.
- Perf (200k FL_TEST_FRONT, engaged, mean of last 40 samples): step
  8.11 ms vs baseline 8.67 ms — within budget (+0.5 ms allowed); the
  emit path actually got cheaper (wall_out branch moved to the serial
  pass). Held-regiment drift 2.28 m (0030 band: 2.40/2.47) ✓.

## Files

unit_types.rs (stat rows), movement.rs (DamageEvent, apply pass,
DirTestStats), regiments.rs (spawn_dir_test + dir_test_log), ai.rs
(FL_TEST_DIR stand-down).

## Play-test session fallout (same day)

- **Drag-line facing bug FIXED (ad51301)**: the drawn front line chose
  facing away-from-selection with ties falling to the raw drag
  perpendicular — dressing a line in place made facing follow the drag
  hand (left-to-right = backward). Now the facing side is the
  strength-weighted enemy-army side of the line (fallback: away from
  selection when no enemy signal). FL_TEST_FORM gained a drag-direction
  invariance stage; whole test green.
- **FL_ARMY_GAP knob (9796b13)**: army spawn gap was a 60 m constant.
- **FL_ENEMY_STATIC=1 (8c36284)**: enemy spawns in hold + AI stands
  down — practice-dummy army; still defends in melee, still routs.
- **Formation auto-face REMOVED (0bcb771) — the big one.** Owner
  play-tested a lone rear charge on a static heavy block: no kill
  difference. Root cause: the brace-facing branch turned every
  standing soldier of a regiment toward the nearest enemy regiment
  within 60 m at YAW_RATE 10 — a free 0.2 s about-face for 1000 men,
  so unpinned rear attacks always landed on fronts. Researched actual
  M2TW behavior (heavengames/GameFAQs/org): formed units NEVER rotate
  themselves — they go Idle->Ready in place (brace, no turn), facing
  is the player's job, and only individual soldiers turn to fight
  attackers that reach them (gunpowder units are the lone exception).
  Fix = swap branch priority: formed regiments dress to ORDERED
  facing; only Blob/broken groups keep threat-facing. The fighting rim
  still turns via swing-target facing (owner's "outer row" behavior,
  for free). FL_TEST_DIR pair 3 (lone rear charge + victim yaw probe):
  yaw dev ~0 pre-contact, ~1.0 rad mid-fight (rim only), rear charge
  now WINS an even 500v500 (victims break ~t=46) instead of mirror-
  grinding. SURROUND/ROUT/FORM unchanged.
- **TEMPORARY defaults in the working tree** (units.rs, regiments.rs,
  marked TEMPORARY in comments, do NOT commit): FL_UNITS 5000,
  FL_ENEMY_REGS n-1 (5v4), FL_ARMY_GAP 250 — the owner's phase-A
  play-test sandbox. Restore 100_000 / n_regs / 60.0 after sign-off
  (they were briefly restored for commits 0bcb771/8c36284 and
  re-applied).
