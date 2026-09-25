# 0122 — Contact frame, stepping, gap fill

Written by Claude Opus 5.5. Date: 2026-09-25. Branch `feat/melee-footwork` (off main cc7b08b). Builds the design Gota approved in 0121, minus the deferred crowding counter and sidestep.

## Commits

- fd8834e: `FL_DIAG_REAR` (rear-rank motion by rank, order type and driving term, plus casualty totals) and `FL_AUTOSTART` (a normal AI battle with no key press; with `FL_DEPLOY=0` it skips the picker and deployment). Sim fingerprints unchanged.
- 2750700: contact frame, step pace, the two clear-path waits, facing kept while stepping.
- 0eddd31: losses replaced from the rank behind.

## What changed

- Contact frame (frontline.rs `update_groups`): an attacking, formed, unbroken regiment enters contact once 3% of its men (4 minimum) are in a melee wind-up, against any enemy. Its anchor then holds laterally where the block stood (`contact_lateral`), and in depth puts its front-most slot on the fight line. The fight line is the mean position of the men swinging at an enemy ahead (within 60° of the frame's facing). The depth moves only when the line has moved a full rank pitch. `prepare_tick` resolves a contact regiment's order to None, so it holds like a standing regiment. It releases when engagement ends, and the attack order resumes. FL_RECTFIGHT keeps its own freeze; the frame only runs with it off.
- Step pace (movement.rs): a holding man outside the 0.7 m deadzone walks at 0.8 m/s at least (`STEP_PACE`, M2TW's shuffle). Once moving he finishes to 0.35 m (`STEP_STOP`) instead of stalling at the deadzone edge. Below 1.05 m/s he keeps his facing (`STEP_FACE_SPEED`) instead of turning to his velocity.
- Clear-path waits (movement.rs): the surge toward a remembered enemy is dropped when a comrade's body stands within 1.4 m inside a 45° cone of that direction. In an engaged regiment, a holding man's step back to his mark is dropped the same way. The slot wait is limited to engaged regiments: applied to a dressing block it held men off their marks and doubled the FORM slot error (1.3 to 1.5 m).
- Gap fill (combat.rs `fill_vacated_slots`): each slot a dead man of a formed regiment leaves goes to the nearest living man behind it in the same file (within half a file pitch) who is in Ready state. Front slots go first, one slot per man per tick. Blob regiments are skipped.

## Measured (8 v 8 of 500-man regiments, `FL_AUTOSTART=1 FL_DEPLOY=0 FL_UNITS=4000 FL_REG_SIZE=500 FL_HEAVY_FRAC=0.4 FL_DIAG_REAR=1`)

| | main (fd8834e) | branch (0eddd31) |
|---|---|---|
| idle men, ranks 2+, share moving | 60 to 97% | 20 to 28% |
| idle men, ranks 2+, mean speed | 0.1 to 0.4 m/s | about 0.2 m/s |
| lost per side at 30 s | about 1,300 | about 650 to 700 |
| lost per side at 60 s | about 2,600 | about 1,400 to 1,600 |

The frame alone did most of it (ranks 2+ at 29% moving; kill pace halved). The surge gate took ranks 1 and 2 off their comrades' backs. Idle front men (ranks 0 and 1) still move most ticks, at a real walk (1.4 m/s mean): 50 to 76% of it is body contact at the fight's edge, with direction reversals at 1 to 2%. That is the press, not a creep.

Runs of a normal battle vary with AI timing (AI and auto-engage run in Update), so compare trends, not digits.

## Battery

- FORM: slot error 0.57 to 0.74 m, facing 0.00, spacing wall 1.05 < normal 1.37 < loose 1.88. OK.
- DIR: dmg/hit front 21.8 / side 28.1 / rear 49.2 (2.26x), rear kills dominant, yaw dev 0.00.
- CHARGE: wall dz -1.8 m vs open -3.3 m; the wall lane trades evenly (354 v 360), the open lane loses (300 v 438).
- Clippy clean. Fingerprints change by design (DIR diverges from tick 1020). New baselines: `tmp/hash-baselines/{dir,arch}-footwork-0eddd31`, deterministic on rerun (19/19). The refactor at the end of the branch must match these.

## Open

- Pace is about half of main's. The contact frame stops the two blocks from pushing into each other, so fewer men are in reach at once. Gota's feel call; the levers are the engage and step rules, or the deferred sidestep, which is what lets M2TW's surplus men flow around to the flanks.
- Sideways steps still play the forward walk (facing is kept, so it reads as skating) until Astra's shuffle tables land (`tmp/handoffs/HANDOFF-side-shuffle-2026-09-25.md`).
- Not yet run: ROUT, SURROUND, PILE, JOIN, 200k perf.
- The movement.rs refactor, last on this branch.

## Round 2: Gota's play test (e5214c6)

Gota, after playing 0eddd31 (screenshot resources/unit_movements_debug3_092426.png): after engaging both units swing back and forth noticeably; as the battle goes on big gaps open between the fighting front and the rear ranks; sometimes a unit starts walking backward right after engaging.

Causes and fixes:

- Gaps: the gap fill moved only the one man behind a dead man, leaving his own place empty, so the holes piled up right behind the front. Now the whole file behind the hole closes up one place and the holes collect at the back. Depth profile after (men per file per rank-deep band behind the front): 1.6 to 1.9, 0.70 to 0.84, 0.77 to 0.84, then about 1.0. The front band is dense because the second rank closes up into the fight; no hollow.
- Back and forth: the frame followed the fight line both ways in whole ranks. It now only moves forward.
- Walking back at contact: a charge bunches the rear ranks behind the stopped front, and the frame's full spacing sent them walking back. Contact-frame regiments no longer step backward to dress (forward and sideways only). Idle rear men walking backward in the seconds after contact: 23 to 25% before, 6 to 7% after (the rest is impact recoil).
- First cut applied the no-backward rule to every engaged regiment, measured against the regiment's facing. A regiment struck from behind then stopped stepping back toward its attackers to hold its ground, and DIR's lone rear-charged victim survived with 215 of 500. Restricted to contact-frame regiments: lone 43 of 500 (0 at 0eddd31), rear dmg/hit 49.4 vs front 21.9, rear kills 736.

Battery at e5214c6: FORM OK (slot err 0.57 to 0.74, wall 1.05 < normal 1.37 < loose 1.87); DIR as above; CHARGE wall lane 338 spears v 292 heavies, open lane 294 v 401 (ordering intact; the open lane's dz now reads +3.1 m because its files close up forward as its front dies, so its center moves up). Pace in the 8 v 8: about 875 lost per side at 34 s (main about 1,300 at 30 s). Baselines: tmp/hash-baselines/{dir,arch}-footwork-e5214c6. Backup of the pre-fix working tree: refs/backup/footwork-fix2-wip and tmp/backups/footwork-fix2-wip-2026-09-25.patch.
