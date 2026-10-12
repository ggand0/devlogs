# Archer draw first candidate

Written by GPT-6 Astra.

Gota reviewed standing_ready_v2 as good enough for now and requested the draw animation next. The review and deferred quiver relocation are recorded in devlog 0234. Quiver placement should move toward the M2TW archer before authoring retrieval; no quiver or model changes were made here.

Built assets_dev/archer/draw_v1/archer.draw.json from the exact reviewed ready sample: 31 samples at 20 Hz, 1.5 s, zero travel. The hands gather at 0.40 s, then raise the bow and pull the string to the jaw. Aim is steady from 1.30 s and stored separately in archer.aim.json. The chest turns while the reviewed legs remain planted. events.nock=0.40 controls arrow appearance in the review; the player still needs that draw-specific visibility wiring.

Inspected the M2TW attack_missile_ready and hold sheets and every reference frame in ordered montages. Four event correspondences: ready:0 at our 0 s, attack_missile_ready:51 at 0.40 s, :64 at 0.80 s, attack_missile_hold:0 at 1.50 s. The full reference includes retrieval and lasts 4.5 s. Our motion is independently authored from contact targets. Comparisons use equal projected scale, with the reference's 2.2 m cells padded into our 3.0 m framing. References and derived measurements remain uncommitted.

The shared clip validator passes. Dense interpolation checks at 151 times cover all four detail levels; sole drift is below 0.000004 mm. Maximum visible width is 0.842 m; arrow reach is 1.175 m. String/nock error between saved samples is at most 1.70 mm, arrow/grip line error 2.54 mm. Saved drawn samples have exact string contact within floating-point tolerance.

Known unresolved contact: the braced upper string crosses the bow forearm briefly during the first hand movement (sampled at 0.15 and 0.20 s). The body and arrow are clear in the final candidate's sampled surface checks; string/finger contacts at the nock remain expected. Earlier candidate paths crossed the torso and were corrected by moving the draw path outward. Several bow-turn paths worsened body clearance and were not retained. The delivered candidate exposes the remaining sleeve contact in the review instead of claiming clear geometry throughout.

Nine spacing phases found no lateral crossings at 0.9, 1.0, 1.05 or 1.4 m. Forward/rear neighbors can intersect, including a ready or offset-draw figure ahead at 1.4 m. The check includes visible arrows and reports sampled intersections, not penetration depth or continuous clearance.

The four reference comparisons have 3, 10, 10 and 5 rows over tolerance. The page explains each: reviewed fixed footwork, upright bow during nocking, more upright chest and contact targets adapted to this model. Full aim differs most at the right upper arm (32.4 degrees), with left forearm 21.6 and left upper arm 16.4 degrees. No reference match is claimed for those arms.

The review includes oblique, side and gameplay videos, a 24-second orbit in both directions, key poses on all levels, full local reference GIFs and event comparisons. Model, standing and ready bytes match standing_ready_v2. No public source or game integration changes. Draw v1 is awaiting visual review, and the sleeve contact remains follow-up work.

Reviewed all consecutive authored motion frames, the aim from side/oblique/top cameras, the matching full-aim reference sheet, and browser playback captures. All four MP4s decode and advance through complete browser loops, including the 24-second orbit. Main/reference links and images pass. See draw_v1/review_validation.json.

## Review outcome

Gota said the draw looks good. The next requested work is both release and upward aim, recorded in devlog 0236.
