# One knight model and repaired block

Written by Codex (configured model: GPT-6 Astra).

2026-10-13. Gota required one model for the existing animations and block/parry work, following the block_v1 rejection and attachment diagnosis in devlog 0243. Work is in assets_dev/knight/block_v2. Shipped assets and rejected block_v1 remain untouched; every v1 package checksum matches.

## Common model

compatible/knight.glb retains 19 joint names and their parent order. The existing shield joint receives a pivot at (0.3737520201, 1.0967340515, 0.1869226020) m and a constant mounting rotation in every compatible clip. Other rest joints remain unchanged. The hand stays neutral relative to the forearm, with the board and straps fitted across it. No extra joint is needed.

The initial remount changed the rest board orientation too much for rigid-part playback. The final binding encodes the rotation in the shield joint and retains the original rest board orientation, with a common translation of approximately (0.049216, 0.103042, 0.000881) m. This 11.42 cm translation changes fallback appearance; it is not an unchanged-fallback claim. Moving the correction into the joint reproduced sampled posed vertices within 0.000092 mm. The far and fallback animations still need in-game visual review.

Straps are refitted around the hand and proximal forearm. L0 gains three sleeve cuts at the elbow, L1 one. Counts are 2,976 / 714 / 250 / 52 triangles. Shield-arm surfaces on L0–L2 have zero open, nonmanifold or inconsistent-winding edges and zero degenerate triangles. Other boundary counts match the original model, including its existing concealed body and leg boundaries. GLB byte inspection verifies exported channels and pivots.

## Existing animations

Ten compatible clips sit beside that single model: stand, ready, charge, charge_attack, cut, stab and the four shuffles. Their unrelated joint channels and root samples are exactly preserved, as are timing, travel and metadata. Left-arm placement is reconstructed with the common mount, an elbow hinge, and forearm rotation about its length. Standing uses a carry pose on the same model. Ready has a relaxed board tilt so block can visibly square the shield. Shared ready endpoints remain exact within the package.

All ten pass the shared validator. Shield/body and shield/sword checks sample runtime interpolation at 40 Hz on L0–L2, with no crossings in the final candidate. Dimensions and ground minima use 100 Hz on all levels. Standing/ready, standing/charge and charge/ready blends were checked at 41 amounts on L0–L2, also clear. These are sampled offline checks, not full engine acceptance.

## Repaired block

The block uses the common model and ready: 19 samples at 20 Hz, zero travel, non-looping, guard at 0.20 s, release at 0.70 s, end at 0.90 s. It braces through the crouch and holds the sword back. The left elbow bends through its hinge instead of independently aiming the forearm to orient the shield.

The main reference is dismounted_feudal_knights/defend_mid_stab frame 4; swordsmen_militia/defend_mid_stab is checked at that guard event too. Both meet all 18 workflow tolerances. Angular RMS is 2.7 degrees for knight and 3.5 for militia; worst angle is 9.5 degrees. Shield-face error is 1.2 degrees. Reference material stays in ignored local_reference/. Curves are authored independently.

Both endpoints exactly equal common ready. Left elbow flexion is 88.65–110.40 degrees; wrist bend is zero. Across 181 interpolated poses, elbow-plane deviation peaks at 0.0242 degrees, about 0.104 mm at the wrist from quaternion interpolation. The physical check permits 0.25 mm; authored poses use the analytic hinge. Heel-pivot drift is 4.051 mm and interpolated ground penetration approximately 4.026 mm; saved samples lie essentially on the ground. Blade/arm and weapon/body checks are clear on L0–L2 at 91 phases. Maximum L0 block width is 0.92111 m.

## Review and integration

review.html shows all eleven animations on one model, seven-view sheets, block at full and half speed, and a 24-second two-direction orbit. Rendering uses the game's CPU skin. Existing movement previews are treadmill clips with original travel preserved in JSON. All thirteen videos decode fully and complete browser playback, including wraparound; captures and playback counters are in review_checks.json and frames_compatible/browser/. Ordered saved-motion and browser playback captures are inspected. This is not an in-game viewing claim.

The model and compatible JSON clips must be adopted together after review. Mixing the new model with old clips changes the mount. compatible/knight.guard_diagnostic.json is an inspection pose, not a clip to integrate. The loader reads clips beside the overridden model, so FL_GLB_KNIGHT can review the whole candidate without replacing shipped files. No src code changed and no game was launched.

Parry is the next separately reviewed item and must use this same model and ready if accepted. No commit or push. The README index marks block_v2 for review. Package hashes are in delivery_checksums.json. Model SHA-256: b453cc79a58bb6b9c879e70208db83b6309311482dd2d9c933d00d3c4d0e85c4. The local review server is http://127.0.0.1:8766/review.html.
