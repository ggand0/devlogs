# Roadmap from 0.1.1 to Early Access
Written by Claude Fable 5.1.

Date: 2026-09-26, after the merge of PR #10 (main 66eb416). No code change. This entry records the milestone plan agreed with Gota so the graphics tree can rank its work against it. The plan itself, with status marks, is docs/plans/018-roadmap-to-early-access.md.

## What was decided

- 0.1.1 ships what is already on main plus the terrain look once it merges: textured models, LOD, the GPU render path, footwork, the two tick branches. No new feature gates it. The README and screenshot get refreshed first, since the README still says there are no external art assets.
- 0.2.0 is the battlefield: a map format and loader, the band grid rebuild, one authored map at Astra's proposed 2,048 by 1,536 m, terrain as physics (slope fatigue, downhill charge momentum, height for archers), the AI using the map, and three graphics items only: atmosphere and fog, sun shadows with the unit shadow draw, soldier variety.
- Pathfinding is in 0.2.0 only if the authored map has ground that regiments must walk around. That is decided when the map layout is fixed. Otherwise it goes to 0.2.1.
- 0.2.1 is the scenario file and presets. Gota moved it out of 0.2.0 because 0.2.0 is already full.
- 0.3.0 is cavalry, with the pair pass done first so the fifth kind is built once on the final kernel. Second map, authored vegetation and props, and the remaining animation clips ride with it.
- 0.4.0 is the Steam Early Access candidate: settings, rebinds, saves, a tutorial, the death audio pool.

## Announce beats

Technical audiences with 0.1.1, indie and RTS audiences with 0.2.0 in the same week the Steam coming-soon page goes live, r/totalwar only after cavalry, Early Access with 0.4.0 and a Next Fest demo, realistically February 2027.

## What this changes for the graphics tree

After terrain: atmosphere and fog, then soldier variety textures. Authored vegetation and props wait for the second map in 0.3.0. Capsule art and unit portraits from the GLBs are needed before the 0.2.0 announce, for the Steam page.

## Immediate next steps in the main tree

The remaining plan 017 perf items (Gota's view log run, the band grid rebuild), then the terrain branch review with a feel pass and new baselines, then the 0.1.1 packaging, then the 0.2.0 map format.
