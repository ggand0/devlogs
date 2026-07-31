# 0028 — Controls bundle, audio batch 3, animation variants (2026-07-13)

Commits `d3477b6` (audio), `0e80bc9` (controls), `d058c93` (animations)
on `feat/battle-feel`. Last Fable session of the sub period — owner will
renew for the formations milestone.

## Controls (`camera.rs`, `orders.rs`)

- **S = halt** while regiments are selected: orders dropped, anchor set
  to the current centroid so the block holds where it stands. With
  nothing selected S remains camera pan-south (camera checks
  `Selection`).
- **Ctrl+1..9 / 1..9**: store/recall selection masks (`ControlGroups`
  resource). Dead regiments drop out via count checks on recall.
- **Edge pan**: cursor within 12 px of a window border pans, camera-yaw
  relative, speed scales with zoom, diagonal clamped.
- **Smooth zoom**: scroll drives `target_distance`; `distance` chases it
  with a 0.12 s time constant.

## Audio batch 3 (owner-generated, `d3477b6`)

March bed (freesound boots loop, plays 0.28 with the drums when own
regiments march), ui_attack (0.45) / ui_order0 (0.35) / ui_select1 (0.5)
click feedback — order cues now fire on order CHANGE so re-orders click
too. Rally pool split: 01/02 = rally-from-rout, 03/04_celebrate =
victory cheer when an ENEMY regiment breaks (layered over their rout
wail). Remaining to generate (also in Claude project memory):
bodyfalls, sig_horn_rout_short (+ the double-blast mixer wiring), and
optionally wire the existing freesound wind files as a base bed.

## Animation variants (shader-only, zero new instance data)

All seed-driven (`i_color.a`, the stable per-unit anim seed), so corpses
and repeat views stay consistent:

- **Three attack styles** (`fract(seed*7.31)`): 40 % overhead chop
  (original), 35 % stab (arm draws back 0.30, thrusts +1.05 in local Z,
  blade near level), 25 % horizontal slash (rot_y sweep −1.1 → +2.3
  around the body axis). All reuse the raise/chop curves so the blow
  lands exactly at lunge 1.0 = the damage tick — visuals changed, combat
  timing untouched.
- **Taunt**: standing units (walk ≈ 0) of a regiment in battle stance
  pump the blade to ~1.7 rad with an 16 Hz shake for ~1.5 s every ~7.3 s,
  phase-staggered by seed. Rear ranks jeer while the front fights.
  Enabled by un-scaling the stance band from walk amount (the sprint
  lean is walk-gated in the shader instead, so jammed chargers still
  don't posture).
- **Death fall variants** (`fract(seed*13.73)`): ~45 % forward topple,
  ~17 % backward, ~38 % sideways (either side). Corpse buckets keep the
  seed → ground poses persist.

Verified by close-up screenshot mid-melee: mixed blade angles across the
line, rear-rank taunts visible, corpses in varied orientations, 93 fps
at 6k units with the inspect panel live.

## Round 2 (owner playtest, commits `19a806f`..`d031c86`)

- **W-key bug**: cursor parked at the bottom screen edge made edge-pan
  exactly cancel W. Keyboard pan now takes priority over edge pan;
  **halt moved from S to Backspace** (the TW keybind), S is pure camera
  pan again.
- **Attack styles are per-SWING** now, picked by the sim at wind-up
  start (spare `swing` bits 3-4, encoded as the 2s digit of the
  positive anim band). The overhead chop — which WAS the original swing
  animation — is retired per owner taste. Pool = stab/slash; TEMP
  pinned to STABS ONLY while the owner evaluates (one-line switch at
  the style pick in movement.rs).
- **Bracing** (owner priority over taunt polish): standing units of a
  regiment with enemy in watch range — engaged, attacking, or merely
  APPROACHED (new 0.25 stance-band tier from `enemy_near`) — plant a
  split-leg stance (legs ±0.24 rad), crouch 0.05, blade raised +0.35 to
  a ready guard, and TURN TO FACE the nearest enemy regiment
  (`threat_dir` per regiment from update_groups, wired into the facing
  priority chain for near-stationary units). Screenshot-verified: both
  front lines brace facing each other pre-contact; melee is all level
  thrusts, no chops.
- Owner asset housekeeping tracked: trimmed ui_select, vox_rally_03 →
  _03_celebrate rename, batch-3 files committed. Unwired-but-present:
  three wind ambiences, a 25 s march variant, _org backups, voice/.

## Round 3 (owner, `6402c33`)

- Attack pool = **stab + classic swing**, random per swing (owner kept
  the stab after evaluation; slash benched in the shader).
- **Morale-gated taunts**: stance band gained a tier — fighting at
  morale > 50 sends 0.65 (taunts allowed), ≤ 50 sends 0.5 (wavering:
  brace only, no jeering). Sprint threshold moved to 0.7.
- **Brace strengthened** (owner couldn't see round 2's: too subtle at
  RTS zoom + narrow windows since attacking regiments RUN): legs
  ±0.32 rad, crouch 0.07, and the arm is a per-unit 50/50 mix of the
  forward POINT (owner's favorite old stance) and a raised guard.
  Screenshot: a standing regiment reads as a wall of mixed points and
  guards, all facing the threat.

## Round 4 (owner, `5e79cca`)

- **Whole-pose standing mix**: the round-3 mix only varied the ARM;
  owner wanted some soldiers in the plain old forward-point stand and
  others fully braced. The per-unit pick now gates the ENTIRE brace
  (legs + crouch + guard); non-bracers are exactly the old idle point.
  Guard raised to a proper 0.6 rad ready position.
- **Braced walk**: advancing in enemy watch range lifts the blade most
  of the way to level (carry mixes on `ready`, not just `stance`) with
  a slight crouch — the last 60 m of an approach reads tense.
- **Victory cheer** (distinct from taunt): falling edge of
  `hostile_near` (any UNBROKEN enemy regiment within 60 m) sets
  `celebrate` ~5 s on the survivors — blades pumped skyward + bounce
  hop, encoded as positive-band style digit 3 (celebrants have no one
  to swing at, so the channel is free). Cancelled if a new hostile
  closes. Logged `regiment N CHEERS`; trigger log-verified (opponent
  BREAKS → next-tick CHEERS). Pose shares the verified taunt/brace
  mechanisms; owner eyeballs the first won melee.

## Post-mortem: the invisible-army bug (`4f21d47`)

Round 4's bracer pick used `seed` before its declaration. WGSL compiles
at RUNTIME — cargo build stays green — and a failed pipeline renders
NOTHING while the overlay still reports "drawn N". The round-4
verification screenshot showed exactly that (counts up, screen empty)
and got misread as "camera missed the battle". Rule going forward: any
run after touching a .wgsl must grep the log for
`failed to process shader` BEFORE trusting a visual, and an empty frame
with a nonzero drawn count is a pipeline failure, not bad framing.

## Round 5 (owner, `208659c`)

- Standing brace **benched** ("weird") behind `BRACE_ON` (shader const,
  0.0) — standing units near an enemy hold the plain forward point
  again. The braced-walk blade lift on the approach stays. The whole
  brace impl (split legs, crouch, guard, per-unit mix, threat facing)
  is intact for a re-look, likely with a better pose.
- **Pose transitions smoothed** (owner: one-frame stance changes aren't
  immersive): stance tiers are per-regiment and snap, so a per-unit
  ~0.35 s EMA (`band_ema`, beside the walk EMA) now carries the pose —
  blade leveling, sprint lean, taunt gate, and any future brace all
  blend. The cheer encodes progress (z = 6 + t) and eases in/out
  (8%/10% ramps).

## Round 6 (owner, `5feaca7`)

Taunt benched behind `TAUNT_ON` (owner rework planned; impl kept). The
5v4 TEMP hunks became a real knob: `FL_ENEMY_REGS` caps enemy regiment
count, run_5v5.sh sets it, FL_UNITS default back to 100k — working
tree finally clean. Branch considered PR-ready pending owner verify;
refactoring review next, clippy sweep on its own branch after.

## Futureproofing notes

- Attack style is derived from the seed — a per-kind style table (e.g.
  spears mostly stab) just reweights the thresholds by `kind`, which the
  shader can read from the bucket (kind = bucket) if ever needed.
- The taunt gesture is one pose; alternates (fist pump, shield bang)
  can branch on another seed hash in the same gated window.
- March-in-step (roadmap) needs a REGIMENT-synced walk phase — that
  wants a per-regiment phase value in instance data (or seed quantized
  per group) and interacts with the tonal-variation seed; deliberately
  left for the formations milestone.
