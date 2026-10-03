# 0185: the standing pose's forward reach, what the limit really is

Written by Claude Fable 5.1.

2026-10-03. Follows Astra's devlog 0184 (`standing_v2`, the diagonal carry) and `work/notes/033-knight-standing-v2-reach-review-2026-10-03.md`. No code or asset changed.

## The question

Handoff 085, item 1, asked for a standing knight "no wider than 0.85 m; nothing more than about 0.5 m ahead of his centre". Astra's `standing_v2` is 0.815 m wide and reaches 0.573 m ahead, 7.3 cm over the second number. Gota finds the pose good and asked whether the 0.5 m is a hard requirement.

## The answer

No. The 0.5 m was a round number Claude chose when writing the handoff. `standing_v2` is within the real requirement at 0.573 m. Final approval of the pose stays with Gota.

The requirement behind the number: a standing man's weapon must stay clear of the rank ahead at the depth the slots are apart, 1.4 m in normal order and 1.15 m in a wall (`FormSpacing::pitch()` in `src/formation.rs`).

## The limits, for the record

| Case | Between centres, front to back | The man ahead reaches behind his centre | Forward reach at which the blade touches him |
|---|---|---|---|
| Normal order, every man on his slot | 1.4 m | 0.16 to 0.21 m (measured on the four shipped models) | about 1.2 m |
| Wall | 1.15 m | the same | about 0.95 m |

Men do not stand exactly on their slots. The game's log shows standing neighbours as close as 1.17 m (`nn min` in `tmp/runs/scale/s164.log`), and a marching man leans forward and swings his weapon arm. With that taken off, the working limits are:

- about 0.9 m in normal order;
- about 0.75 m if the rear ranks of a wall hold this pose.

`standing_v2` at 0.573 m is inside all of them. Astra's own check agrees: in the static five by five block the smallest surface gap is 0.67 m at a 1.4 m pitch and 0.33 m at 1.05 m, and in both the closest pair stands side by side, not front to back.

## What this does not change

- The width target, 0.85 m, is the number that matters for the standing pose. `standing_v2` meets it.
- The press, where men are 1.0 m apart or less, is not the standing pose's concern. Men in a fight hold the ready pose (item 2 of the handoff), whose rule stands: sword up and back, nothing levelled forward.

The same text is in `work/handoffs/085-narrow-stance-and-guard-for-astra-2026-10-03.md`, in the notes under the essential set.

## The other numbers in the handoff

Gota then asked whether the other four items carry similar round numbers. They do. The handoff now has this table under "Which numbers above are hard":

| Item | Number in the handoff | Hard or round | The requirement behind it |
|---|---|---|---|
| 1 standing | no wider than 0.85 m | round; the limit is 0.9 m | 0.9 m is the distance at which the sim stops two bodies overlapping (`HARD_RADIUS`), so a man must be narrower than that |
| 1 standing | about 0.5 m ahead | round; working limit about 0.9 m | clear of the rank ahead at the slots' depth (above) |
| 2 ready | no wider than 0.8 m | round (M2TW's body); the limit is 0.9 m | the same 0.9 m, because pressed men end up that far apart side by side |
| 2 ready | ranks 1.0 m apart in a press do not touch | hard, as a check | the whole pose, shield included, reaches less than about 0.7 m ahead: pressed ranks are 0.9 to 1.0 m apart and the man ahead reaches 0.2 m behind his centre |
| 2 ready | nothing levelled forward | hard, and not a number | the blade never points past the shield's front; this is the fix for swords in the rank ahead |
| 3 side shuffle | none | | starts and ends exactly in the ready pose, facing kept, the planted foot holds the ground. The distance a cycle covers is free; record it |
| 4 forward and back shuffle | 1.35 m and 1.22 m in 1.5 s | reference only, M2TW's values | the same three requirements as item 3. Any cycle distance works, because the game plays the clip by ground covered |
| 5 the cut | lands about 1.6 m ahead | round; anything from 1.4 to 1.8 m | 1.8 m is the knight's reach in the sim, the ceiling. The sim's fighting distance will be set from where the clip lands, so record it |
| 5 the cut | contact on the hit tick | not a number to hit | mark the contact frame; the engine fits the wind-up to the knight's 0.4 s |
| all | height 1.80 m, part ids, pivots, atlas, triangle budgets, L3 body only | hard | the model contract (plan 009) |

Gota then asked for the handoff itself to be revised, not only annotated. Its "What to make" section is rewritten: a short part on how to read the numbers, the table with a requirements column and a targets column for each of the five items, and the reason behind each requirement. The appended table above is folded into it.
