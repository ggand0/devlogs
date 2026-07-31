# 0024 — TW-style orders, engagement & charge; war cry fix (2026-07-11)

Uncommitted on `feat/battle-feel`. Started as "war cries play nonstop"
(the vox_warcry spam made battles comedic) and ended, after three failed
inference-based attempts, as the owner-directed Total-War order model.
The lesson is the headline: **don't infer intent from density fields —
represent it.** Everything below is state the game knows for certain.

## Order model (`orders.rs`)

`Option<Vec2>` became `Option<Order>`:

- `Order::Move(Vec2)` — RMB on ground. Ends on arrival (point becomes
  the new anchor), exactly the old attack-move semantics.
- `Order::Attack(u32)` — RMB within `ATTACK_PICK_RADIUS` (20 m) of an
  enemy regiment's centroid. The destination re-resolves to the target's
  CURRENT centroid every tick (chases), never "arrives", and ends when
  the target is wiped (`attack target N destroyed, holding` — anchor set
  to the spot so the regiment doesn't march back to a stale anchor).
- `Groups::goal(g)` resolves either kind to this tick's destination;
  `movement.rs` consumes only that — the sacred movement kernel is
  untouched.
- All selected regiments attack the same target (no arrangement
  translation on attack, TW behavior).
- A red gizmo arrow hangs over any regiment the player is attacking
  (player-facing UI, drawn regardless of the G debug-viz toggle). AI
  attack orders draw nothing.
- AI (`ai_think`) now issues `Attack` orders directly from its existing
  target assignment (it previously marched to a stale centroid snapshot
  — this also upgraded the AI to chase).

## Engagement = any soldier fighting (`frontline.rs::update_groups`)

TW rule: one soldier in combat engages the whole regiment.

```
fighting(unit)     = alive && swing state == WIND-UP
engaged(regiment)  = any unit fighting, held ENGAGE_HOLD_TICKS (45,
                     ~1.5 s) past the last wind-up
```

WIND-UP is the only usable per-unit signal — verified the hard way:

- `units.target` is NEVER cleared after a swing (doc says "only trusted
  within a swing cycle") → permanently truthy after a unit's first
  fight.
- `units.swing` SPAWNS in Recover (strike-stagger, units.rs:156) →
  "state != Ready" was true for entire armies at tick 1. First
  verification run: all 80 regiments "ENGAGED" 1.6 s after spawn in
  every scenario, including a no-combat one.

A wind-up can't be stale and can't exist without an enemy in reach.
The hold bridges recover/ready gaps between swing cycles.

The old `ENGAGED_T` blurred-density-at-centroid definition is deleted.
Every consumer (arrival clearing, AI idle check, morale recovery gate,
far/mid audio beds, overlay count) now reads ground truth.

## Charge phase

```
charging = has Attack order && target alive
           && distance(centroid, target centroid) < CHARGE_RANGE (60 m)
           && not engaged && not broken
```

Pure predicate, evaluated per tick — no latch, no linger, no speed gate,
no density probes. It starts when an attacking regiment crosses charge
range and ends at PHYSICAL contact (engagement is real now), which is
what all the linger hacks were trying to approximate.

## Pose (render + shader, negative lunge band)

Per unit, priority: wind-up raise/chop → battle stance → carry.

- Battle stance 0.5 (blade leveled at the enemy, +0.25 rad): regiment
  engaged OR under an attack order — swords come up on the click.
- Charging 1.0: adds sprint lean (+0.24), +35 % stride, heavier bob.
- Otherwise: blade carried lowered (−0.55 rad) while moving; neutral
  forward hold standing. Scaled by walk amount so jammed units don't
  posture. Shader decodes the band with two smoothsteps (stance 0–0.5,
  sprint 0.5–1).

Pose history: a raised-arm hold (take 1) reads as CARRYING with this
chunky mesh; the forward point reads as charging — so ordinary moves
lower the blade and combat levels it.

## War cry (`audio.rs::event_cues`)

ONE rule (owner-corrected twice: a per-onset ack "cried at any
distance and bad"; the engaged-suppression made close attacks silent):
a regiment cries when **"attack target within CHARGE_RANGE" newly
becomes true** — attack ordered at close range (standing or engaged),
closing to 60 m on an approach, or RETARGETING to a different nearby
enemy mid-melee (M2TW). Far attack orders get the horn only; their cry
arrives at 60 m. Both teams.

- The edge cry plays immediately (beats the vox gate; prox-scaled
  volume, skipped only if effectively out of earshot) and opens a roar
  budget of `WARCRY_CLIPS` (3, ~8 s ≈ a heavy's 60 m run onto a
  stationary line).
- While an unengaged run-in (`charging`) lasts, the roar re-fires per
  2.5 s gate window until the budget dries — covers long approaches,
  can't loop forever on pursuits/jams.
- Re-charging the SAME still-in-range target does NOT re-cry (the
  condition never went false) — deliberate anti-spam.
- Clip pool: `vox_warcry_01` only (owner benched 02). Plays log
  `war cry (edge, vol, prox)` / `(vol, prox, N clips left)` for tuning.
- Loudness audit (ffmpeg volumedetect): warcry clips mean −14.5 dB,
  peak 0 dB — as loud as rout vox, louder than clangs. "I can't hear
  it" ≠ quiet asset; watch the vol/prox log values instead.

## Verification (LOG-based — owner: screenshots are not evidence)

Transitions logged per regiment: `CHARGES` / `CHARGE ENDS` / `ENGAGED` /
`DISENGAGED`, plus order lifecycle lines. 110 s runs:

- **AI skirmish** (attack orders): 27 charges; per-regiment lifecycle in
  the log reads `CHARGES → (2.4–3.7 s run-in) → ENGAGED → CHARGE ENDS →
  … → attack target destroyed, holding → DISENGAGED → CHARGES` on the
  next AI order. No rapid on/off pairs.
- **FL_TEST_FRONT** (move orders): 0 charges (moves never cry); first
  ENGAGED at t≈9.6 s = physical contact after the t=3 s advance; 101/51
  engage/disengage over the whole battle, worst regiment 7 transitions
  (mop-up cycles, not flicker). Its 15 s re-target now issues Attack
  orders at the nearest enemy regiment.
- **FL_TEST_ORDERS** (pure marches): 0 charges, 0 engagements; spread
  ratio 1.38 (acceptance < 1.5 intact).
- **FL_TEST_SURROUND**: pocket and line both end by MORALE BREAK at
  ~30 % losses (0021 behavior intact); 14 engage transitions total.

Not log-verifiable: the red attack marker (visual only) — owner eyeballs
it on first right-click.

## Knobs

`CHARGE_RANGE` 60 m (how far out the roar/sprint starts),
`ENGAGE_HOLD_TICKS` 45, `ATTACK_PICK_RADIUS` 20 m (RMB attack vs move
disambiguation), vox gate 2.5 s (roar density).

## Watch list (owner, 2026-07-11)

Feature is on probation — owner playtests read the charge/stance
transitions as correct, but he saw a possible one-off sword-drop near an
enemy unit. Most plausible mechanism: the ENGAGE_HOLD (1.5 s since the
last wind-up anywhere in the regiment) expired while enemies were still
adjacent but nobody was mid-swing, and the regiment had no attack order —
moving units then legitimately drop to the march carry. If it recurs,
candidates: longer hold, or a "ready stance" fallback when enemies are
within a few meters of any unit (needs a cheap per-regiment signal, NOT
the old centroid-density inference). Keep watching before building more
on top.

## Later

- Charge speed boost + unifying per-unit `SWING_CHARGE` (1.75× momentum
  damage) with the regiment charge phase — combat balance, deliberately
  out of scope.
- `units.target` staleness is a live landmine for anyone else reading it
  — either clear it on Ready-with-no-candidate or keep treating WIND-UP
  as the only truth.
- Attack-order UI: hover highlight on the would-be target before the
  click (pairs with the planned controls bundle).
