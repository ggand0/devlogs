# 0061: archers and the arrow projectile system

Date: 2026-07-27. Branch feat/archers, plan docs/plans/archers.md, evidence devlog 0060 (+ tmp/archer-evidence/). Owner calls locked before impl: plain archers first (120 m, no AP; longbow is a stat promotion later), hood look (helmet variant possible later), damage anchor ~5% casualties per 4-volley exchange, datamine if needed.

## What went in

- KIND_ARCHER = 3 (unit_types.rs): fills the spatial 2-bit kind field EXACTLY. Melee row is a weak knife fallback (hp 120, atk 4, armour 1, no shield, morale 3, mass 0.8, EDU-scaled); ranged stats live in `unit_types::missile` (attack 6, range 120 m, ammo 30, draw 1.5 s, reload 8 s, launch 20..48 m/s, g 9.81, scatter sigma 3 m range-independent, BASE_DMG 12 = the calibration knob).
- `Units.ammo` new SoA column (u8, swap-removed in process_deaths like every column).
- arrows.rs: SoA arrow pool (cap 65k) + two instanced buckets riding the EXISTING unit pipeline (any entity with InstanceMaterialData is queued, so no pipeline duplication; arrow orientation = yaw in anim.x + pitch in anim2.z via a new `part > 6.5` shader branch). Tick: integrate under gravity, collision only inside the ground strike band, segment-vs-body test through the spatial grid, damage applied where the shaft lands (armour always, shield front+left, NO defence skill, friendly fire real with no modifier), kills feed recent_deaths/recent_kills so morale sees arrow casualties. Ground hits transfer into a StuckArrows ring (FL_ARROW_LITTER, default 50k) and stay planted at the flight angle. Ordering: after step_sim (fresh grid, stable indices), before process_deaths.
- Loose cycle rides the existing swing machine via a SWING_RANGED bit (bit 7): READY archer + regiment fire solution + standing still -> WINDUP(draw, 45 ticks + jitter) -> loose (arc solved, ammo--, RECOVER 192..252 ticks). Per-soldier staggered timers give ragged volleys for free (M2TW has no engine volley sync either). A reloading archer contacted in melee drops the reload to knife tempo.
- Fire solutions are regiment-level, precomputed per tick in step_sim: attack order = the target once inside range (stand-off halts the regiment at 0.85 x range with the same anchor-snap engagement uses); no order = fire-at-will at the nearest live enemy block in range; Move orders and melee engagement silence the bows (foot archers halt to shoot, M2TW-evidenced).
- Arc solve (arrows::solve_launch): flat low-root shot at full draw when the path clears every friendly regiment disc en route, else the slowest lofted arc that reaches (~45-65 deg lob, matching M2TW's max_angle 65). A shooter's own regiment disc is in the block list, so rear ranks loft over their own front rank with no special rule. High ground = longer reach for free. Aim = footprint sample + range-independent scatter + centroid-velocity lead (RegTracks EMA).
- Skirmish (arrows::skirmish_and_ammo): manual-evidenced "keep a safe distance"; formed enemy inside 35 m -> auto Move directly away until 55 m reopens, then stand, dress, resume. Player orders always win; hold blocks it. Default ON for archers, toggleable.
- HUD: Fire at Will (T) and Skirmish (K) buttons + hotkeys through the same FormCmd pipe, disabled/active states over the archer subset of the selection; ammo bar (straw strip under fatigue) on archer cards from the per-tick regiment ammo tally; A letter + bow icon; slate-blue card fill; guidon flag mesh; overlay names "Archers"; wall_kind = none for archers.
- Spawner: FL_ARCHER_FRAC (default 0.2) archer regiments in the REAR ranks; heavies/spears/lights unchanged. AI: auto_engage never melee-charges archers (fire-at-will engages for them; skirmish handles proximity).
- Mesh: build_archer (22 cuboids): gambeson + team tabard, bare face in a team HOOD (deliberately no steel next to the kettle-hat kinds), back quiver with fletched shafts, vertical bow in the left hand on new PART_BOW_ARM = 6 (draw tilts arm+stave toward the loft angle; the right hand is plain PART_ARM whose stab pull-back IS the string draw). Arrow mesh: 3 cuboids, slightly oversized for zoom readability.

## Deviations from the plan doc

- No lead iteration on individual soldiers: aim is footprint + centroid lead (scatter dominates at 3 m sigma; revisit if archers feel blind vs fast movers).
- Audio deferred (needs ElevenLabs assets): ArrowStats.loosed/landed counters are in place for the budgeted one-shot pattern.
- Overlay inspect plaque doesn't show ammo yet (unit-card bar does).

## Verify (owner playtest)

- `FL_ARCHER_FRAC=1 FL_ENEMY_STATIC=1 FL_ARMY_GAP=300` arcs, stand-off, litter.
- `FL_ARENA=1` is heavies-only; calibration check is a normal small battle (FL_UNITS=4000, FL_AI=0) watching a light regiment under 4 volleys (~5% target).
- Skirmish: walk melee at an idle archer block, expect the back-step at ~35 m, re-stand at ~55 m.
- Perf: watch step_ms/update_arrows spans at 200k with default 20% archers; live arrows steady-state estimate ~16k, pool cap 65k, dropped counter should stay 0.

Build + clippy clean. FL_TEST_* batteries untouched by construction (scripted scenarios spawn no archers), but should be re-run before merge.
