# M2TW archer orders from the decompile: fire at will, the attack order, skirmish

Written by Claude Fable 5.1. 2026-10-18, on main 8810113, before the archer branch. Research only, no code. Gota's list for the branch is in plan 026; Gota asked for the M2TW behaviour on the fire topics first.

Sources: `../m2tw-decompile/docs/BATTLE.md` (the task list, the idle task, the engage task, the volley, the skirmish task, the missile action), the verified reimplementations in `../m2tw-decompile/reimpl/src/` (`battle_unit_react.c`, `battle_unit_auto.c`, `battle_task_idle.c`, `battle_task_skirmish.c`, `battle_task_beneath.c`) and, where the notes stop, the lifted code in `../m2tw-decompile/decomp/src/` and `decomp/by_class/UNIT.cpp`. Function addresses are given so the reading can be checked.

## 1. How a missile unit decides to shoot on its own

A unit that is not busy runs an **engage check** every tick (`Unit_engageCheck` 0x007472a0). It tries three automatic reactions in this order and takes the first that applies:

| Reaction | Target | What starts | Order queue |
|---|---|---|---|
| 0 | an enemy this unit is in melee contact with that is not its ordered target (0x0073b600) | `UNIT::vf25`: ATTACK_ENGAGE on it | emptied |
| 1 | the skirmish trigger's enemy (0x00742a70, below) | `UNIT::vf23`: the SKIRMISH task, record +0xbf0 set | kept |
| 2 | the fire-at-will target (0x00742a00, below), only when that enemy would not reach the unit within about 10 s | `UNIT::vf24`: MISSILE_FIRE_VOLLEY at it (FIRE_BY_RANK for gunpowder) | emptied |

Each reaction needs its enabling byte (+0x208a plus its number) and is not repeated against the same target while held. Skirmish comes before the volley, so a unit in skirmish mode runs rather than shoots when the trigger holds, and the order it was carrying survives the run.

**The 10 second rule** (reaction 2): with d the vector from the enemy to the unit and v the enemy's velocity relative to the unit, no volley starts when 10 (d · v) > 0 and |d|² ≤ 10 (d · v), that is when the enemy closes and would arrive within 10 s. The purpose is not visible in the code; with skirmish off the unit does not start a volley at a charge that is 10 s out. Not copied into flanks (plan 026 says why).

**Fire-at-will target** (`FUN_0073a530`, called by 0x00742a00 with the unit's own position):

- candidates: every enemy unit on the alliance's list that is not routing (states 0x12 to 0x15), has men, and passes the unit's **can-shoot test** at its missile range (`UNIT::vf175`, below);
- **excluded: an enemy whose own record has +8 set**, which the notes read as "in melee contact" (the pursuit code: "the unit is in contact when its record's +8 is set"). The writer of +8 is not traced here; `Unit_setStateFighting` 0x0073e7b0 sets the unit's state 0xb from "any soldier in melee", not this flag;
- score = −distance, +1000 for the unit's ordered attack target, +500 for an enemy with siege engines when both units pass slot 204, −500 for an enemy in a withdrawing state (0x007356e0: states 0xd to 0x10, 0x17);
- the best score wins: the nearest in-range enemy not in melee, preferring the ordered one.

So in the decompile, fire at will does not shoot at an enemy unit that is fighting in melee. Gota's memory is that archers in M2TW shoot the men of an engaged unit who stand away from the fight. The two readings can both be right only if +8 means something narrower than "any man in melee"; ways to settle it: trace the writer of record +8 in the lifted code, or run the installed M2TW with archers on fire at will behind an engaged line and watch whether they loose. The explicit attack order is different: it shoots at its target whatever the target is doing (section 2).

**Gates** on the whole thing: `Unit_fireAtWill` 0x00756d50 (the order byte +0x1e84 on, a shooting type, not in states 6, 7, 0xa, 0xc, 0xd, 0xf) lets an idle unit stand ready; reaction 2 also needs the unit's "may volley" flag (+0x1e44 bit 1), refreshed each tick from: missile type (slot 159), ammunition left (slot 173), may fire (slot 180).

## 2. The attack order on a missile unit

The ATTACK_ENGAGE task's state 0 (`Engage_next0` 0x00777940) for a missile unit:

- out of ammunition: the order ends;
- **can shoot at the enemy from where it stands** (`Unit_x734670` / `Unit_x734610`, the can-shoot test at the unit's range): state 10, **shoot where it stands**, a volley task on the target; the volley's READY state turns the unit only when the target lies more than 2° off its heading;
- otherwise, if it may charge: state 9, **close in: a move order at 0.8 of the missile range**, then back to state 0 and the test again.

**The can-shoot test** (`UNIT::vf175` 0x0073be90, `(target, range, flag)`): true when the two units' footprints (the record's shape at +0xb0, not the centres) are within

    range + 3 s × (the target's speed toward the unit) + (the unit's height above the target)

of each other, and either both stand in the same battle area or the line-of-fire check `FUN_007cf6c0` passes. The lead term lets a unit open fire on an enemy that will be in range by the time the shafts land; the height term is the high-ground bonus at the unit level.

**Per man** (`MissileAct_ensureTarget` 0x007cf460, `MissileAct_loose` 0x007d07f0): each soldier is paired with a random man of the target unit and keeps him while he fights; he shoots only when that man is within the unit's range stretched by `unit/max-missile-range-extension` (20 percent, so 1.2 × range), and only with a solution. Before the flat shot he checks the line of fire; if something stands in the way within 15 m of the aim point he gives the flat shot up and retries lobbed, and the other way round. No rank limit, no friend-avoidance beyond that trajectory check.

## 3. Skirmish

**The trigger** (`FUN_0073a740`, the target of 0x00742a70; also the condition that keeps the SKIRMISH task alive, `TaskSkirmish_beneath` 0x00778ba0). Seven preferences per class, read at start-up (`text_00750000.c` 0x0075xxxx) with these defaults for infantry (gunpowder and cavalry have their own sets):

| Preference | Default | Used as |
|---|---|---|
| `unit/skirmish/infantry/default/max-range-scale` | 1.5 | enemies considered: within 1.5 × range |
| `.../min-range` | 15 m | an enemy footprint within 15 m of ours: run |
| `.../min-stopping-range` | 50 m | our own path ahead, min(range − 3, 50 m): an enemy footprint there: stop and shoot |
| `.../collision-reaction-time` | 8 s | the enemy's path ahead = its speed × 8 s |
| `.../retreat-time` | 12 s | added to the 8 s while already skirmishing (20 s): the hysteresis |
| `.../range-factor/moving` | 1.1 | cap on the enemy's path ahead while we move: 1.1 × range |
| `.../range-factor/shooting` | 0.8 | the same cap while we stand: 0.8 × range |

For each enemy within 1.5 × range (not routing, not withdrawing, with men, in the same battle area):

1. footprint within 15 m of ours: **run** (target set, the task runs);
2. the enemy has a direction (its move order's point, its attack target, or its velocity): project its footprint forward by min(distance to us, its speed (slot 138) × 8 s, range × 0.8 or 1.1) and test three points of that strip against our footprint. If they hit, look for a **screen**: a friendly unit that is fighting, not itself skirmishing (its +0x1e44 bit 2 clear), within the same distance, whose footprint covers those points. Unscreened hit: **run**;
3. we have a direction of our own: project our footprint ahead by min(range − 3, 50 m); if it hits the enemy's footprint and nothing else triggered: **hold**, that is stop and shoot (the record's hold byte +0xbf3 = 1; the task then fires a volley at the record's target instead of moving).

**The point** (`FUN_00793430`, computed every tick of the task): 32 headings are tried; for each, the squared angular difference to the bearing from every fighting enemy unit on the field to us is summed (every enemy counts equally, distance plays no part); the heading with the smallest sum, that is the direction most nearly away from all of them, wins. The distance is the missile range (150 m for a unit without one), times (1 + min(1, the nearest enemy's speed / our speed)) when that enemy is moving: up to twice the range from an enemy as fast as we are. The point is kept inside the map; with no valid point there is no skirmish.

**The task** (`TaskSkirmish_update` 0x0077b0b0): hold set → volley at the record's target or halt; otherwise an INTERCEPT_SKIRMISH move to the point, facing the enemy with the special ability, else plainly ordered away. It ends when the trigger no longer returns an enemy (the enemy footprint beyond 15 m and no path hit with the 20 s look-ahead), and the order underneath resumes.

The manual's one line ("keep a safe distance, usually its missile range") matches: the retreat goes a full range, the trigger is the enemy's approach, not a fixed gap.

## 4. Melee

Peasant Archers (EDU): `stat_sec 2, 1, no, 0, 0, melee, melee_simple, piercing, knife, 25, 1.8`: a knife at attack 2. The `MTW2_Bowman` skeleton lists only the missile attack clips and borrows the knifeman's swim set; the melee moves come with the secondary weapon's skeleton. For an archer's melee reference our rendered `refs/animation/m2tw_model_refs/hand_gunners/` is the closest set: sword, no shield, the `MTW2_Non_Shield` moveset with the knifeman's ready, walk, run, charge and parries.

The unit controls (manual, key list): fire at will I, skirmish K, guard mode B, run H, special ability J, withdraw L. The key list we hold omits the weapon toggle; Gota: the melee toggle is F (2026-10-18). A first reading of this devlog said there was none, which was wrong.

## 5. Open in this reading

- The writer of the enemy record's +8 (what exactly "in contact" means for the fire-at-will exclusion).
- Slot 138: the notes call it the walking speed in one place; in the skirmish point it is chosen against slot 137 by the unit's running flag.
- Why reaction 2 refuses a target 10 s out; whether a soldier's own ready action ever shoots without a volley task (nothing found that says so).
- The volley task's +0x20 flag, read as "firing at will" in the notes, is set by the gunpowder reload step; it may mean "reloading".
