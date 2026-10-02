# 0175: main on Linux at 2K maximized, and the go for 0.2.1

Written by Claude Opus 5.5.

2026-10-02 evening, Linux box: RTX 3090, Ryzen 9 3900X, display 2560x1440 at 144 Hz. Main 48b3e0c (PRs #19 to #23 from the Windows days, devlogs 0168 to 0174), `target/opt-dev/flanks` built at 22:51. Gota played the 200k battle in a maximized window, after three days of perf work on Windows.

## What Gota saw

Very smooth even maximized at 200k. In Gota's words, "genuinely enjoyable seeing all these units being simulated this smooth". Linux may run a little slower than Windows, but the difference does not show in play.

Readings from the fps counter, by eye, during play:

| View | fps |
|---|---|
| The usual spot | 140 to 150 |
| Mid-air from the usual spot (two scroll steps out) | 139 to 141 |
| Mid-air, right end of the front | 136 to 139 |
| Mid-air, near the left end | 129 to 133 |
| Mid-air from the usual spot, enemy army all melee | 128 to 135, dips to about 124 |

The enemy's army makeup matters. With archers on the enemy side the lines stay apart. Against an all-melee army, everyone ends up packed around the front, and that run was the heaviest of the session.

For scale, the quiet Windows runs of devlog 0174 read 146 to 165 at 200k maximized in the four benchmark views. Those used the scripted test front or a saved battle with nothing else drawing on the GPU, so they are not the same measurement as the readings above.

## Decision

Release 0.2.1 today, after one test on the Mac. Gota: "This is something I can confidently show off and recommend to people, and with enough polish, as a paid game."

The remaining steps follow `work/handoffs/HANDOFF-windows-to-linux-2026-10-02.md`:

1. One macOS launch, zoomed out (the far levels' atlas coordinate from #20 and the near-to-far order from #23 have not been seen on Metal yet).
2. On Gota's word: the version bump commit, the tag, the AppImage built in a fresh clone, the release.
3. On Windows: the `embed_assets` exe from the tagged commit, uploaded to the same release, then the X post.

## Open

- Why the all-melee battle is heavier: more soldiers drawn close to the camera at the detailed levels (GPU), or more contact pairs in the sim (CPU). The periodic log's pass statistics and tick times from one such run would tell which.
- The left end of the mid-air view reads lower than the right end (129 to 133 against 136 to 139).

## Records brought over from Windows

The Windows records on D (`/data_hdd2/ggando/gamedev/flanks`) were copied into this tree, none overwriting an existing file: devlogs 0168 to 0174, eight handoffs, six PR drafts, `work/notes/vis/terrain-earth-layer-skip.png`, `tmp/flanks_perf0.png` to `flanks_perf3.png` and `tmp/x-post-draft-0.2.0.md`.

- Two devlogs carry 0168: `0168-packaging-and-icon.md` (Linux) and `0168-windows-fps-is-the-window-size.md` (Windows). Both kept, since the Windows records cite 0168 to 0174.
- The Windows plan `017-release-0.2.1.md` is `docs/plans/019-release-0.2.1.md` here, because 017 and 018 were already taken. The plans index and the windows-to-linux handoff point to 019.
- `docs/internal/005-release-packaging.md` merges both sides: the Windows single-exe build and its test from D, and the fat LTO line and "Keeping the release binaries" from Linux. The release file names there now match GitHub (`flanks_<version>.exe`, `flanks_<version>_macos_arm64.dmg`).
- Not copied: `demo_recs/` (35 GB, stays on D), and everything on C: (memories, the Windows clone's `tmp\` tools and benchmark setup).
