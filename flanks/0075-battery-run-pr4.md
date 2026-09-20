# 0075 — Pre-merge battery run for PR #4, and what batteries are worth (2026-09-20)

Branch fix/controls-audio-qol (17 commits) opened as PR #4. Muted
insurance run of three batteries, 110 s each, FL_VOLUME=0. Picked for
coverage, not habit: the PR's only sim-behavior change is the archer
aim rework, and ARCHERY is the only battery that measures archery.
DIR and CHARGE ran as general canaries; SURROUND/FORM skipped
(untouched paths).

## Results: all green

- ARCHERY: 739 arrows in flight in the opening seconds, ammo
  draining, middle target block wiped by volleys by ~t=45 s. The
  late-run silence (flight 0, ammo frozen) is the scenario's own
  shape: the flank blocks sit deliberately beyond bow range, so once
  the middle dies there is nothing legal to shoot. Zero engagements
  all run — the fire-at-will hold rule never even triggered. The
  per-soldier aim pipeline (pick member, flight, kill) works.
- DIR: kills by sector front 586 / side 297 / rear 616, dmg/hit
  22.1 front vs 49.6 rear (2.2x, rear bucket dominant). Melee code
  untouched by this branch; this is today's baseline reading, not a
  delta.
- CHARGE: wall lane holds (spears ~150 steady at dz -5.5 m, heavies
  106 -> 102), open lane spears run down 350+ m and cut from 81 to
  45. The phase-B story intact.

## Usefulness verdict (owner asked, wants a real evaluation later)

Half-yes. Six minutes bought: proof the aim rework did not break the
archery pipeline, plus two canaries. Structural weaknesses: no
machine-checkable pass/fail (results are log archaeology against
thresholds living in devlog prose), silent rot as the sim evolves
(FL_TEST_ROUT already stale; ARCHERY's silent tail cost an analysis
detour before it could be called healthy), and zero coverage of what
this PR mostly is — feel and audio. Where the class has genuinely
earned its keep: REFACTOR INVARIANCE (pipelined tick validated by
identical DIR tallies; FL_HASH bit-determinism is the sharpest tool
for pure refactors).

Queued improvement (fold into the pre-itch determinism-audit item):
each battery emits a final self-judging PASS/FAIL line against
stored thresholds and a nonzero exit code, so a run is a one-command
answer; prune or fix the stale batteries while at it. Scope when the
owner says go.

## Merge

Cleared: batteries green, build + clippy clean at all 17 commits,
every audio system owner-approved, controls exercised through a
month of live play. Merge via PR #4; the local branch gets deleted
after (owner permission, as always). Next thread after merge:
scenario format (tmp/handoffs/HANDOFF-scenario-format-2026-07-30.md)
toward itch 0.1.0; death-pool regen rides its own later branch.
