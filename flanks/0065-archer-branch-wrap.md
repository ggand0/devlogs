# 0065: archer branch wrap

Date: 2026-07-29. Final polish round on feat/archers before the PR; owner verdict on the 20k battle: "feels like m2tw".

## Landed since 0064

- Firing indicator (1616d7a): amber-tinted mini bow icon in the archer card's top-right while the regiment holds a live fire solution. Driven by `GroupData::firing`, written each tick from the step_sim fire-solution precompute (gated on ammo_left too); hidden for broken/dead regiments.
- War-cry fix, two rounds (ff08317, 25c7ce4): archers ordered onto a close target roared like a melee charge. Round 1 gated the cry edge and the `charging` flag on `firing` — still cried ONCE sometimes: the cry edge evaluates on the frame the order lands, while `firing` lags a sim tick. Round 2 gates both on `ammo_left > 0` (intent-level truth, available the instant the order exists). LESSON for cross-schedule flags: anything a per-frame Update system edge-detects must not depend on a FixedUpdate-written flag being fresh. The empty-quiver knife charge still cries and charges — deliberate.
- Per-arrow fly loops went live once the owner generated `sfx_arrow_fly_loop_01/02` (seamless loops). Full archer audio now: budgeted loose/impact/ground one-shots at volley mass, attached fly loops (M2TW ARROW_FLY: prob .3, fadeout at the arrow's own impact), flyby whooshes per falling shaft near the camera, MAX_LIVE_ONE_SHOTS=64 mixer guard. Volley sheet cues stay benched (VOLLEY_SHEETS_ON=false); M2TW's own loose sound is distance-tiered per-unit group clips — the authentic re-enable path if wanted later.
- Refactor pass: `arrow_cloud()` helper dedupes the live-pool mean that combat_one_shots (death-scream gate) and archer_one_shots both computed inline. Nothing else worth extracting — the ranged loose stays inline in step_sim's integrate arm by the same convention as the melee machine.

## Battery check under the HP revert

Absolute numbers shifted with the c0ecffb revert (expected); the acceptances are ratio/shape-based and hold:

- FL_TEST_DIR: rear 51.6 dmg/hit vs front 23.2 (2.2x), rear kill bucket 267 vs front 255 with HALF the feeding regiments — per-attacker rear rate >2x frontal, acceptance met.
- FL_TEST_CHARGE: wall lane holds (dz -0.0 m vs -0.4 m unbraced at contact), wall kills chargers first, staggers flow on both lanes without stunlock.

## Branch summary (23 commits)

Evidence (0060) -> design doc (docs/plans/006-archers.md, local) -> KIND_ARCHER + arrow SoA pool + orders/HUD (0061) -> no-fire engaged-flag fix + art rounds (0062-0063, kimi k3 landed the Sherwood hooded longbowman) -> HP-placeholder revert + damage calibration vs the Withwnar anchor + per-team cloth + fixed-2-regiments + AI soft-targeting + terrain LOS + arrow stagger + volley screams (0064) -> audio suite, indicator, war-cry gating (0065). PR draft: work/drafts/pr-archers.md.

Deferred to future branches (owner-directed): balance iteration from playtests, AI positioning (screen archers behind the line), flaming arrows, plaque ammo readout, stakes (with cavalry), META kind-field widen for a 5th kind (longbow/crossbow elite — the AP counter to arrow-proof plate).

## Post-PR final check (2026-07-29)

Full-diff review against main after the PR opened. Found and fixed: the two sfx_arrow_fly_loop clips were generated after the sound-set commit and never staged (fresh clones had a silent flight layer) — committed; README had no archer feature bullet and no T/K rows in the controls table — added. No debug leftovers, no workflow tells in the diff.

STALE BATTERY, not an archer-branch regression (owner correction — the first write-up here blamed the HP revert): FL_TEST_ROUT's lone blue regiment fights to ~-7 and never reaches the -11 rout band. The battery predates the MORALE REWORK (PR #10): the rework's leadership term makes the test's ONLY blue regiment the army's command regiment, and the measured general's-own-regiment bonus (+8 self, +2 army) props it above the band once casualties (-12) and flank (-6) cap out. Arithmetic for an ORDINARY regiment in the same grinder: 5 + 2 - 12 - 6 - 1.5 = -12.5 < -11, routs fine — which is why routs work in real battles (owner's 20k pass). The lone-regiment-is-the-bodyguard setup is the artifact; M2TW's bodyguard IS the stubborn unit. Battery fix for a future branch: spawn a decoy command regiment away from the fight (leader assignment picks the first heavy, else first alive — morale.rs ~line 280) so the battery tests the ordinary rout path again. SURROUND/DIR/CHARGE acceptances hold.

BISECTION CONFIRMED (owner asked for proof): pre-rework cb95895 routs at 491/1000 alive (the historical "ROUT 53%" green); post-rework main 876471d fights to 16 alive and never breaks; the archer branch behaves identically to main. The battery went stale at the morale rework merge, full stop. Bisect trap for next time: commits before the project rename build a binary named `frontline`, not `flanks` — running the stale `flanks` path silently re-runs the previous build.
