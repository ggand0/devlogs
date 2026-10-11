# Spearmen animation session

Written by GPT-6.

2026-10-11. This entry collects the standing, ready and subsequent spearman animation work from this session, as requested by Gota. Detailed standing measurements remain in devlog 0217. Update this entry as each next item is completed or reviewed.

## Standing

Gota accepted `assets_dev/spearman/stand_v1/` with “looks good”. The asset and review package were already committed as `1db4993` (Add the spearman standing pose). Record acceptance in the private iteration index without changing the accepted clip or scripts. Width 0.878436 m, forward extent 0.318006 m, grounded feet on all four levels, upright shaft with 16.412 mm butt clearance. The reference and surface limitations shown in the review remain documented.

## Current scope

Gota initially asked for ready and thrust in sequence, then corrected that instruction: ready only first, because the thrust depends on its review. Author and present ready, then stop. Do not start the thrust until ready is settled.

Shared rig and tools remain outside this task. Engine integration stays with Claude. Work continues in the graphics worktree root, with authoring and review scripts in each private iteration folder.

Standing acceptance is recorded in private asset commit `64e2bb6` (Mark the spearman standing pose accepted). The shared iteration index's concurrent knight row was preserved and excluded from that commit.

## Ready v1

Recorded the intended key pose before authoring in note 055, measured on armored sergeants and peasants `ready:0`. The reference holds its weapon nearly level (77.94/79.02 degrees from upright); the game needs a raised guard so the point stays below 0.7 m ahead. Authored a single original sample with pelvis/chest turned -35/-60 degrees, pelvis down 0.13 m, both feet planted, left foot forward and right foot back. Across/fore-aft ankle separation is 0.34/0.70 m. Spear pitch is 30 degrees from upright, yaw zero, with the grip fixed to the hand.

L0 width is 0.828534 m, forward extent 0.685612 m, rearward extent 0.719277 m. L1/L2/L3 widths are 0.827781/0.830637/0.852940 m. All soles are grounded to numerical rounding. Shared clip validation passes. Triangle counts remain 2992/664/250/56.

Equipment/body, spear/upper-arm-and-forearm, shield/body and spear/shield surface checks find no intersections at L0–L2. Whole-neighbor fore/aft checks find no intersections down to 0.9 m spacing at every level; lateral bounds leave at least 47.1 mm clearance at that spacing. The crouch does expose coat/leg intersections: L0 41 versus standing's 6, L1 42 versus 44, L2 16 versus 20. These are documented for review with positions and close views, not passed off as clean surfaces. The fixed shield mounting produces large reference differences; all eight out-of-tolerance rows are listed in the local comparison.

Separately tested 101 direct joint blends from accepted standing. They sink the feet up to 21.4 mm and widen the figure to 1.022 m; an endpoint check alone would miss both. Note 057 gives Claude the static pitch and the coat/transition limitations. A transition needs authored footwork after ready is settled. No thrust or transition clip was authored.

Ready v1 is archived in the private asset repository as `7afdb54` (Add the spearman ready pose), including its iteration row and LFS review media. The review includes matching 3.6 m seven-view sheets, both local reference units, body details, all levels, three formation spacings and a 640 × 640 orbit at 24 fps. Browser checks confirm that images load and the 24-second video plays through both camera directions and loops. Input hashes match, the clip validator passes, and the graphics worktree remains clean. Other sessions' files and the concurrent knight README row were left out of the commit.

The review stop is ready v1. Wait for Gota's judgment before authoring the thrust. Keep subsequent spearman work in this same devlog as requested.

## Ready v2: point toward the enemy

Gota found v1 close but requested the spear face the enemy as in `refs/spearmen/debug0.png`, with care over the wrist and hand. Recorded the plan in note 059 before editing. Preserved v1 and authored `ready_v2` with the same torso, shield and legs. The spear is 77.936704 degrees from upright, or 12.063296 degrees above horizontal, and yaw -3.229874 degrees. The direction matches armored sergeants `ready:0` within 0.01 degree; the weapon upper arm and forearm are 5.7/6.3 degrees from that reference.

The forearm rolls around its own length toward the shaft direction, then the wrist takes the shortest remaining turn. Wrist local rotation falls from 54.253 to 15.232 degrees. The weapon's local rotation is identity, and checking every spear vertex in hand space on all four levels proves its fixed rest transform is preserved to numerical precision. The review adds three wrist close-ups for each iteration.

L0 width is 0.889224 m and forward extent 0.979359 m. All levels remain below 0.9 m wide and have both soles grounded. The explicit request for the forward guard takes precedence over v1's raised guard: forward reach exceeds the earlier 0.7 m rule by about 0.28 m. That tradeoff is stated on the review page and in note 059. Same-facing neighbor surface tests still find no crossings at the four checked spacings; the narrowest lateral bounds clearance is 5.8 mm at 0.9 m spacing. Equipment crossings remain zero on L0–L2. The coat skin issue is unchanged.

The direct standing blend remains unsuitable, reaching 1.320 m width and 21.4 mm ground penetration. It is diagnostic only; no transition or thrust was authored. This iteration changes the requested weapon arm, hand and spear direction, and stops for ready review.

Ready v2 is archived as `acca7f4` (Aim the spearman ready guard forward). Clip, geometry, reference and fixed-grip checks complete; the isolated browser loads all images and plays the complete 24-second orbit through its reversal and loop. Review screenshots show the moving camera at three-second intervals. The shared rig and tools are untouched. The iteration index records v1's spear-direction revision and v2's review. Devlog remains uncommitted.

## Ready v3: rear-third grip

Gota found v2 usable, but marked a grip nearer the butt in `refs/spearmen/debug1.png`, and suggested a slightly lower arm. Measured the supplied model: its 2.50 m spear has 1.165 m behind the grip and 1.335 m ahead, placing the grip at 46.6% along the spear. Handoff 102's description of a rear-third grip was inaccurate. Note 068 records this finding and the plan before authoring.

Created an isolated model variant under `ready_v3`, moving only spear vertices +0.40 m along their rest axis on all four levels. The grip now has 0.765 m behind and 1.735 m ahead, 30.6% along the shaft from the butt. Skeleton nodes, pivots, bind matrices, weights, non-weapon geometry, other attributes and atlas bytes are preserved and compared against the source. The shared model and tools remain unchanged. This is a new fixed grip placement, not an animated slide.

Lowered the weapon hand 5.1 cm and kept the wrist at 15.23 degrees. The shaft retains v2's forward direction. L0 width is about 0.885 m and forward reach about 1.405 m. All levels remain narrower than 0.9 m and grounded. Equipment and same-facing neighbor checks pass; the original coat overlaps remain. The lower upper arm is 19.5 degrees from the reference and is recorded as an exception. The forearm still meets tolerance.

The model variant also affects standing: with the accepted standing clip, the spear butt rises from 0.016 m to 0.416 m and the tip from 2.516 m to 2.916 m. Added matching standing views so the permanent-grip tradeoff is reviewable. The JSON cannot encode the changed grip by itself; note 068 tells Claude how the isolated model variant was made and what the source model build must preserve if accepted. No transition or stab was authored.

Archived ready v3 as private asset commit `76ba30c` (Move the spearman grip toward the butt). GLB byte inspection, model preservation checks, clip validation, fixed-grip checks and review links pass. Browser playback runs through the 24-second orbit and loops; screenshots at three-second intervals show the full spear throughout the orbit. The engine loader takes its scale from trunk height (`Level::y_range` on the finest level), so the raised spear tip does not change the character's scale. The review stop is the rear-third grip and its standing consequence; shared model integration remains separate.

Gota approved ready v3 and the rear-third grip with “looks good, I'd approve this,” then requested the stab. The asset package remains `76ba30c`; the private iteration index records acceptance separately. Use `ready_v3/spearman.glb` and the exact saved `ready_v3/spearman.ready.json` as the stab's model and endpoints. Approval includes the presented standing consequence. Continue this session log for the stab.

## Thrust v1

Ready v3 acceptance is recorded in private asset commit `a9b23b4`. Planned the stab in note 069 before authoring, using four static `stab_mid` events (frames 0, 4, 9, 30) from both armored sergeants and peasants. The new clip uses the approved rear-third grip model and exact ready v3 endpoints. It has 29 samples at 30 Hz: contact at 0.333333 s, then 0.600 s recovery. This explicit sampling exception places the sim's timing on exact saved frames; no shared format/player change was needed.

The point lands at (-0.458383, 1.022428, 2.386115) m. The spear remains rigid in hand space across all four levels, with identity weapon local rotations. Wrist rotation peaks at 37.94 degrees. The upper arm and forearm rotate together through the return; the front foot supports the lunge while the rear foot turns about its toe. Attempts to force a straighter hand path produced elbow-plane discontinuities and excessive local joint speed, so the final clip retains the stable outward sweep.

The shared validator passes, including 872.42 degrees/s maximum saved joint speed. Dense runtime interpolation on all levels reports up to 1.570 mm ground penetration, 2.333 mm front sole drift and 3.734 mm rear toe-anchor drift. Width reaches 1.51 m during extension. Equipment self-crossings remain zero on L0–L2 at 201 sampled phases; coat and leg intersections remain, with coat crossings rising to 48/55/26 from ready's 41/42/16. Formation checks show intersections in some arrangements at every tested spacing, including 1.4 m; this is not a clearance pass for packed ranks.

The review includes full-speed oblique/gameplay videos, half-speed whole-body and wrist views, an orbit that reverses its camera while the clip keeps playing forward, seven-view sheets for all levels, and formation overlap examples. Both reference units have measurements and explicit exceptions for all four compared events. References and their derived files remain local and ignored. The model, rig, shared tools and game code are unchanged; integration remains separate. Note 069 carries the input model and timing contract for Claude.

Archived thrust v1 in private asset commit `eac2ff2` (Add the spearman skeleton thrust), including only its iteration folder and index row. All eleven review media files use LFS; reference-derived files, frame sequences and build output are excluded. Local links and source hashes pass. All five MP4s decode completely and advance through loops in the isolated browser. Inspected the seven-view key sheet, formation examples, timed browser wrist captures and orbit captures through both camera directions. This is offline motion inspection, not an in-game test. Other sessions' index rows and untracked folders remain untouched. Stop for Gota's stab review.

## Thrust v2: contact elbow and wrist

Gota accepted the overall motion but marked a strange elbow/wrist near maximum extension in `debug2.png`. Note 070 records the diagnosis and correction plan. V1's speed and wrist-angle checks passed, but its upper sleeve did not follow the elbow bend plane and the near-straight arm made the 37.94-degree wrist bend conspicuous.

V2 keeps v1's body, feet, shield, timing and exact ready endpoints. It changes the weapon arm only: more elbow bend, a forearm farther across the shaft, and an upper-sleeve roll aligned with the actual elbow bend at contact. The roll starts earlier than the wrist correction so the arm turns smoothly. An initial attempt to incorporate the changed roll into the whole arm spline changed the approach and exceeded the speed limit; it is excluded from the final clip.

The contact wrist is 13.63 degrees, with 21.27 degrees maximum through the full clip. Elbow direction change at contact rises from 32.59 to 56.39 degrees. Modeled sleeve/actual bend-plane error falls from 152.32 degrees to numerical rounding. These are geometric checks, not a claim about medical joint ranges. The point now lands 2.298710 m ahead, still within the 1.8–2.4 m brief. It is 1.023659 m high and 0.505405 m toward the weapon side.

Shared clip validation passes with maximum saved local joint speed 1,386.87 degrees/s. Dense equipment checks remain clear; unchanged foot and coat limitations are retained in the review. The fixed grip passes on every level. Neighbor tests are rerun and still show some intersections at all tested spacings. The new review leads with a synchronized before/after wrist video and contact image, followed by rebuilt motion and five-sided arm detail. All progress remains in this session devlog.

Archived the correction as private asset commit `10ed7d1` (Correct the spearman thrust elbow and wrist), including the v2 folder and only the two spearman index rows. Fourteen review media files use LFS. Complete video decoding, local links and source hashes pass. Browser time-update events confirm all six videos loop, including the orbit; inspected timed wrist playback captures, the matching contact pair and five-sided arm views. The first browser loop check sampled short clips too sparsely, so its loop observation uses actual time-update events. Shared code and other sessions' changes remain untouched. Stop for review of the corrected contact arm.

## Thrust v3: constrain the elbow itself

Gota rejected v2: the elbow still rotated in an impossible direction. That criticism was correct. The correction had aligned the sleeve while allowing independent arm rotations, and its check measured the contact silhouette rather than the elbow's degrees of freedom. Reinspected the peasants' stab sheet and the armored contact, then recorded a different authoring method in note 071.

V3 rebuilds the weapon arm using one fixed hinge calibrated to ready, a separate forearm roll and a wrist-flexion channel. Shoulder orientation aims the arm while maintaining the forward spear direction. The forearm cannot independently swing sideways or pass through straight into a backward bend. The body, legs, shield and timing retain v1, with exact ready endpoints. The arm's loading and withdrawal path changes to satisfy the hinge.

The new check exposes both rejected clips: v1 reaches -31.28 degrees of signed elbow bend at contact; v2 departs from the hinge plane by up to 66.48 degrees and has -135.69 degrees of forearm roll at contact. V3's bend stays between 20 and 100 degrees, with -60 to 0 degrees forearm roll relative to ready. Dense runtime interpolation departs from the hinge plane by at most 0.609 degrees. Contact places the elbow below the wrist, with 20 degrees of elbow bend and 35 degrees of wrist flexion. The prior goal of reducing wrist flexion did not excuse the wrong elbow movement.

The point lands 2.282163 m ahead and 1.128147 m high; width peaks around 1.38 m. The shared validator passes with maximum local joint speed 1,375.63 degrees/s. Equipment/body and spear/arm checks remain clear, and the fixed grip passes on all levels. The unchanged coat and foot limitations remain documented. Review media includes a v2/v3 close comparison and side, front, back and oblique arm views through contact and withdrawal, plus the full motion and reference comparisons.

Archived v3 as private asset commit `aa75754` (Constrain the spearman thrust elbow to a fixed hinge). The commit contains only the iteration and the two relevant index rows. Fourteen media files use LFS. All six videos decode and loop in the isolated browser; links and input hashes pass. Inspected the rendered five-sided contact/withdrawal sequence and timed wrist playback captures through repeated cycles. This is offline inspection; no game integration is claimed. Other sessions' files and index rows remain untouched. Stop for Gota's review.

## Thrust v4: forward arm direction

Gota rejected v3's outward arm in `debug3.png` and drew a forward, downward reach. V3 checked elbow bending and spear direction but missed the arm's horizontal direction. Its contact hand was 0.394 m to the weapon side of the shoulder and only 0.287 m ahead. Note 072 records the correction plan before authoring.

V4 retains the fixed hinge and directs both upper arm and forearm forward and down. Contact places the hand 0.050 m outward and 0.369 m ahead of the shoulder. Upper and forearm horizontal directions are 8.90 and 6.62 degrees from forward, compared with 55.80 and 51.80 degrees in v3. Added saved-runtime checks for those directions through the contact interval. Elbow bend stays between 20 and 100 degrees, with at most 0.154 degrees of departure from its hinge plane during interpolation. Forearm roll is 0 to 15.17 degrees relative to ready.

The fixed transverse grip requires 60 degrees of wrist rotation for this reach. The prior effort to keep that angle small contributed to the lateral arm. Contact now reaches downward, as drawn, with the point 2.360 m ahead and 0.758 m high. This differs from the higher forearm in M2TW contact frame 9 and is recorded in the reference exceptions. The approved model, grip, ready endpoints, body, shield and foot motion remain unchanged.

Clip validation passes with 982.99 degrees/s maximum saved local joint speed. Dense equipment checks remain clear, the grip remains fixed to numerical precision on all levels, and maximum width drops to 1.270 m. Coat/leg and formation overlaps remain; ground penetration and foot interpolation errors are unchanged. These limits remain in the review.

Archived v4 as private asset commit `14d4b27` (Aim the spearman thrust arm forward). All six videos decode and loop in the isolated browser. Inspected timed playback captures through extension, withdrawal and the ready return, plus front, side, top and oblique approach frames and the contact comparison. Review links and input hashes pass. Fourteen media files are tracked by LFS. Only the v4 folder and two spearman index rows were committed; other ongoing work remains untouched. Stop for review of the forward arm and its lower contact height.

## Thrust v5: align the hand with the forearm

Gota found v4 almost good but marked the outward wrist in `debug4.png`. The actual wrist-to-grip direction was 79.53 degrees from the forearm, even though the local wrist quaternion was only 60 degrees: the modeled palm already adds 19.53 degrees. Previous checks did not measure that visible alignment. Note 073 records the correction plan.

V5 keeps every v4 channel except the hand and weapon. At contact the wrist rotation cancels the modeled palm angle, placing the palm along the forearm. The spear uses the existing weapon joint to pivot at the grip, without any translation or slide. The hand-space grip pivot and the distance from grip to butt remain fixed. This replaces the overly strict identity-weapon constraint that had forced the wrist bend and earlier sideways-arm revisions. Added palm/forearm alignment checks through the contact interval, retained the signed hinge checks and verified unchanged upper/forearm rotations against v4.

The aligned hand brings the grip forward. Retaining the old shaft angle grazed the cuff and reached 2.437 m. Raising the shaft by 16 degrees cleared the cuff but dropped the butt into the coat; turning it 8 degrees toward the center clears both. Final contact is (0.028239, 1.264942, 2.365294) m. Dense equipment checks find zero crossings on L0 through L2. Existing coat/leg overlaps, foot interpolation errors and formation intersections remain documented. The model, rig and shared tools remain unchanged.

Archived v5 as private asset commit `210390e` (Align the spearman thrust hand with the forearm). Validation, palm alignment, hinge motion, unchanged arm channels, grip-pivot and spear-rigidity checks pass. All six MP4s decode and loop in the isolated browser; inspected timed wrist playback captures through repeated extension and recovery, five-sided close-ups and front/side/top approach frames. The reference comparison records the raised shaft as an explicit clearance exception. Fourteen media files use LFS. Only v5 and the two spearman index rows were committed; other ongoing files remain untouched. Stop for wrist/hand review at `assets_dev/spearman/thrust_v5/review.html`.
