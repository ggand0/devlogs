# Knight standing pose

Written by GPT-6.

Created the first standing candidate on `feat/knight-stand`, using M2TW standing sheets as visual references and the shipped knight as the geometric basis. The pose lowers the sword, draws the arms closer and leaves both feet planted. A separate table preserves the original animation joint basis. The L3 shield required a static mesh correction because it is part of the all-body far mesh.

L0 width drops from 1.150418 m to 0.832389 m; L1 and L2 are 0.839465 m. The corrected L3 is 0.699596 m. All heights remain 1.800 m and triangle counts remain 2768/678/250/52. Furthest forward reach is 0.410989 m, and sword ground clearance is 0.143902 m. The 5x5 formations at 1.4 and 1.05 m pitch have minimum surface gaps of 0.681553 and 0.335490 m.

The first wrist blend folded the cuff. Spreading the wrist turn along the forearm removed negative deformation Jacobians in dense triangle-interior checks across L0-L2. The shield received a slight backward pitch to increase its body clearance to 19.8 mm. The sword/body gap is 35.7 mm. Weapon/body and shield-board/body edge-crossing checks pass at L0-L2.

The raw GLB comparison preserves L0-L2 positions, UVs, vertex colors, atlas pixels, part IDs, pivots and triangle connectivity. Blender changes custom normals slightly on re-export; this is measured in validation.json. Surface topology counts match the source, including its inherited L0 exceptions. The saved scene was reopened and compared with the JSON deformation, with less than 0.000061 mm position error. No engine code changed and no build, game or sim hash run was needed.

Review package: `assets_dev/knight/standing_v1/review.html`, the four labeled sheets, posed Blender scene, table, candidate GLB, scripts and validation reports. The iteration is preserved in the private assets repository on main. Integration details and limitations are in `work/notes/032-knight-standing-pose-2026-10-03.md`. Stop here for Gota's review of the standing pose.
