Written by Claude Opus 5.5

# 0124: Footwork perf at 200k

Branch `feat/melee-footwork`, after 5a5f8ff. Gota's ask: the kinematic step went from 6.5 ms (main) to 9.6 ms (branch) at 200k after the footwork work (devlog 0123); make it lighter.

## Measuring

Wall step times at 200k swing 15% window to window, so every build got a cycle counter: rdtsc around each integrate chunk, summed over all chunks, divided by soldiers. That's CPU cost, the number that matters with the pipelined tick sharing Bevy's compute pool with the frame (devlog 0079). An earlier pass also split the per-soldier loop into sections (pre, fused scan, perception, acquisition, swing, close, integrate, facing, position). Same instrumentation in a main build (`git archive` copy in tmp/runs/rear/mainsrc).

Run: `FL_LOG_STEP=1 FL_AUTOSTART=1 FL_DEPLOY=0 FL_HEAVY_FRAC=0.4` (200k), 100 s, last 12 five-second windows averaged.

Where the regression was (tick ~2850, cycles per soldier, 5a5f8ff vs main):
- Fused neighbor scan: same candidate count (about 30 per soldier late in the battle on both), but 61 vs 40 cycles per candidate. The footwork tests (way blocked, sides, comrade ahead, comrade going, mark blocked) sat inside the separation closure everyone runs, though only 13 to 26% of soldiers had a direction to test. The closure got heavier for every soldier: about 600 of the ~940 extra cycles.
- The 15 m sight scan for engaged regiments: 10x the candidates of main's 4 m acquisition (about 500 per scan), +110.
- Rest +230: the target lookup (random reads of the remembered enemy's team and position every tick), join logic, and a general +10 to 15% on sections whose code is identical (the battle itself differs).

## What changed

Four commits:
- ef428e3: the comrade tests moved into their own branch-free pass, run only by men going somewhere. The grid now carries each unit's tick-start velocity (a column in grid order) and regiment (upper bits of `meta`), replacing the random `vel_snap[o.idx]` / `group[o.idx]` gathers and the per-tick velocity copy. Fused scan back to 42 cycles per candidate. Bit-identical.
- 51a9142: the target lookup only when its answer can matter (no enemy in reach, or still in formation). Bit-identical.
- ed39965 (behavior): a man no longer counts himself as a comrade running to the fight (the 6 m check had no self-skip), and comrades' velocities are always read (before, only while some regiment anywhere was in melee: global knowledge leaking into a soldier's perception). Two-on-one victim dies about 15% faster (176 vs 211 alive at 55 s); wide-line roll-up slightly slower at 15 to 20 s.
- bee555e (behavior): a man looks at the comrades around him every 8 ticks in his own rhythm (`sight` column, SIGHT_* bits) and acts on what he last saw in between. Roll-up and pile-on track ed39965 within a few percent.

Result, kernel cycles per soldier (CPU per tick):

| build | cycles/soldier | kernel CPU/tick | step wall |
|---|---|---|---|
| main | 1780 | 88 ms | 6.8 ms |
| 5a5f8ff | 2599 | 131 ms | 9.5 ms |
| bee555e | 2162 | 109 ms | 8.1 ms |

About half the regression recovered. What is left is mostly the 15 m sight scan at its 1-in-8 rhythm and the new per-soldier logic.

## Tried and dropped

- Far look (15 m sight scan) once a second instead of every 8 ticks: engaged men picked up nearby enemies up to a second later, the regiment's melee clock started about 2 s later (wide line: 0 out of formation at t=10 s vs 67), and DIR's lone victim died much slower (115 alive at 38 s vs 52 to 76).
- Cell occupancy skip for the sight scan (per-cell team bits, skip cells without a living enemy): exact, fingerprints equal, but no gain (2215 vs 2162, noise) and +0.5 ms on the grid rebuild. Near the fight the 15 m box nearly always holds enemies. Patch: scratchpad only.
- Grid rebuild as a band-parallel two-digit radix sort: its output is bit-identical (the sort order is exactly (cell, index)), but its two sorting passes took about 13 ms of pool CPU per tick (6.5 + 7, summed task time) to replace a 2 to 3 ms serial merge that runs on the dedicated worker thread, off the pool. Wall time barely moved (4 to 5 ms down to 3.2). Not a win for frame time. The grid itself costs the same on main and the branch (bounds 0.3, count 0.9, merge 2 to 3 serial, scatter 1.2 ms).
- A "fix" limiting comrade ahead to a fixed radius (the reach is set by grid cells, 2 to 3.5 m ahead depending on where a man stands). Rejected by Gota as an arbitrary-number hack; the grid-dependent reach stays as it is.

## Gates

- The 200k AI battle is not deterministic: `ai_think` and `auto_engage` run in Update on wall time, so orders land on different ticks run to run (two of six runs diverged at tick 480). Its fingerprints prove nothing.
- Deterministic gates, work/scripts/gate.sh (writes tmp/runs/scripts/gates/): DIR, ARCHERY (40 s), pile-on wide line (`FL_TEST_PILE=1 FL_PILE_N=1 FL_PILE_FILES=12 FL_PILE_ATEASE=1 FL_PILE_VICTIM_FILES=100`) and two-on-one (`FL_TEST_PILE=1 FL_PILE_N=2`), 60 s each. work/scripts/cmp.sh compares numerically (hashcmp.sh sorts ticks as text). work/scripts/pilesum.sh summarizes the pile-on log every 5 s.
- Baselines: work/baselines/{dir,arch,pilewide,pile2}-footwork-bee555e.hash (the refactor must match these), pilewide/pile2 for 5a5f8ff too.
