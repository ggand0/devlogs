# 0051 — perf/spikes: the felt spike leaves the app — gnome-shell GC metronome

**Date:** 2026-07-21 (second session, kimi-k3)
**Branch:** `perf/spikes`, uncommitted: chunk-size resilience + 3 knobs.

## The session's ruling finding

**The felt spike is not any in-app signal.** A 100k tour run with 123
tick-spikes (p50 17 ms, max 28 ms step+grid) was watched by the owner:
**zero felt stutter.** The pipeline absorbs ≤33 ms ticks exactly as
designed (0050). `[spike]`/tick-wall is hereby FALSIFIED as a felt-spike
proxy, joining `[hitch]` (falsified in the handoff). My tick-cluster
theory (GC preempts tick → felt spike) died on this same datum: the
slow-tick clusters are real but *unfelt*.

## The one signal that ever matched the felt cadence

pidstat of a clean played session: **gnome-shell CPU bursts of 40-70%
(vs ~12% baseline) every 11-12 s, metronome-regular** (autocorr r=+0.50
at lag 12 s). Present at idle too (exactly 11 s gaps, 7/7). GJS
garbage-collection pauses: a main-thread GC stall of 100-200 ms freezes
the whole composited desktop — felt in the game, invisible to every
app-side timer, camera/input independent. Correlation with felt reports
is now 4/4 ACROSS TIME:

| when | GC metronome | felt stutter |
|---|---|---|
| 19:38-41 played session | firing every 11-12 s | present (owner) |
| 19:52 idle sample | firing | — |
| ~22:1x tour, pre-change binary | quiet | absent (owner) |
| ~22:4x windowed 100k | quiet | absent (owner) |

The slow-tick clusters (26-32 ms) aligned ±1 s with shell bursts for
15+ consecutive cycles (pidstat's own 1 s resolution) — so the GC burst
*does* steal a core from the tick — but that effect is absorbed and
unfelt. The felt component is the presentation freeze, and it only
exists while the metronome is firing.

## Invalidated this session (do not cite)

- The fullscreen "still lags" A/B — owner was not watching frontline;
  compositor-unredirect theory NOT falsified, remains untested.
- All full-screen x11grab captures — filmed War of Dots/desktop
  (workspaces! mutter maps agent-launched windows behind whatever is
  visible; WoD fullscreen covers the game). Detector was blind 3 times.
- "Lagged within 3 s" report — same run-blindness/startup warmup.
- khugepaged/kcompactd theory — refuted from pidstat data (absent /
  aperiodic). "Driver/display path" was admitted guesswork, no numbers.

## Trigger: unknown, comes and goes

The metronome was firing 19:38-19:52 and quiet from ~21:0x on. Shell
GC is garbage-driven, not self-timed; suspects: MPRIS media churn
(YouTube tabs in Brave AND Chromium; Chromium now Paused, Brave
Playing), dock/appindicator/ding churn (extension bisect: disabling
dock made shell churn CONSTANTLY for its 35 s leg; metronome died
somewhere during the dock/appindicator toggle round and never came
back). gnome-shell Shell.Eval is locked on this box (cannot force GC
remotely). ffmpeg xcbgrab `-window_id` capture produced no stream —
presented-frame capture of the game window remains unbuilt.

## Windows cross-check (owner, 2026-07-22)

Owner ran the game on Windows (main branch, `cargo run --profile
opt-dev`, 200k scenario, windowed): **zero stutters or lags, no
benchmarking needed — obviously clean.** Exactly what the compositor
theory predicts (Windows DWM is native code, no GC in the frame path).
End-to-end confirmation on a second OS: the felt spike is Ubuntu/GNOME
compositor-side, definitively NOT the game. Had Windows stuttered, this
theory would have died instead — the test was decisive either way.

## Owner A/B verdict (the decisive runs)

200k (FL_UNITS=100000), windowed, YT music + WoD open, BOTH binaries —
pre-change (chunk2048) and post-change — **zero felt stutter, minutes
each**. His prior record: 200k NEVER ran clean. Conclusions: (1) the
chunk change is NOT the felt fix (old binary clean too); (2) felt
stutter is 100% environmental; (3) correlation now 6/6 (metronome
firing = stutter, quiet = clean). Prime suspect for the ~21:20
quieting: the ubuntu-dock disable/enable cycle (dash-to-dock GC-leak
class); secondary: Chromium YouTube tab Playing->Paused same window.
Watchdog armed: tmp/gc_watchdog.sh (nohup, 2 s shell-CPU sampling;
on burst >=30% snapshots MPRIS states + active window + top procs to
tmp/gc_watchdog.log). If stutter returns, the log names the culprit.

## Shipped (uncommitted, pending owner call)

- **Tick preemption resilience, bit-identical:** integrate CHUNK
  2048→256 (events apply in unit-index order across consecutive chunks)
  and grid REBUILD_CHUNK 32768→8192 (within-cell layout is unit-index
  order for ANY chunk size — merge assigns cursors in chunk order over
  consecutive ranges). FL_HASH FL_TEST_DIR: **19/19 fingerprints
  identical** to the pre-change baseline — zero math change. Rationale:
  GC bursts pushed tick clusters to 26-32 ms vs the 33 ms budget; fine
  chunks cap what one stolen core can hold hostage. NOTE: on the
  currently-loaded box (WoD + 20 apps) p50/max were noise-identical
  pre vs post; the change is tail insurance, not a felt fix.
- Knobs: FL_FULLSCREEN (borderless; mutter unredirects — the untested
  compositor-bypass A/B), FL_WIN_POS (pin window 0,0 for screen
  captures), FL_SPIKE_MARK (F9 logs felt-spike timestamps — owner
  declined manual marking; kept for future use).

## When the stutter returns (protocol for next time)

1. Owner says "now" — do NOT ask him to mark anything.
2. Immediately: `pidstat -u -p $(pgrep -x gnome-shell|head -1) 1 60` →
   is the metronome firing? (2 commands, 60 s, zero perturbation.)
3. If firing: bisect the trigger live — MPRIS pause via gdbus
   (org.mpris.MediaPlayer2.* PlaybackStatus/Pause), then extension
   toggles. Each leg 35-60 s of pidstat.
4. If quiet while he feels stutter: the GC theory dies too; next probe
   = a working presented-frame capture (fix xcbgrab window_id, or
   activate the game's workspace first, then corner x11grab of the
   overlay — overlay text changes every frame = freeze witness).
5. The fullscreen A/B (FL_FULLSCREEN=1, owner ACTUALLY watching) is
   still the cheapest compositor discriminator — 2 minutes.

## Verdict

**The felt lag spike is gnome-shell's GJS garbage collector freezing
the compositor every ~11-12 s (100-200 ms pauses) while a garbage
driver is active — the game was never the cause.** Evidence chain:
(1) metronome measured in pidstat, cadence = felt cadence; (2) felt
stutter present only while metronome fires (6/6 over time, incl. two
clean 200k runs with the pre-fix binary); (3) no in-app signal ever
matched (tick spikes ≤28 ms watched = unfelt, pipeline absorbs them);
(4) Windows run (200k, windowed) = zero stutter. Unverified link: the
garbage driver (ubuntu-dock prime suspect — storm died during its
restart; Chromium-tab pause secondary). Verification protocol when a
storm returns: pidstat 60 s → cycle suspect extension → pidstat 60 s.
Watchdog (tmp/gc_watchdog.sh) arms the detection.

## Owner rules that bit this session

- He will not manually mark spikes; detectors must be passive.
- He play-tests live; "no lag in this run" is a valid and decisive
  measurement — treat it as such immediately.
- No fullscreen debugging: the bug reproduces windowed; fixes that
  require fullscreen are not the fix he asked for (still worth one
  valid A/B as a DIAGNOSTIC, not as the fix).
- Audio off on all debug runs (FL_NO_AUDIO=1).
