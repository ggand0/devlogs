# 0099: Apply the fitted face and coif to MAA

Written by GPT-6 Astra.

2026-09-23. Gota accepted the spearman face/coif fit and requested the same for MAA, plus an update to the existing Claude handoff.

Created `assets_dev/man_at_arms/textured_v3_face_fit/`. The face, coif and closed neck match the accepted spearman source geometry exactly. The same original facial image and landmark mapping are retained. The exposed skin follows the cheeks and jaw, with a curved mail opening instead of the rectangular beige extension.

The neck adds 20 triangles. MAA was already at 2,998, so the tiny 20 × 26 mm crown cap uses ten segments instead of twenty, saving 20 triangles. Only the top 6 mm of the crown changes; all geometry below it, including the brim, is preserved. Connected cap topology avoids reversed winding on the isolated cap found during export verification. Final L0: 2,998 triangles, 1.80 m tall, 1,852 source / 3,660 exported vertices.

Parts: body 1,466 triangles; weapon arm 430; each leg 320; shield arm 462. Pivots in GLB XYZ: weapon arm (-0.224, 1.435, 0), shield arm (0.224, 1.435, 0), legs (±0.115, 0.910, 0). Pivot nodes and all exported non-body position sets match the prior v3 exactly. MAA retains the rigid arm and sword in part 1. No new animation.

Independent GLB/channel checks, embedded texture checks, per-triangle part checks, closed face/neck and shield checks pass. The crown has no new opening or winding error. Viewed front, angled and low-front renders; the face follows the approved spearman contour. The pre-existing 468 non-shield open boundaries remain documented; full-model surface acceptance is not claimed. Only L0 is authored here.

Verified all 190 recorded prior files unchanged, including MAA v2/v3, the approved spearman face_fit folder and shipped assets. Source images also match. Updated `tmp/drafts/handoff-faces-and-shield-backs-v3-gpt6-astra.md` with the latest MAA/spearman paths, crown budget adjustment, part contract and verification results. Updated the earlier spearman handoff to point there.

No src/, renderer or Cargo work, no runtime launch, no asset promotion or commit. All output remains in the new ignored WIP folder. Reproduction commands and measurements are in its README; exported scope checks are in verify_export.py and validation.json.
