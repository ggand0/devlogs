# 0036 — M2TW melee engine research (the real mechanics)

Date: 2026-07-18. Sources: M2TWEOP reverse-engineered engine structs
(EOP-Labs/M2TWEOP-library, read directly — struct-level ground truth),
Feral's official Rome Remastered engine docs (same engine family),
community mechanics research recovered from a deep-research pass
(twcenter, totalwar.org, heavengames). Facts labeled [ENGINE] are
struct/official-doc verified; [OBSERVED] are consistent community
observation.

## Per-soldier mechanics

- [ENGINE] Every soldier keeps his formation slot ALIVE through the
  whole battle: `soldierInBattle` stores `formationX/formationY` (his
  slot), `formationDisplacement` (how far he has strayed), and an
  `isInFormation` bit. The slot is never dropped during melee — it is
  the thing he returns to.
- [ENGINE] Melee is per-soldier, one target at a time: a single
  target pointer per soldier, and the VICTIM tracks `targetCount`
  (how many enemies currently target him).
- [ENGINE] The melee attack runs as an FSM action (`actAttackMelee`)
  that is crowd-aware: `crowded`, `isSideStepping`, `allowAttack`,
  `readyStance` bits, a `hintSoldier` pairing pointer, and an action
  context holding up to 100 nearby enemies AND 100 friendlies with
  per-soldier `collided` flags plus a `blockedCounter`. A soldier who
  cannot attack stands in ready stance, sidesteps, waits for room.
- [ENGINE] Crowding around a victim is gated by BODY OCCUPANCY, not a
  rule: each soldier has a hidden collision `radius` EDU param ("the
  area he occupies as the engine perceives it"); smaller radius =
  soldiers "fight more closely to each other" = more kills. Measured
  by modders: radius 0.4 -> 1.0 halved kills in identical fights. The
  attacker-per-victim limit IS the physical space around the victim.
- [ENGINE] Collision `mass`: "units with big mass can push their
  enemies harder and break through enemy lines easier" — melee
  contact is a physical pushing interaction. (We already have this.)
- [ENGINE] Rank matters per-soldier: the engine computes each
  soldier's ROW from the formation array and branches behavior on it
  (M2TWEOP reconstruction: rows 0-1 / row 2 / deeper get different
  brace-strike states; spear strikes gated by rank vs rank depth).
- [OBSERVED] Attack cadence is a per-soldier stochastic timer (EDU
  delay in deciseconds ± random jitter) — no synchronized rounds.

## Per-unit mechanics

- [ENGINE] The formation frame is destination + width-in-men + facing
  (the move-order signature). `frontRankSoldierCount` is tracked.
- [ENGINE] Pathfinding has `formation_hold_distance` (vanilla 25.0,
  "formations update after the last point") — the frame dissolves
  into per-soldier behavior near the destination, and the pathfinder
  is compute-once, not continuously replanned.
- [ENGINE] Engagement is MANY-TO-MANY and counted: each unit keeps an
  array of `engagedUnit` records — one per enemy unit in contact —
  with `engagedSoldiers` count, `spearPoints`, `engagedRatio`. An
  engagement exists when enough enemy soldiers are in a proximity
  zone (count + squared-distance threshold). Multiple friendly units
  CAN fight the same enemy simultaneously; there is no perimeter
  allocation anywhere.
- [ENGINE] Unit-level blocking exists: `isBlocked`, `blockedBy`,
  `forcedToBlock`, `waitingToMove`, and a `formationMovingThrough`
  formation state for deliberately passing through a friendly unit.
- [ENGINE] `reforming` is a discrete unit action state (alongside
  fighting/charging/pursuing/bracing/infighting): formation recovery
  is an explicit mode the unit enters, not a passive drift.
- [ENGINE] A "Melee Manager" subsystem outside the AI order hierarchy
  seizes control of units in close combat (unit-vs-unit fighting,
  flanking, retreat decisions).
- [ENGINE] Guard mode is a per-unit bitflag (with skirmish and
  fire-at-will); in the reconstructed brace logic it indexes the
  per-row brace table — it changes what each ROW of soldiers does.

## The behavior this produces ([OBSERVED], consistent across sources)

- Guard OFF (default): soldiers SURGE FORWARD OUT OF THE FORMATION
  FRAME to engage — individually, driven by target seeking. An
  attacking unit's soldiers do not stay on the frontage: they
  progressively WRAP ("flow") around the defender's sides; two equal
  units grinding frontally slowly rotate counterclockwise (each
  envelops the other's left) — envelopment is emergent per-soldier
  flow around occupied space, not assigned positions. Defenders
  individually turn to face non-frontal attackers.
- Guard ON: soldiers hold their slots — first line fights, rest hold
  position, no pursuit, no wrapping. Units in guard mode measurably
  ENGAGE FEWER ENEMIES and fight worse; players use it only for
  anvils/pikes. Guard mode is our H-hold, and it is the WORSE-combat
  stance by design.
- Soldiers with no reachable target: ready stance NEAR the crowd
  (crowd-gated by collision), sidestepping for room — pressed up, not
  parked at parade pitch.

## The historical smoking gun

Launch M2TW (v1.0) HAD the "only the first line fights" behavior:
once the front rank engaged, rear-rank soldiers held back instead of
pressing into contact (some even turned away) — IN CONTRAST to
RTW/MTW where soldiers press forward. The community called it broken;
diagnosis blamed new "anti-blobbing" code; the same era had charges
aborting if a single friendly soldier stood in the path. Patches
walked it back toward the RTW press. In other words: the exact
passive behavior the owner rejected three times was TRIED BY CA,
hated by everyone, and patched out. The M2TW everyone remembers is
the press + wrap + slot-memory behavior above.

## What this means for frontline (mapping, no design commitment)

The M2TW default melee = three per-soldier ingredients, all physical:
1. SURGE: every soldier of a fighting unit seeks a target and pushes
   toward the fight; the frame stops updating near contact
   (formation_hold_distance) — the men leave it and it follows.
2. OCCUPANCY: how many can fight is gated by body radius/collision —
   surplus men pack against the crowd in ready stance and sidestep
   for room; the wrap around the flanks is those men flowing where
   space is.
3. SLOT MEMORY: every man keeps his slot and displacement the whole
   time; the unit re-forms as a discrete state when it wins or
   disengages. Training scales slot tidiness.
Unit-level blocking/waiting exists in the engine but is for MOVEMENT
(march-through vs blocked), not a cap on who fights.

Our current gaps vs this model: (a) our crowd-jam zeroes a blocked
man's drive instead of leaving him pressing/sidestepping at the
crowd's edge; (b) our order goal translates the frame THROUGH the
enemy instead of freezing near contact and letting men surge; (c) we
have no re-form-as-a-state after winning (only event reforms).

Owner decides next step; no code from this entry.
