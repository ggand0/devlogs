# Archer lower-body rework

Written by GPT-6.

2026-10-12. Gota rejected ready_v1 because the stance exposes the tube-shaped legs and the legs cutting through the lower tunic. I had inspected the rendered sheets and sampled browser orbit captures, but treated the surface problems as reportable limitations when they should have blocked the candidate. A reference score and a width pass do not establish acceptable modeling.

## Inspected cause

The L0 hose in tools/blender/archer/geometry_reference.py has seven elliptical rings without enough separation of thigh, knee and calf. Its knee rings are at 0.41, 0.49 and 0.56 m while the knee joint is at 0.455 m. The skin blends over a 0.12 m band. The knight joint revision reshaped its sleeve and localized the elbow bend; its knee profile was revised in the same way.

The lower tunic is entirely rigid on pelvis in assets_dev/_skel/archer.py. With the leg pose changed, that cannot keep cloth over the moving thigh. The knight has coat_r and coat_l helper joints; skeleton.bake_helpers follows the thigh's swing and removes its twist. The archer lacks these because its body, bow limbs, strings and arrow already consume the 20-joint limit.

## Isolated 20-joint experiment

New work lives only in assets_dev/archer/lower_body_v1/. The build reads the delivered skeleton and modifies L0 positions, normals and skin attributes in its own output GLB. Shared model/tool code and the ready pose are unchanged. L1–L3, pivots, joint list, indices, part IDs, atlas and UVs compare unchanged. L0 stays at 2,987 triangles.

The new hose separates calf volume, a narrower knee at the actual knee joint, thigh taper and ankle. Knee blend is 0.07 m wide. The upper thigh blends back to pelvis at the hip. The tunic's left/right panels blend from pelvis to the respective thigh, with a higher slit and a small allowance in depth.

Standing and ready both have zero tunic–leg edge crossings with the revised mesh. The ready pose before the revision has 90 when checking the complete tunic against both legs; devlog 0216's 36 counts used only the lower subset of body triangles. Those counts must not be mixed.

A ten-second diagnostic alternates the foot stagger while retaining the upper-body ready pose and grounded ankle targets. It is explicitly a deformation test with sliding feet, not a proposed gait. At 201 checked times, including between saved frames, the original model peaks at 116 cloth-leg crossings; the revised 20-joint trial still peaks at 13. Most remaining crossings are a right-side cloth panel entering the opposite thigh during the reversed stance. Direct thigh weights carry the thigh's twist into the cloth. This prototype is not a finished fix and is not approved for shipping.

Close ready-pose renders show the first leg and garment changes. Further comparison/movement renders were blocked by an existing flanks process, PID 2186447. It was not started or stopped by this task.

## Required decision

Recommended next step: add the knight-style coat_l and coat_r helper joints, making a 22-joint archer, then test the same poses and diagnostic sweep. Shared authoring tools already bake these helper rotations by joint name. Do not enlarge the skirt repeatedly or narrow the stance to hide the rig problem.

Player capacity is declared in src/unit_meshes.rs (SKEL_JOINTS and its consistency assertion) and src/shaders/unit_pose.wgsl (joint arrays, function parameter arrays, and POSE_SLOTS_SKEL). src/render_units_gpu.rs derives its slot count from the Rust constant. The archer shot path itself remains Claude's integration task from note 067.

An asynchronous question asks Gota to approve the 22-joint rig and choose whether Claude or this task handles the player files. AGENTS.md requires permission for those shared source files; no such files or shared skeleton scripts were edited. While that decision is pending, keep the isolated L0 experiment and source unchanged. No 22-joint model was built.
