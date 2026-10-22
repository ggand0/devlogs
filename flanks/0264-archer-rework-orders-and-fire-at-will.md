# Archer rework: the range test, the attack order, fire at will per man

Written by Claude Fable 5.1. Branch `feat/archer-rework`, cut from main 8810113 on 2026-10-18. Plan 026 (approved the same day), research devlog 0263. The build log of the branch; the numbers are from the scripted archery scene.

## Stage 1: the edge gap, the stand-off, the man each archer shoots (2026-10-18)

### What changed

- **One range test**, `in_bow_range` in `sim/job.rs`: the gap between the two blocks' edges along the line between their centres (the rectangles, `formation::block_reach_along`, not the circles) is within 120 m plus 3 s of the target's speed toward the shooters plus the shooters' height above the target. Replaces the two centre tests (0.85 of the range for the attack order, the range for fire at will).
- **The attack order** keeps a stand-off state on the regiment (`GroupData::standoff`, `standoff_target`): a fresh order shoots where it stands when the test passes, with no facing change and no reform (`attack_regiments` skips both for archers); out of range it faces the target, dresses once and marches until the gap is 96 m (`CLOSE_IN_GAP`), then stands; it marches again only when the gap passes 120 m.
- **Each archer shoots a man of his own** (`Units::quarry`, a new column): picked as the draw starts (`pick_quarry`), kept through the draw, shot at the loose while he lives; a third of the men keep theirs between shots (`KEEP_QUARRY_SHARE`), the rest pick afresh. A man is in reach within 144 m (`missile::REACH`). A man without a pick looks again every 8 ticks, three random members a look.
- **Fire at will** (`archer_fire_solutions`): the regiment's target is the nearest block in range that is not in melee; when every block in range is in melee, or a block is coming for the regiment within 8 s (`approach_hits`, the path test of plan 026 §4 with a friendly block as a screen), the men **search** the blocks in melee for **clear men** (`ShootAt::search`, `search_blocks`). The attack order on a block in melee searches the same way.
- **Clear marks**: every man notes an enemy in his reach each tick (`SIGHT_FOE`); a man of a searched block looks every eighth tick for an enemy within a shaft's reach of him (`SIGHT_CLEAR`: the aim scatter box's corner, 7.4 m, plus the strike band's run, 2.0 m, so 9.4 m) and drops the mark when nobody searches. The job lists the clear men of the searched blocks beside the living members (`clear_members`), so an archer's pick is one lookup and one distance, no grid query of his own. The card's firing mark needs clear men in a search.
- The archery scene gained `FL_ARCHERY_ORDER=1` (both regiments ordered at the east flank block) and `FL_ARCHERY_MELEE=1` (the static blocks beyond range, a spear block attacking the end of the screen: the picture of plan 026 §3), and its log prints the firing and stand-off flags, the men drawing and the men struck per side.

### What the scenes showed (opt-dev, 500 men a regiment, 60 s each)

| Scene | Result |
|---|---|
| base (fire at will at the middle block) | as before: both regiments volley 500 shafts a cycle, 0 blue struck |
| order (both ordered at the east flank block, gap about 115 m) | stand-off at t=0 for both (flags `---OFH`), no march, no re-dress; the east regiment volleys 500 a cycle; the west one, further off, 36 to 167 men a volley (the rest beyond 144 m of their man); it fell out of range at t=44 as the dying block shrank, marched, stood again at t=56 |
| melee, first run (clear radius 5.25 m) | the men search and shoot the spear block's open side: 123 blue struck of 4510 by t=56 |
| melee, second run (clear radius 9.4 m) | 25 blue struck by t=8 (the volleys loosed as the block made contact), none more through t=36 over four volleys, then 85 by t=56 as the block melted from 500 to 164 men and the screen pushed into where the shafts land; 3966 orange struck |

The old skirmish rule ran in the melee scene (the spear block passed within 22 m of the east archers on its way to the screen; both regiments ran, and the `skirmishing` flag stayed set after the move arrived, the stale flag plan 026 §4 names). Stage 2 replaces it.

### Checks

- opt-dev build, strict clippy and the 50 tests pass.
- `FL_TEST_DIR` fingerprint on the branch (45 s): 21 fingerprints, last `1260 3222 99dc6fe27e005c91`. Main's binary (a clone of 8810113 in `tmp/runs/archer-rework/main-src`) was building when this was written; the comparison is in stage 2's section.

## Stage 2: the skirmish run as a reaction over the order (2026-10-18)

### What changed

- The skirmish rule left `arrows.rs` (which keeps the ammo tally, `ammo_tally`) for the tick's prep, `skirmish_reactions` in `sim/job.rs`, run before the fire solutions. Three triggers over the enemy blocks within 1.5 ranges: an enemy edge within 15 m (`SKIRMISH_NEAR`); a block coming for the regiment within 8 s (`approach_hits`, 20 s while the regiment already runs: the hysteresis); enemy men already in reach of the regiment's own men while no real melee has begun (the block's leading men on it). A regiment walking into an enemy block within 50 m of its own path stands and shoots instead (`skirmish_hold`). Hold position, skirmish off and a regiment in a real melee (`melee_ticks > 0`) never run.
- The run is a Move order with the regiment's own order parked in `skirmish_saved` and handed back when the trigger lapses (`end_skirmish`); an order anyone writes during the run (the player, the AI, the run's own arrival) is parked the same way. The retreat point (`retreat_point`): a bow's range times (1 + the nearest enemy's speed over the regiment's, at most 1), in the one of 32 headings most nearly away from every unbroken enemy block, clamped to the map and refreshed every 15 ticks. No room to run (the map's edge): the regiment is cornered and stands and shoots until room appears or the trigger lapses.
- A friendly block screens the regiment from an approaching enemy only when it covers the enemy's path centre line or both of the path's sides; a block touching one side is passed by, as the enemy's men stream round it.
- A block coming for the regiment preempts its attack order: its men shoot that block whatever they were ordered at. A regiment with a few enemy men in reach keeps shooting (the gate is the melee clock, not the engaged flag) and may still run.
- The archery scene's attacker now comes from the west flank with nothing in its way (`FL_ARCHERY_ATTACK=1`); `FL_ARCHERY_NOSKIRMISH=1` turns skirmish off for the archers; the log gained the hold flag.

### What the scenes showed (opt-dev, 500 men a regiment)

| Scene | Result |
|---|---|
| attacker, skirmish on | the west regiment ran at t=1 with the attacker 116 m off (8 s of its approach plus the two blocks' reach), kept 75 m ahead of it for 30 s, reached the map's corner, was cornered and stood: one volley of about 500 at the attacker at 45 m (orange struck 1014 to 1480), then the melee |
| attacker, skirmish off | the archers kept volleying the middle block until the attacker was 52 m off, then turned on it: a volley at 21 m as it crashed in (62 blue struck of about 1400 shafts, the men running under their own shafts), then the melee |
| attacker, the regiment under an attack order at the far flank block | the run parked the order and gave it back at the corner (flags `---OF-h`); the cornered regiment ignored the far target and shot the attacker at 45 m with 429 men drawing |
| attacker, first try | the attacker came from the north through the screen's end: the screen counted as a cover, the attacker's leading men engaged the archers, and the engaged flag forbade the run, so 500 archers died with full quivers. That is what the screen rule and the melee-clock gate fix |

The runs end at the map's corner in this scene because the attacker is as fast as the archers and the scene's map is small; on a battle map the run goes a full range and the trigger lapses when the attacker falls 20 s behind. Gota judges the cycle in play.

### Checks

- opt-dev build, strict clippy and the 50 tests pass.
- `FL_TEST_DIR` fingerprint against main's binary (8810113, built from a clone in `tmp/runs/archer-rework/main-src`): identical after stage 1 and after stage 2 (21 fingerprints over 45 s, last `1260 3222 99dc6fe27e005c91`): the melee kinds are untouched.

## Stage 3: swords out (2026-10-18)

### What changed

- `GroupData::melee`, the swords-out mode of an archer regiment: no shooting (`shoot_at` is None), an attack order is a charge like any infantry's (no stand-off in `archer_fire_solutions`, `attack_regiments` turns and dresses it, the charge phase in `frontline.rs` no longer excepts it), the skirmish run is off, and `ai.rs` engages it at ease as infantry.
- `FormCmd::Melee`. The F key stays `FormCmd::Wall`, which now does the wall on the swordsmen and spearmen in the selection and the swords on its archers, who form no wall. The card gets a sword button ("Swords Out (F)", `melee_shapes`), lit when every archer picked has its swords out; the wall button reads the other kinds only.
- `FL_ARCHERY_SWORDS=1` in the archery scene: both archer regiments in the mode with an attack order on the middle block. The log gained the M (swords) and C (charging) flags.

### What the scene showed

Both regiments marched with no stand-off, charged at about 10 s, engaged and crashed into the block as infantry does, with full quivers (flags `E--O---M-`, no F, no H).

### Not in this stage

The sword on screen (plan 026 §5: the sidearm in the draw hand, the bow slung, the per-man draw at about 10 m from an enemy block's edge) waits for Astra's model with the sword, the scabbard and the bow's stow transform (handoff 138). The sim side of the per-man rule needs a render-visible bit, which the packed per-soldier record has no room for today; it comes with the model.

## The rest of the thread (2026-10-18 to 2026-10-22)

- **A 200k measurement started too early.** After stage 3 a flank-view comparison against main (binaries `tmp/runs/perf4/bin/flanks-mainarch` from a clone of 8810113 in `tmp/runs/archer-rework/main-src`, `flanks-archer3`) was started and Gota stopped it: the model and the clips are what costs on the GPU, so the one measurement is at the branch's end with them in. Nothing from that run is recorded.
- **The archers' knife references** (2026-10-21). The reference game's bow units carry a second skeleton in the model list for the sidearm, the knifeman's, and the archer model carries knife groups the bow renders never showed. `assets_dev/_m2tw_refs` (commit 416b36c) gained the unit `peasant_archers_melee` (the knife groups on `MTW2_Fast_Knifeman`, the bow left out as the game hides it, the quiver on the back), a unit entry that may name its model list entry, and the knife's own blows, punch, hook, kick, charge blow and defends in the clip index. 44 clips rendered to `refs/animation/m2tw_model_refs/peasant_archers_melee/` (frames in `tmp/runs/m2tw_archer_melee`): stand, ready, the steps, the shuffles, walk, run, charge, advance, retreat, combat jog, the starts and turns, three blows (two cuts and a stab, the right-to-left cut with its blow at 0.90 s), a punch, a hook, a kick, the charge blow, nine defends by height and blow, and the fails of the cut and the stab. Looked at: the ready is a low knife crouch; the right-to-left cut looks the same as the knight's (different file, the makers most likely reused the motion; `pose_check.py` can tell in numbers).
- **Gota's decisions**: no sword-draw clip (the game swaps weapons in one frame; the knife appears and the bow goes away at once); the knight's melee clips and shuffles are to be reused on the archer rather than authored (the body joints match by name: 17 of the knight's 19 are on the archer's 22; the archer lacks `weapon` and `shield`, and has the arrow and the bow's limbs and strings besides; the loader refuses a clip whose joint list is not the model's, so the reuse is a loader change, mapping tracks by joint name and leaving unnamed joints at rest).
- **The approved model arrived** (handoff 140, 2026-10-22): the hand-built archer with the new body and arms, the hood and scarf, the straight back quiver without a harness, the dagger in the right fist and two single-sample melee poses (`archer.melee_stand.json`, `archer.melee_ready.json`), on the same 22 joints, with no `weapon` joint, no sword part of its own (the dagger is in the hand part) and no bow-stow transform. The integration is handoff 141.
- Handoff 139 (a generated body route) was withdrawn by Gota (memory `ai-model-route`).
