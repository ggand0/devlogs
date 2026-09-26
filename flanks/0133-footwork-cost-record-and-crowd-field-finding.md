Written by Claude Fable 5.1

# 0133: The footwork's cost on the fps counter, the variant record, and the crowd field's main-thread cost

Branch `feat/melee-footwork` at 5639239, nothing committed, no source touched (all builds are instrumented copies). Gota's readings on the fps counter's two second average in the 200k battle once the lines engage, his own view: this build 90 to 100, 796ff14 110 to 120, main 125 to 135. He set the bar for this branch at 110 to 120, moved the perf work to another branch (plan: docs/plans/013-peak-kernel-pair-pass.md, with the reasoning and the clarifications of the proposal exchange), and asked for a short measurement of the branch's variants for the record, including the tick's components with mean and median, raw records saved.

## Recipe and tooling

- work/scripts/perf/tickrec.py patches a copy of movement.rs so every sim tick logs one line: grid, kernel, field and damage-apply milliseconds, damage events, units, kernel cycles (rdtsc over the chunk loops, as cyc.py). It is idempotent over the older cycle counter in tmp/runs/rear/mainsrc.
- work/scripts/perf/bench4.sh builds instrumented copies of named commits (`git archive` into the session scratchpad, `--target-dir target-agent`, binaries copied out as tmp/runs/perf4/bin/flanks-<name>) and runs each once: `FL_VOLUME=0 FL_LOG_STEP=1 FL_AUTOSTART=1 FL_DEPLOY=0 FL_HEAVY_FRAC=0.4 FL_CAM_LOCK=1 FL_CAM_DIST=900 FL_NO_CULL=1`, 80 s, the locked view of devlogs 0079 and 0081. A lingering binary is killed by its own name, never `flanks`.
- work/scripts/perf/tickstats.py summarises the logs over the engaged window (battle time from 30 s): fps is the overlay's two second average, the same number as the top-left counter; the tick columns are mean / median / p90 of the per-tick records; the pacing columns come from the periodic frame-pacing lines.
- Raw: tmp/runs/perf4/logs/<name>.log (every tick and every periodic line), build-<name>.txt, bench.out, summary.md. Builds took 19 s each with the shared dependencies.
- Caveats: one run per variant (differences under about 5% are noise), the AI battle is not deterministic run to run, the load average of 4 to 6 during the runs was the game itself, the GPU clock sat at 210 to 750 MHz (the view is CPU-bound). Trap of the day: the chain reused one log name for the plain and the FL_FAR_LOOK=1 runs of 5639239 and overwrote the first; the plain run was repeated with `LOG=` set.

## The record, 200k, engaged window (battle time 30 to 79 s)

| run | fps (mean / p50 / p10) | frame ms | grid ms | kernel ms | field ms | apply ms | events per tick | kernel cycles per soldier | frame ms without a tick p50 | frame ms with a tick p50 / max | fixed tick p50 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| main (cc7b08b) | 161 / 162 / 155 | 6.21 | 3.60 / 3.57 / 4.17 | 5.91 / 5.80 / 6.65 | 0.79 | 0.028 | 67 | 1674 / 1677 / 1802 | 4.0 | 9.1 / 11.2 | 5.5 |
| 796ff14 (seek, wait, sidestep, grip) | 146 / 145 / 144 | 6.85 | 3.75 / 3.71 / 4.37 | 7.60 / 7.57 / 8.20 | 0.78 | 0.026 | 56 | 2198 / 2205 / 2366 | 4.6 | 9.5 / 11.8 | 6.0 |
| bee555e (v1 perception cadence) | 148 / 150 / 139 | 6.75 | 4.03 / 3.96 / 4.70 | 6.93 / 6.82 / 7.63 | 0.79 | 0.027 | 62 | 1981 / 1986 / 2125 | 4.4 | 9.6 / 12.0 | 6.2 |
| 5639239 with FL_FAR_LOOK=1 (bee555e's behavior, the crowd field still built) | 115 / 116 / 109 | 8.71 | 4.04 / 3.99 / 4.66 | 7.03 / 6.90 / 7.75 | 4.24 / 4.07 / 4.96 | 0.025 | 62 | 2011 / 2009 / 2150 | 5.2 | 13.1 / 15.9 | 9.5 |
| 5639239 (current, rerun with its own log) | 119 / 120 / 116 | 8.41 | 4.02 / 3.97 / 4.68 | 7.00 / 6.91 / 7.60 | 4.21 / 4.04 / 4.95 | 0.018 | 40 | 1978 / 1974 / 2110 | 5.1 | 13.0 / 15.6 | 9.4 |

Cells with three numbers are mean / median / p90. The current build and its far-look variant are the same within noise on every column but the events (40 against 62 per tick: the v2 behavior lands fewer blows per tick in this run), so the kernel of the approved behavior costs what bee555e's does, 1978 against 1981 cycles per soldier. The main-thread breakdown was the same in every run (fixed loop p50 0.5 ms, update and post 2.6 to 2.7 ms), so it is not in the table.

## What the numbers say

1. **The crowd field's rebuild is the fps drop.** On 5639239 the density field costs 4.2 ms per tick against 0.8 ms on bee555e and main, and that cost sits on the main thread inside the fixed tick (`update_field` runs there, before `step_sim`): the fixed tick's median went from 6.2 to 9.5 ms, frames that carry a tick from 9.6 to 13.1 ms, and the counter from 148 to 115 in this view. The kernel of the same binary is at bee555e's level (7.0 against 6.9 ms, 2011 against 1981 cycles). The extra 3.4 ms is what 5639239 added to frontline.rs: per chunk of 16k units, three full-field scratch arrays (count, position sum, box) cleared and resized every tick (about 9 MB of writes at 13 chunks), a serial merge of chunks times cells times three arrays, then the two raster sweeps per team. The sweeps are tens of microseconds; the scratch and the serial merge are the cost.
2. **The soldiers' own cost is what devlog 0124 said.** Kernel cycles per soldier: main 1674, bee555e 1981 (+18%), 796ff14 2198 (+31%), 5639239 with the far look 2011. The kernel wall follows: 5.9, 6.9, 7.6, 7.0 ms. Today's absolute level is below the two earlier sessions (1780 and 2517 for main); the ratios hold.
3. **The grid** costs 3.6 ms on main and 4.0 ms on the branch (the velocity column, devlog 0128). The damage apply is negligible (0.03 ms). The field on main and bee555e is 0.8 ms.
4. **Frames without a tick** grew from 4.0 ms (main) to 4.4 (bee555e) to 5.2 (5639239): the job's contention with the frame's systems, devlog 0081's mechanism, grows with the kernel.
5. Gota's readings in his own view (90 to 100, 110 to 120, 125 to 135) scale the same way as this view's 115, 146, 161: his view is heavier to render, the ratios match.

## What follows from it

The fps gap between bee555e and 5639239 is not the behavior and not the soldiers. It is the field rebuild on the main thread, and it has an exact fix of a few hours: the crowd maps are consumed only by the tick job (copied at prep), so they can be computed inside the job from `pos_in` on the worker thread, in parallel with `util::sim_scope`, with the per-chunk scratch cleared by its own task and the merge parallel over cell bands. Same positions, same arithmetic, bit-identical maps. The density blur and the contour for the drawn front line stay on the main thread at their 0.8 ms. That should put 5639239 back at bee555e's frame times, which is Gota's 110 to 120 in his view. Gota's decision: not on this branch. The diagnosis and the fix design are kept in docs/plans/014-crowd-field-cost.md for whoever brings the field back. The branch continues from bee555e (the fastest build, whose behavior he approved), with the crash phase and the committed man's facing rule cherry-picked from the v2 commits, then the refactor.

The remaining gap to main (the kernel's +18% and the grid's velocity column) is the branch's behavior and the shape of the tick, which is the plan in docs/plans/013-peak-kernel-pair-pass.md.

## Conditions of the record (Gota's notes, added the same afternoon)

- Nobody touched the keyboard or the mouse during the five runs, so the results are clean of input. winit caps the frame rate at 60 when the game window is not focused; every run here read well above 60, which proves the window had focus. Any future unattended run must check that: an fps column at 60 means an unfocused window, not the sim.
- The CVAT docker stack was crash-looping and loading the box for about two days before it was stopped on 2026-09-25 (devlog 0127). Benchmark numbers taken in those days (devlogs 0124 to 0127, and possibly the probes of 0128) may carry that load. Today's runs had only the desktop (load average 0.9 before the first run).
- The scenario is lenient. FL_AUTOSTART's AI gives the orange regiments attack orders against blue, and blue gets none, so blue's rear ranks stand until they are attacked. In Gota's own 200k runs he orders every unit toward the enemy, everything blobs up and fights, and more men are active than here. His counter readings (90 to 100 on this build, 110 to 120 on 796ff14, 125 to 135 on main) come from that harder scenario and his own view, so the absolute fps here (119, 146, 161) are higher than his, and only the ratios and the tick components transfer. A benchmark scenario that orders both sides in, or the all-fighting line layout of docs/plans/013-peak-kernel-pair-pass.md, is the right gate for the perf branch.

## The variants, what each one is

| build | how a soldier finds an enemy | how he notices comrades | where he heads with nobody in sight | regiment melee clock | added structures |
|---|---|---|---|---|---|
| main (cc7b08b) | the 2 m touch scan for reach, plus the 4 m sparse acquisition every 8 ticks for regiments near an enemy | not at all: a formed man keeps his slot and the line's facing | nowhere: he holds his slot; the attack order drags the slot grid after the target's centre | engaged flag from wind-ups | none |
| 796ff14 | the acquisition scan widened to 15 m (FL_SEEK_R) every 8 ticks, then go to him through open ground at the jog or the advance pace | the comrade tests (blocked, sides, ahead) inside the separation closure, every tick, for everyone | nowhere yet: no joining, men without an enemy in sight hold their slots | wind-ups | the contact frame (frozen at contact), standing grip |
| 372b115 to 5a5f8ff (not measured today) | as 796ff14 | a comrade running to the fight seen in the 2 m scan, then within 8 m on the far look | the fight point (the target regiment's centroid) after a delay, then after seeing a comrade go, then patience | wind-ups | the melee clock, out_form (a man stays out until the melee ends) |
| bee555e | the naive 15 m look, every 8 ticks, about 500 men read per look, the remembered enemy re-validated every tick by random reads | the comrade tests in their own pass, only for men going somewhere, every 8 ticks, remembered in the `sight` column; runners within 6 m on the far look | the fight point | wind-ups | the grid's velocity column |
| 125f66c (v2, not measured today) | the nearest enemy in the 2 m touch box he already scans; no far look | start events: a man who begins to fight or sets off is noticed by comrades within 6 m, who remember it 2 s and roll to follow | the fight face rectangle of the enemy block | men with an enemy in reach | the `go_t` and `in_reach` columns, the start-event pass |
| dc41a22 (not measured today) | as v2, plus the enemy crowd within 15 m counts as an enemy in sight | as v2, set-off events at 0.5 m/s repeated every half second | the nearest bin of the target regiment's 3 x 3 mass map | in reach | the mass map per regiment |
| 5639239 (current) | as dc41a22, the crowd read from the density field | as dc41a22 | the nearest point of the nearest crowd cell's box (the 8 m density grid, cells with at least 4 men, raster-swept) | in reach, and the crash phase of a charge before it | the crowd field (this is the 4.2 ms on the main thread), the crash state |
| 5639239 with FL_FAR_LOOK=1 | bee555e's look, in the same binary | bee555e's | the fight point | wind-ups, no crash | the crowd field is still built, so this row isolates the behavior from the field's cost |

## The lesson, in Gota's words

Kernel cycles per soldier were the metric of every perf session on this branch, and fps was never measured. The end user sees the frame. With the pipelined tick the kernel reaches the frame only through pool contention, while anything on the main thread's fixed tick lands in it at full weight, so a 3.4 ms field rebuild cost more frames than the whole perception rewrite saved, and nobody noticed. From here the gate is the fps counter's 2 s average in Gota's scenario (every unit ordered in, everything blobs and fights), with the frame anatomy lines next to it. Kernel cycles, grid and field milliseconds are diagnostics under that number, never the result.
