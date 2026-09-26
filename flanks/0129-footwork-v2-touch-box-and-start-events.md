Written by Claude Fable 5.1

# 0129: Footwork v2, perception through the touch box and start events

Branch `feat/melee-footwork`, commit **125f66c** on bee555e (devlog 0127 is the measured dead end before it; 0128 is another agent's probe, not read here). Gota's direction: no incremental perf; a fundamental fix within the footwork's scope, revising the footwork logic if needed. Proposal note: work/notes/footwork-v2-perception-2026-09-25.md, picture work/notes/vis/011-touch-wave.png. The commit is untuned and awaits Gota's feel checks, which he wants before detailed verification.

## Why the footwork cost what it cost

Its ~380 cycles per soldier over main were perception by polling: the 15 m far look (every idle man of a fighting regiment read ~500 men every 8 ticks), the per-tick validity check of the remembered enemy (two random reads), a second pass over the touch box for lanes; the rest is men moving who stood still on main. The 15 m sight was ours, not M2TW's: devlog 0121 lists the soldier's seek radius as unknown, and 796ff14 chose 15 m against a hollow at the edge of sight before any joining mechanism existed. Gota's own M2TW observation (devlog 0123): the joining rolls outward with a delay, no hard leave-formation distance.

## What changed (125f66c)

- **Enemy seen** = the nearest enemy in the touch box (the 2.0 m scan every man runs every tick; candidates up to ~3.5 m), one extra compare per enemy candidate in the closure. The 2nd rank sees the enemy through the front rank; the 3rd does not. The remembered enemy and his position come from that scan; the memo validity lookups are gone. An engaged man closes on a remembered enemy only while he sees one.
- **Start events.** A man whose first swing of the melee takes him out of formation, or a man out of formation whose ground speed crosses 1 m/s (the SIGHT_RUN bit's edge), pushes an event (his position and regiment) into his chunk's buffer. The next tick job applies the events after its grid rebuild, before the integrate: one 6 m grid query per event (JOIN_SEE_R, comrades of his regiment still in formation) sets a per-man memory `go_t` to FL_GO_MEMORY ticks (60). While the memory runs the man rolls FL_JOIN_REACT per half-second window, as before. Events carry positions, not indices, because a death sweep may reindex between ticks; a job from a dead world drops them. A man who merely resolves to go and stands blocked emits nothing: nobody has seen him do anything.
- **Fight face.** Each regiment records its living men's half extents along its facing and to its right (frontline.rs `ext_f`, `ext_r`) and which regiment its fight point belongs to (`fight_target`). A man out of formation with no enemy in sight heads for the nearest point of that block's rectangle (movement.rs `FightFace::goal`), the centroid only when he is inside it. Before this the two-on-one victim died 47% slower than on bee555e (274 vs 186 alive at 55 s); after it 30% slower (241).
- **Melee clock.** The regiment's melee clock and contact frame start when 3% of its men (at least 4) have an enemy in reach (`in_reach`, written by the kernel every tick, counted in frontline.rs as `contact_n`) instead of that many in wind-up at once. Found the hard way: with sight limited to the touch box the wide victim line never reached 15 simultaneous wind-ups, never counted as in melee, and nobody joined until patience ran out at t=36 s; bee555e only crossed the wind-up count because its 15 m sight walked extra men into contact first.
- **Knob.** FL_FAR_LOOK=1 restores the polled perception (far look, runner look, wind-up clock) in the same binary. With it the four deterministic scenarios (DIR, ARCHERY, pile-on wide, two-on-one) match bee555e bit for bit at every step of the build, which is how each plumbing change was checked.

New columns: `go_t` (u8) and `in_reach` (bool) in Units, the job, spawn and the death sweep; `starts` buffers per chunk in the job; `fight_target`, `ext_f`, `ext_r` in GroupData; `fight_face` per regiment in the job.

## Scenario statistics so far (deterministic scenarios, bee555e baselines in work/baselines)

Wide line (100 files, at ease) hit by a 12-file block; men out of formation at 10 / 15 / 20 / 25 / 30 / 35 s, victims alive at 55 s:

| build | out of formation | alive at 55 s |
|---|---|---|
| bee555e | 74 / 270 / 357 / 390 / 401 / 394 | 282 |
| v2 before the clock fix | 0 until patience (t=36), then 368 within 4 s | 370 |
| v2, events on any resolve, memory 60 | 64 / 451 / 462 / 441 / 428 / 407 | 343 |
| same, memory 30 | 64 / 450 / 451 / 433 / 416 / 395 | 335 |
| same, memory 15 | 64 / 429 / 421 / 403 / 383 / 367 | 345 |
| v2, events on first swing or breaking into a run (125f66c) | not yet measured (Gota's feel checks first) | |

The memory length barely matters: with events on every resolve the wave lights the whole line within 5 s, twice as fast as bee555e, because blocked men who never moved still triggered their neighbors. The committed rule limits events to men seen fighting or running.

Two on one, victim alive at 15 / 25 / 35 / 45 / 55 s: bee555e 459 / 376 / 304 / 244 / 186; v2 with the fight face (resolve events) 470 / 408 / 351 / 296 / 241. DIR at 38 s: damage per hit by sector identical (21.1 / 27.6 / 48.6); kills front 429 side 181 rear 662 vs 406 / 175 / 653; control alive 189 vs 174, test 0 vs 0, lone 39 vs 92. ARCHERY identical (no melee).

## Cost

Not yet measured for 125f66c: the cycle-counted pair (bee555e vs v2) is built and the box is clean (CVAT stopped, devlog 0127); three ABAB pairs at 200k are the next step after the feel checks. Expected from the design: the far look (~115 to 150 cycles per soldier), the memo check (~130) and the runner look gone; added: one compare per enemy candidate, the start-event queries (microseconds per tick), the fight-face clamp for joiners. Target: ~1880 to 1900 cycles per soldier against 2162 on bee555e and 1780 on main.

## Open

- Gota's feel checks on 125f66c (8 v 8, the two pile-on scenarios, 200k), then the roll-up tuning (FL_JOIN_REACT, FL_GO_MEMORY, the event reach) against the bee555e curve and his M2TW observation, then the two-on-one lethality, then the cost ABAB.
- The polled gates must stay bit-identical to bee555e after every change (they did at every step so far); new v2 baselines once the behavior is approved; then the movement.rs refactor against them.
- Trap for the reader: in frontline.rs the wind-up block also computes the fight line depth from each fighter's target; an early edit here moved that into the in-reach block and broke the polled gates on the pile scenarios until put back.
