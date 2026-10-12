# Archer release without the end nod
Written by GPT-6 Astra.

2026-10-12. Gota described release and upward aim v1 as mostly good, but asked to remove the small recoil nod at the end. Revision: `assets_dev/archer/release_aim_v2/`.

Removed the authored two-degree body settle from both releases, including its chest/head rotation and the accompanying bow-arm target movement. The release starts from its aim pose and changes only the draw arm and bow limb/string joints. The draw-hand follow-through and 0.15-second string return remain. The 35-degree aiming transition, held endpoints, existing draw, model and skin are unchanged.

Checked saved samples: pelvis, chest, head and bow-arm rotations stay fixed within quaternion serialization precision; the entire pose is identical from the 0.25-second saved sample through the 0.60-second endpoint. Both release starts exactly equal their source held-aim poses. Upward aim, elevated hold and GLB match v1 byte for byte.

Shared validators pass for all three clips. Dense ground, foot, string and surface checks pass with the same contact limits as v1. L0 width remains 0.546892 m; maximum string-half separation during runtime interpolation is 4.47 mm level and 4.26 mm elevated. Bow/body and forearm/body sampled crossings remain zero; string/draw-hand contact is confined to held aim and release time zero. Sampled neighbor results retain lateral clearance at 0.9–1.4 m and rank clearance at 1.4 m for these new clips.

Both fixed-view shot sequences and two-direction orbits are refreshed. Unchanged frame images are copied only when their sampled poses match v1 within 1e-7; changed poses are rendered in this iteration. Seven-view sheets and M2TW correspondence remain included. Engine integration, quiver relocation and reload are separate work.

Inspected consecutive right-side frames for both releases. The final poses remain steady after the draw-hand motion. All eight videos decode and play through their loops in Firefox, including both full orbits. Review links, images and framing checks pass.

Private assets commit: `195421af3ef99f43e0738829d76808aae0aa643a`.

## Approval and handoff

Gota approved release_aim_v2 on 2026-10-12: "good, approved." The approved package includes both releases without the end nod and the unchanged upward aim. Standing/ready and level draw were reviewed earlier. Motion and media are already committed in the private assets repository; the approval is a separate README status commit. No animation bytes changed in this close-out.

Handoff: `work/handoffs/111-archer-shot-clips-for-claude-2026-10-12.md`. It gives the current model, all eight clips and hashes, exact endpoints, arrow visibility events, 22-joint layout, existing capacity commit, runtime timing decisions and measured limits. The full firing cycle is still incomplete: quiver relocation and reload remain later work. Public runtime integration and model-build promotion remain Claude's work. The graphics tree was clean at handoff; unrelated private experiments and spearman README changes stay untouched.

Approval status commit: `6fc5ce13862f44d36dbde8b837d77fdb637584b5`. The authored clips and review media remain those in `195421af3ef99f43e0738829d76808aae0aa643a`.
