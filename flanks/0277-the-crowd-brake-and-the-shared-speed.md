# 0277: the crowd brake, what it is for, and why the shared speed does not replace it

Written by Claude Opus 5.5.

2026-10-20, Linux, main tree on `feat/archer-rework`; the code read is main at 8810113. No code changed, nothing built or run. Gota asked whether the rear ranks cannot push a whole unit because of the crowd brake, what the brake is for, and what the point of keeping it is now that the shared speed is the contact rule. This devlog records the answer from the brake's whole history: every commit that changed it, read in full, and devlogs 0002, 0021, 0182, 0208, 0210, 0211 and 0212.

## The insight

The shared speed and the crowd brake do different jobs.

- The shared speed decides what happens to speed when two men touch: along the line between them they split the closing speed by weight and go on together.
- The crowd brake decides whether a packed man keeps walking at all. Without it every man in a press walks toward his slot at full drive on every tick, and in a press his slot is inside the crowd. Its spring cut and its speed cut also calm the spring between men, which acts from 1.4 m, before anyone touches; the shared speed never sees that range.

Every run of the shared speed, in the port and in the game, had the brake on. Whether the shared speed alone keeps a press calm and from packing is not known.

## What the brake does on main

Each neighbour within 1.4 m adds 1 minus his distance over 1.4 to a man's crowd number (`scan`, `crowd += w`). `crowd_brake()` turns that into `jam`, 0 when free and 1 when packed: the fade starts at six neighbours at 1.3 m (`BRAKE_FAR`, arm's length of the 1.70 m man) and ends at six at 0.9 m (`BRAKE_NEAR`, his width with arms and shield), both times the unit scale. With `jam` at 1:

- his own drive is gone (`yield_to_crowd`: `desired *= 1.0 - jam`);
- the spring between him and his neighbours is gone (`steer`: `push * (SEP_STRENGTH * (1.0 - jam))`);
- his speed loses 40% on every tick (`steer`: `new_v *= 1.0 - 0.4 * jam`).

The overlap correction (0.15 of each overlap a tick) and the shared speed still act. A man under the brake's start with nobody in reach also stops chasing a remembered enemy (`close_in`, `crowd < crowd_brake().0`).

## How the brake came to be, commit by commit

1. **2026-07-08, b986e15, devlog 0002 ("the compression saga").** Naive steering and separation at 100k men packed the crowd tighter and tighter (nearest neighbour about 0.6 m and falling, minimum 0.28 m for 0.62 m cubes). The cause: the separation pushes cancel in a symmetric crowd, so a man inside it feels no net push while his walk to the goal keeps compressing the pack. The fix counts the crowd as a scalar that cannot cancel and fades the goal drive to zero between `CROWD_SLOW` 1.2 and `CROWD_STOP` 2.5 (six neighbours at 1.12 m and at 0.82 m). A fade that only approached zero (`1/(1+k·c)`) was not enough: the crowd never stopped compacting. Result then: nearest neighbour 0.74 to 0.78 m, stable.
2. **2026-07-10, 0a9c24c, devlog 0021.** The crowd twitched every frame. Two overlap solvers fought inside 0.9 m (a hard force boost and the positional correction, packed pairs ping-ponging about 3 cm a tick), and the spring's energy bounced between wedged men. The fix resolves overlap by position only and uses the brake's value, now named `jam`, to cut the spring by 60% and damp speed by 25% a tick when packed.
3. **2026-07-10, 0684e3d, devlog 0021, round two ("quasi-static clogs").** The packed crowd still twitched. The spring now fades to zero at full jam and the damping rises to 40% a tick; the correction is under-relaxed (30%, 0.1 m cap). The rule it recorded: at high density only positional correction, forces for the open field. Engaged lines' mean movement fell from 0.03 to 0.11 m a tick to 0.005.
4. **2026-07-12, f8c509b.** The brake's start became the test for open space: a man with no enemy in his near scan looks out to 4 m only under it, so dense presses are not touched by the wider search.
5. **2026-07-20, 9073669 and c1c3f93.** Changed what the brake counted, inside the opt-in `FL_RECTFIGHT` mode only; the default path was unchanged and the mode was removed on 2026-09-26 (448b41c).
6. **2026-09-26, 28a8775.** The tick split into `src/sim/soldier.rs`; the brake became `yield_to_crowd`. No change in behaviour.
7. **2026-10-04, 3ab4fc5 and b87b77b.** A shove limit on top of the brake (a man's drive gone at six neighbours at 1.2 m), played and removed the same night: a unit could no longer be forced through a gap in the unit ahead (devlog 0182).
8. **2026-10-05, e8840e0, devlog 0208.** With the body distance at 1.1 m and wider, the fade still ended at a fixed 0.82 m, so packed men kept driving into men they already overlapped, were set back out, and twitched every tick. The fade's end now followed the body distance.
9. **2026-10-07, 8bf1152, devlog 0210.** The gap waves: set from the body distance, the fade was 11 cm of squeeze at a 1.2 m body, so when a gap opened at the front of a press the whole lane behind it got its drive back within a tick and surged and stopped as one body. The fade now runs from six neighbours at 1.3 m to six at 0.9 m by the men's real size, with the legs capped at 4 m/s². In the port's push scene the pair took the breach's wave patches from 45% (the soft wall as played) to 17%; the band alone gave 39% and the cap alone 29%. This was judged on the soft wall; under the shared speed the port could not judge it, because the mass broke the holding line before a wave could form (devlog 0211, 2026-10-08).
10. **2026-10-07 and 08, 9518d48, 545c854, 46bb464.** Comparison switches added and removed, and main's old band removed with `FL_FIGHT_ROOM`. Then cdc4227 (2026-10-08) made the shared speed the default contact.

So the brake's two halves come from two reasons: the drive fade from July 8 against a crowd that packs itself, and the spring cut and damping from July 10 against a crowd that shakes. Both are three months older than the shared speed.

## What the brake costs today

- **The push.** It slows a press's push without stopping it. Under the defaults the pile's attacked front gave 5.1 m by 24 s and 8.5 m by 40 s (devlog 0211, step 1), where e8840e0 with the hard stop had stopped at 6.4 and 6.3 m. In the port's push scene the red mass got through the holding line by 58 s with the current band and by 22 s with 0.2.1's (figure 043). How far a whole unit is pushed back in the game, its rear edge included, is not measured under the shared speed; in the game's push scene the line's men fight and die and the mass goes through the gaps instead.
- **The killing pace.** The fade starting at arm's length is the whole of the slower killing in the six-on-one pile and the 200k scripted front, and half of it in the 200k AI battle (devlog 0211, step 6). Gota kept that pace on 2026-10-10.

## Corrections to what was said in this thread

- The soft wall is not the default; the shared speed is, since cdc4227 (`share_contact()` is true unless `FL_CONTACT=wall`).
- "A settled press does not push" and "the rear ranks just stand" were wrong for the defaults: the attacked front keeps giving ground (above).
- "The brake takes about a tenth off the push" came from a table comparing two brake bands with the brake on in both; it says nothing about the brake against no brake.
- The brake was not introduced for the shared speed's twitch. Its twitch part is from July (0a9c24c, 0684e3d), and the shared speed reached zero shakers through three things together, all with the brake on: the shared speed, the setting apart at 0.15 and no 1 cm threshold (devlog 0210).

## Open

- The check that says whether the brake can go: the port (`work/scripts/viz/contact_toy.py`) with the brake off under the shared speed, measured for shakers, the press distance, the gap waves in the push scene and the push. About 15 minutes, no game code touched. Not run; waits for Gota.
- Records of this thread's process failures: `work/notes/097` to `100` and memory `read-before-answering`, `report-incidents-to-memory`.
