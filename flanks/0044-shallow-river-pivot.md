# 0044 — Impassable terrain experiment → shallow-wading river (2026-07-20)

Implemented by Kimi K3 (OpenCode), directed and reviewed by the owner.

## The experiment (checkpoint commit 4fbfd65, paused)

Built the full M2TW-style version: a blocked mask on the terrain grid
(river channel + slopes ≥ 1.0), wall-slide gating in the movement
integrate and the charge-shove, terraces restricted to ≥15 m so their
risers became impassable massifs, the river impassable except a north
ford and a south stone bridge (walkable deck via a `height_at`
override), stateless per-tick waypoint routing through the nearer
crossing, AI same-bank target preference, and river-aware reserve-line
deployment.

It WORKED, sort of: FL battery passed (ROUT break ~52%, DIR numbers
identical to baseline, CHARGE ordering, SURROUND pocket t=44s, FORM
exact) and 200k battles resolved. But two narrow crossings funnel
army-scale traffic into 0.0 m-spacing crush crowds that clear slowly,
and one hard-coded waypoint is a poor substitute for pathfinding.
Owner call: pause it. The commit stays as a checkpoint if real
pathfinding ever lands.

## What shipped instead

- **River is wadeable everywhere**: bed raised to 0.6 m center depth,
  removed from the blocked mask. Wading costs ×0.55 speed
  (`Terrain::wade_mult`, applied beside the slope penalty in both the
  ordered and the routing-flee speed paths).
- **Bridge stays**: deck `height_at` override + low-poly stone prop.
  Since `wade_mult` exempts the deck, the bridge is the emergent
  fast/dry crossing — no pathfinding needed. Men wade the river in
  view; armies fight in the shallows.
- **Impassable mask kept for steep ground only**: terrace risers
  (≥15 m steps), gorge walls, crater lips (crater carve refreshes the
  mask locally). Local physics, cannot gridlock an army.
- Deleted: crossing routing, AI river-side penalty, reserve-line
  deployment. Spawn is the plain main-branch layout again (wading at
  spawn is fine).

## Verification (all runs FL_VOLUME=0)

- 200k sandbox: clean ranks with a gap at the river, broad-front
  engagement incl. in the shallows, no crush blobs (nn min 0.35–0.76,
  no more 0.00), steady kill rate, 43–203 fps view-dependent; sim
  tick unchanged (grid ~4 ms, step ~6–9 ms).
- Spot checks: ROUT breaks at 52% (517/1000, baseline ~50% band),
  FORM all OK with spacing 1.05 < 1.36 < 1.87 (matches baseline).
- DIR/CHARGE/SURROUND sites sit 88–228 m from the river; checkpoint
  runs matched baseline exactly and the pivot doesn't touch them.

## Gotchas / notes

- Debugging lesson: `pkill -f <pattern>` and `pgrep -f` match the
  invoking shell's own command line — killed my own shells twice and
  left stale game instances polluting fps measurements. Use
  `pgrep -x frontline`.
- FL_VOLUME=0 exists and mutes everything; use it for all scripted
  runs (owner works nearby).
- The 17 fps full-army zoom reading is renderer fill cost at 2560x1440
  with ~108k soldiers drawn — same envelope as pre-branch, not a
  regression from this work.
