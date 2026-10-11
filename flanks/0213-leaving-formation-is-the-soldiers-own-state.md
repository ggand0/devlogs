# 0213: Leaving formation is the soldier's own state (the shuttle fix)

Written by Claude Fable 5.1. Branch `fix/side-shuttle` off main 9115bce, 2026-10-11. Devlog 0212 has the cause; this one has the fix, how it was checked, and the numbers.

## The rule in plain words

Before: a man was allowed off his slot only while his whole regiment counted as being in a melee, and that count started when 3 percent of the regiment (at least 4 men) were winding up a blow on the same tick. An enemy standing off the flank never got a regiment there. But the chase ran anyway for every formed man whose regiment was not in a melee. So the man went for the enemy, his slot pulled him back, and the two pulls traded the lead at fixed distances forever.

Now: whether a man is out of formation is his own business.

- He leaves formation when he strikes, or by the join's roll (20 percent per half second) for an enemy he sees. While his regiment's melee runs, the enemy can be anywhere in the seek radius (15 m) or a comrade seen running to the fight, as before. When it does not run, only an enemy within the short look (4 m) counts, so a lone enemy at the flank pulls out the two or three men nearest him and nobody else.
- Out of formation he has no slot pull. He chases, fights and steps back as a committed man did before.
- He comes back to his slot once he sees nobody: nobody in reach and no remembered enemy within his closing range (5 m, or 16 m while anyone of his regiment strikes), and his regiment's melee is not running. A Move order, a hold order, a rout or death bring him back at once. While his regiment's melee runs he stays out until it ends, as before.
- The chase for a formed man is gone. A formed man who sees an enemy leaves formation first, so the slot never pulls against the chase.

The regiment's clock, contact frame, join wave and patience are untouched; they still run a real melee.

## The code

`src/sim/soldier.rs`:

- `begin`: `out_form` is cleared only when the man may not leave (routed, dying, his regiment on hold or under a Move order), set by a strike as before. `committed` is now `out_form` alone, not `in_melee && out_form`. Two new `Step` fields: `may_leave` and `memo_d2`.
- `look_around`: keeps the remembered enemy's squared distance in `memo_d2`.
- `decide_to_join`: the return (a committed man outside his regiment's melee who sees nobody within his closing range comes back), then the departure with the short look outside a melee, and the patience only inside one.
- `close_in`: the chase of a remembered enemy is for committed men only.

`src/frontline.rs`: the comment at the melee's end.

`src/regiments.rs`: the wander count in the shake log (below).

## The check: the wander count

The shake log counts turns per tick; a shuttling man turns about once a second, so it missed him. `FL_LOG_SHAKE=1` now also reports, every 4 s, the men who walked more than 4 m in the window and ended within 1.5 m of where they began, with their regiment's state (fighting, moving, standing), how many are out of formation, in a crowd under the brake's start, and the six who walked furthest with their place on the field.

Two binaries, both with the count: main 9115bce built from `git archive` into `tmp/runs/side-shuttle/target-base` (`bin/flanks-base`), and this branch (`bin/flanks-fix`, built before the `may_leave` rename, which changes nothing). Script `tmp/runs/side-shuttle/wander.sh <binary> <prefix> [scenes]`, logs in `tmp/runs/side-shuttle/logs/`.

Wanderers summed over the windows after the first 4 s:

| Scene | Base | Fix |
|---|---|---|
| AI battle 200k, 180 s, run 1 | 397 (152 out of formation, 232 in open ground) | 62 (36 out of formation, 18 in open ground) |
| AI battle 200k, 180 s, run 2 | 625 (143 out of formation, 437 in open ground) | 70 (37 out of formation, 24 in open ground) |
| Retarget, 120 s | 7 | 4 |
| Pile two-on-one, 90 s | 0 | 0 |
| Pile wide line at ease, 90 s | 1 | 0 |

What the base's wanderers are: formed men (not out of formation), crowd near zero, of regiments at the ends of the line or of regiments marked fighting whose clock is not running. Run 1's worst window at 99 s had 56 of them, 34 in standing regiments, all in open ground, mostly regiment 26 at the line's west end. That is the shuttle. The fix's wanderers are the crash contact at 18 s (33 men out of formation jostling in the press, crowd 0.5 to 1.0) and men of marching columns walking in a press (crowd above 1). After the crash, no window on the fix has more than 10.

The scripted scenes never showed the shuttle in numbers; it needs a flank contact of an open battle.

## The pace

The scripted scenes' kills: the retarget scene equal (R's two targets 624 alive on base, 620 on the fix at 70 s; R itself 79 and 85). The two-on-one pile equal (708 alive at 87 s on both). The wide line at ease locks its melee clock 10 s later on the fix (at 22 s instead of 12 s) and then kills at the same rate, so it runs about 60 men behind: before, every formed man of the line within 5 m chased the attackers at once; now the men within 4 m come out one by one on the roll, and the 3 percent lock takes longer to reach.

The AI battle's dead at 123 s: base 10,225 and 17,226; fix 8,531 and 8,650. The base's two runs differ by 1.7 times (the AI battle runs on wall time, devlog 0211 saw a fifth), so the comparison is not a measurement. Both fix runs sit below the base's lower run, and the mechanism above (contacts lock later, thin contacts hold fewer men) can only lower it. Gota's play decides whether that reads as slower.

## What to watch in play

- A lone enemy or a thin blob at a flank: the nearest few men go out to it and fight; nobody shuttles; the rest stand in rank.
- A regiment whose melee has ended but with enemies still standing within a few metres: the men who see them stay out fighting instead of dressing; the rest re-form.
- Regiments arriving at a line: the first contact grows man by man from the front instead of every man within 5 m rushing it.

## Gota's play, 2026-10-11

Fixed in the situation of the report (`refs/shuttling-fix/debug0.png`: a thin orange blob at a blue regiment's flank, nobody shuttling), and the wide and narrow contacts both work. The follow-ups are not needed: only the men within about 4 m of an enemy go out, which is the number Gota wants. Gota's first guess for the cause had been the comrade-go rule (a man following a comrade running to the fight); that rule is untouched and still runs only inside a regiment's melee.

## Commits and the PR draft, 2026-10-11

The two commit messages were rewritten to main's style (one unwrapped paragraph, no devlog references, no log counts): the fix is 371608a, the wander count 405929b; the tree is identical to the first pair, kept at `backup/side-shuttle-2026-10-11` (8ee37c2, aafd015), to delete once the PR is merged. The PR draft is `work/drafts/045-pr-side-shuttle.md`, in the 043 shape: glossary, notes, then a body with no sim words and no log counts.

## Merged, 2026-10-11

Gota played a few more battles ("looks good and fixed"), opened PR #26 with the 640 px copy of `refs/foot-work-swing/debug0.png` and merged it with a merge commit: main 2a1e06b. The final checks before the merge: the diff of the two commits against main (three files), the messages (no footer), clippy clean and 35 tests passing. The backup branch `backup/side-shuttle-2026-10-11` waits for Gota's word to delete.
