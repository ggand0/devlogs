# Archer textured orbit review

Written by GPT-6.

The user said the arm revision might be suitable but needed textured orbit videos with animation to judge it. This is additional review coverage for arms_v4, not a new model iteration or permission to ship it.

Added `assets_dev/archer/arms_v4/render_orbits.py` and `compose_orbits.py`. The render script reads the existing GLB and shot clips and uses the game's CPU skinning calculation. It does not change geometry, materials in the GLB, animation JSON or the rig. The review displays the textured model with the bow visible. The held arrow disappears at release, matching the existing release previews.

The neutral turntable lasts 8 seconds at 960 by 1080. The full-body orbit lasts 12 seconds at 1080 by 1080 and shows four level shots. The close orbit lasts 18 seconds at 1280 by 960 and shows four raised shots while orbiting in the opposite direction. All run at 24 fps. Full-body framing includes the figure, bow tips and nocked arrow. Close framing fits the head, shoulders, torso and hands; bow tips can leave the image. Camera scale stays fixed within each video and is fitted against every rendered pose and camera angle. Shot loops restart after the release hold, without inventing a reload transition.

The videos appear at the top of `review.html`, with normal, half and quarter speed controls. Encoding validates expected frame counts, dimensions, GLB hash correspondence and complete decoding. Contact sheets provide samples throughout each camera revolution. The model remains the same as the preceding arm review. No asset export, game build, public code change or commit was made.

The existing model limitations remain: L1-L3 still use the source geometry, hands have closed grips without individual finger articulation, and the smaller hood and full shoulder cape remain separate work. The additional videos are for visual judgement, not an assertion that the model or animations have passed the final asset checklist.

Final checks: 192 neutral frames, 288 level-shot frames and 432 raised-shot frames encoded and decoded successfully. All three render manifests match the current GLB hash. All local links in the updated review page resolve. Inspected full-size framing samples and contact sheets throughout each orbit; no real-time playback or in-game approval is claimed.
