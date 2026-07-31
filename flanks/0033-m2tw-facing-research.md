# 0033 — M2TW facing research: units do not turn themselves (2026-07-14)

Owner play-tested phase A (0032) with a lone heavy regiment against a
static heavy block: attacking from the front vs the rear produced no
meaningful casualty difference (~20-30, inside noise). His read was
that soldiers facing the nearest enemy neutralizes directionality, and
that only the outer rank should face threats while the rest keep the
ordered facing. He asked for the actual M2TW behavior instead of an
invented rule. This entry is that research and what came of it.

## What M2TW actually does (community sources)

- **Formed units NEVER rotate themselves toward a threat.** A
  stationary unit with an enemy approaching transitions its tooltip
  state Idle -> Ready: weapons and shields come up and the unit
  resists charges better — but its FACING does not change. Facing is
  entirely the player's job (alt+RMB drag to set it). Leaving a flank
  or rear open is supposed to cost you the unit.
- **The one exception is gunpowder units** (arquebusiers, musketeers),
  which self-rotate to bear on targets. Nothing else does.
- **At melee contact, response is per-soldier**: each man fights his
  own combat and turns to fight an attacker that reaches him; men hit
  from behind stagger first, then turn one by one. The formation as a
  whole never wheels on its own.
- **Guard mode** = hold position, keep cohesion, do not chase routers;
  spear/halberd/pike units brace continually in it. It does NOT add
  auto-facing (my earlier assumption was wrong).
- **Rear defense is a fraction of frontal** (roughly one third for
  typical units — skill and shield gone, armour only), which combined
  with no auto-facing is exactly why rear charges are catastrophic.

Sources: Total War Heaven "General Strategy for Newbies" + Formed
Charge tutorial (medieval2.heavengames.com), Proudnerd's M2TW guide
(GameFAQs 81512), org guard-mode thread (forums.totalwar.org 50097),
TWC guard-mode threads, Steam M2TW unit guides.

## What our engine did instead (the bug 0032 shipped against)

`threat_dir` (nearest enemy regiment inside 60 m, frontline.rs) fed a
facing branch in the integrate loop that turned EVERY near-stationary
soldier of the regiment toward the threat at YAW_RATE 10 — one third
of the remaining angle per tick, a 180° about-face ~90% done in 0.2 s.
Free, instant, whole-block. Any unpinned rear approach was met by a
pirouette long before contact, so the phase-A sector test only ever
saw fronts. (FL_TEST_DIR passed because its victims were pinned
frontally first — hammer-and-anvil was honest, lone flanking was not.)

## The fix (0bcb771) — a deletion, per the reference

Branch priority swapped: formed (Rect, unbroken) regiments dress to
their ORDERED facing when standing; only Blob mobs and rallied
remnants still turn toward the enemy mass. The owner's "outer rank
faces the enemy, interior holds the line" needed no new code — a
soldier already faces an enemy that enters his ~2 m combat scan, and a
swinging soldier faces his target, so the fighting rim turns man by
man exactly like M2TW while the ranks hold. The render brace pose
(keys off enemy_near, not yaw) is the Idle->Ready weapons-up moment.
Engaged or bodily blocked soldiers can't comply with anything anyway —
compliance is physical, not a rule.

Verified via FL_TEST_DIR pair 3 (lone rear charge on a holding block,
mean-yaw probe): yaw dev ~0 until contact, ~1.0 rad mid-fight (rim
only), and the rear charge now WINS an even 500v500 (victims break at
~t=46, run down fleeing) where it used to mirror-grind. Control pairs,
SURROUND, ROUT, FORM, ORDERS all unchanged.

## Why the owner STILL doesn't feel it block-on-block (honest gap)

Per-hit rear damage is 2.6x (lights) to 3.8x (heavies) — measured.
But in a block fight only the ~40-50 man contact rim fights at any
moment, and a rear-attacked rim man turns to fight within a swing or
two. So phase A alone turns a rear attack into "wins the grind
clearly" (see pair 3), not the M2TW "unit evaporates in seconds".
That moment is carried by the still-missing phases:

- **B — charge impact**: the rear charge physically knocks the back
  ranks flying and staggers them (can't answer for ~1 s).
- **D — shock morale**: M2TW's charged-in-rear one-time morale hit.
  Our morale reads density envelopment only — TODAY A PURE REAR ATTACK
  FROM ONE SIDE SCORES THE SAME AS A FRONTAL ONE in the morale system.
  Phase D is precisely this.
- **C — rank discipline** keeps the victim block from smearing into
  the attacker (the mush currently lets victims rotate freely as
  individuals once contact deepens).

Phase A built the substrate those phases multiply.

## Round 2: still identical in the arena — two more mechanisms (same day)

Owner re-tested and reported no difference; FL_ARENA_AUTO (the same
Order::Attack the click writes, scripted) reproduced it: lanes within
noise. Two further mechanisms were hiding the sector math, both found
by measuring, both fixed (b11b680):

1. **Free 360° awareness in the swing machine.** A READY soldier
   opened a counter-swing on any enemy entering his 2.6 m scan, and
   the wind-up facing spun him — victims were face-on before the
   attacker's first blow landed (both wind-ups run the same ticks).
   Fix: a man only opens on a target in his forward half-plane UNLESS
   he was just struck (flash) — the first blow lands in his back,
   turns him, then he answers. Every fresh rank a rear attack reaches
   pays that toll.
2. **Morale could not see direction.** Even with rear hits landing,
   hp pools smooth M2TW's kill-rolls: a man at 44% hp fights at full
   output, so attrition alone gave only a ~1.3x exchange edge — and in
   M2TW a rear attack wins through MORALE (rout, then pursuit), not
   attrition. Pulled phase D's core forward as a STANDING drain (the
   M2TW attacked-in-rear modifier, not a one-shot penalty): the damage
   apply tallies per-regiment hits by sector (~1 s decay), morale.rs
   converts saturation-normalized pressure into MORALE_REAR_ATTACK 6.0
   / MORALE_FLANK_ATTACK 2.5 drain in the psych channel (depletion +
   resist + wall damping apply). A first cut gated on charge-flagged
   hits never fired — the crowd brakes chargers below charge speed at
   contact, so the flag barely survives; wounds themselves are the
   signal. Log line: "regiment N is taking blades in the REAR".

**Arena, after (identical 500-heavy duels, auto):** rear-attacked
enemy logs the rear line at contact, morale diverges immediately (32
vs 55 at t=40), BREAKS at ~50% casualties at t=50, rout slaughtered —
attacker ends 307/500 vs 31. Frontal control grinds 25 s longer,
breaks at 65% casualties, near-mutual destruction (231 vs 173). Same
regiments, same distance. That is the hammer moment, pre-knockback.

**Band shifts (phase D re-measurement, owner sign-off pending):**
ROUT breaks at 568/1000 alive = 43% casualties (was ~50%) — the three
converging regiments' side/rear wounds now drain; fled 545 (was 465).
SURROUND: pocket unchanged (dead by ~t=38), line control ends 79/500
(was 115) — line-end wrap-around now hurts for real; both still end
by BREAK, pocket still dies far faster. DIR gates hold (rear/front
per hit 49.0/18.6 = 2.6x); the lone-rear victim collapses by t≈44.

## Round 3: the wound tally was a cheat — perception, not bookkeeping

Owner (correctly) rejected the sector-hit tally: M2TW's attacked-in-
rear modifier is SITUATIONAL — who stands engaged on which arc — not
a count of which direction wounds arrived from. Soldiers telepathically
reporting "that one hit my back" to a regiment meter is bookkeeping,
not perception. Replaced (beafabd): the existing 8-probe morale ring
already reads hostile density per bearing; weight each hostile probe
by its angle to the regiment's FACING — rear arc 1.0, side arcs 0.4,
front 0. Two fully hostile rear probes = full drain (MORALE_REAR_
EXPOSED 6.0, psych channel). Formed regiments only (a mob has no
rear). The encirclement safe-arc term stays direction-agnostic beside
it; the two stack for the hammer-anvil.

Better in every measurement: fear starts as the attacker CLOSES IN
behind (pre-contact waver, like M2TW cavalry circling), the arena rear
lane breaks at ~45% cas by t=42 (attacker keeps 349/500 vs 23; frontal
control grinds to t=92, 206 vs 140), and the tally version's false
positive is gone — the surround line control returns to its pre-shock
band (117 vs 115) because a purely frontal fight now scores exactly
zero. ROUT keeps the earlier break (568 alive = 43% cas, was ~50%)
under true three-sided envelopment — band sign-off still pending.
Log line: "regiment N has enemies at its BACK".

Damage stays per-SOLDIER (each hit sector-tested against that man's
own yaw); only the fear term is regiment-level, as in M2TW.

## Round 4: no morale mechanism at all — the turn was the bug

Owner rejected round 3 too, on the deeper point: he hadn't asked for
morale changes at all — rear attacks should deplete units faster
ORGANICALLY, and the reason they didn't is that soldiers auto-turned
180° "in a few frames". Correct: the yaw update was exponential (a
third of the REMAINING angle per tick), so the larger the turn the
faster it completed — 180° in ~0.2 s, between two enemy swings.

Fix (a8f8674, aecda20): clamp the yaw step at pi rad/s (~1 s for an
about-face, what a burdened man in ranks can do; small corrections
never hit the cap, so marching and duel tracking are unchanged), make
the forward-half-plane attack gate unconditional (being struck tells a
man where to turn — it does not let him swing backward over his
shoulder; he answers once he has turned), and DELETE the rear-fear
morale term entirely (owner will rework morale later).

Measured (auto arena, identical 500-heavy duels): 74% of rear-lane
kills land in the back — men die mid-turn — the rear-attacked enemy
depletes at ~2x from first contact and breaks at ~52% casualties
purely through the casualty drain (t=55), rout slaughtered: attacker
ends 340/500 vs 19. Frontal control: near-mutual grind to t=100
(197 vs 130). Bands: ROUT back in its ORIGINAL ~50% band (487 alive
at break, fled 473) with the morale term gone; SURROUND line 127 (in
band), pocket t<=40; FORM all OK (facing err 0.01 — dressing is
unaffected by the cap); DIR rear bucket now dominates (654 rear vs
422 front kills) and per-hit gates hold (49.5/18.4).

The lesson of the whole entry: three cheap-awareness bugs (regiment
auto-face, proximity counter-windup, superhuman turn speed) each hid
behind the previous one, and the honest fix was deletion or a physical
limit every time — never a compensating mechanism.

## Round 5: blooded men stay on their foes (f3b7463)

Owner: soldiers who got hit shouldn't snap back to the ordered facing
while the enemy is still close — they should keep facing and fighting
unless the player explicitly reorders. Correct — between swing cycles
a man re-dressed to parade posture the moment his foe stood outside
the 2.6 m combat scan, and the attack gate then blinded him to the
same enemy re-closing. Rule now: a soldier whose combat memo holds a
living enemy within 6 m (KEEP_FACING_R, validated by team+distance
like the closing drive) keeps facing him; parade dressing and reforms
apply once the ground near him clears. FRESH men still hold the
ordered line — it is memory of a fight, not proximity awareness, so
the first-blood rear advantage is untouched (measured: arena rear
lane 342/500 vs 22, DIR rear bucket 610 vs front 462, per-hit
49.4/18.4; ROUT 499-alive break, SURROUND line 127 — all in band;
lone-pair pre-contact yaw dev fell 1.15 -> 0.53, the rim flip-flop is
gone).

## FL_ARENA — the hand-testing range (this entry's tool)

`FL_ARENA=1`: two mirrored duel lanes, identical 500-man heavy
regiments (FL_REG_SIZE overrides), enemies HOLD facing south, AI stood
down. West lane: your regiment stands in FRONT of its enemy — attack
head-on. East lane: your regiment starts BEHIND its enemy — attack
into its back. Same kind, size, and approach distance; the only
variable is the sector. `[arena]` logs every 4 s once blood flows:
per-lane strengths plus YOUR kills and damage-per-hit by sector
(front ~16/hit vs rear ~61/hit for heavies). R restarts the range and
resets the counters.
