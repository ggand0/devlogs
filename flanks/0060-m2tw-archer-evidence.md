# 0060: M2TW archer evidence

Date: 2026-07-26. Research pass before the archer design (owner mandate: M2TW-evidenced only). Raw fetched sources are cached in `tmp/archer-evidence/` (vanilla EDU, descr_projectile.txt, Withwnar's tested archery guide and accuracy test, Feral manual, TWC wiki EDU docs, M2TWEOP engine structs). Everything below is VERIFIED against a quoted config line, the official manual, or a controlled in-game test unless marked FOLKLORE.

## EDU stats (vanilla 1.5, verbatim lines in tmp/archer-evidence/edu_vanilla.txt)

| unit | missile atk | missile | range | ammo | melee atk | armour/skill/shield | morale | mass | ap |
|---|---|---|---|---|---|---|---|---|---|
| Peasant Archers / Archer Militia | 5 | arrow | 120 | 30 | 2 | 0/1/0 | 3 untrained | 0.8 | no |
| Longbowmen | 6 | bodkin_arrow | 160 | 30 | 7 (mace) | 0/1/3 | 3 untrained | 0.8 | yes |
| Yeoman Archers | 8 | bodkin_arrow | 160 | 30 | 9 (mace) | 0/2/3 | 5 trained | 1.0 | yes |
| Turkish Archers (composite) | 7 | composite_arrow | 160 | 30 | 6 | 4/3/3 | 3 untrained | 0.8 | no |

- Range unit is METERS: CA's own comment at the top of descr_projectile.txt derives velocity from `d = v^2 / g` with g = 9.81. So basic archers shoot 120 m, longbow and composite 160 m.
- Ammo is 30 per man for every vanilla foot archer (25 mounted). Out of ammo means melee with the (weak) secondary weapon, nothing else.
- For missile units the SECONDARY weapon is the melee weapon (wiki, verbatim).

## Projectile physics (descr_projectile.txt, verbatim)

- `arrow` / `bodkin_arrow` / `composite_arrow`: `velocity 20 48` (m/s band the engine solves the arc within), `min_angle -75 / max_angle 65`, `mass 0.05`, `damage 0` (kill chance comes from the EDU missile attack, arrows carry no intrinsic damage; only artillery has damage), `affected_by_rain`, NOT body-piercing (one victim per arrow).
- Plain projectile ballistics at g = 9.81; flat shot when clear, lofted to clear obstacles/friendlies (lofted is less effective, Withwnar).
- Accuracy: `accuracy_vs_units 0.05` for arrow; the "hyper accurate" bodkin/composite lines (`0.00001` etc.) are COMMENTED OUT in vanilla and fall back to an engine default that Withwnar's controlled test pinned near 0.05. So all vanilla bows are equally (in)accurate.
- Accuracy does NOT improve with proximity to the target (Ecthelion, tested: "makes absolutely no difference"). Scatter is effectively range-independent.
- Rain/thunderstorm degrade accuracy (tested: clear 66 kills vs rain 50 vs storm 37 over fixed volleys); fog and night change nothing.

## Firing behavior

- Fire-at-will: official manual, "will fire at any nearby targets without being specifically ordered". Right-click gives a targeted order on one enemy unit.
- No engine-level volley sync: per-soldier aim targets and firing state (M2TWEOP structs); volleys look loosely synchronized because soldiers share the animation cycle.
- Fire rate is ANIMATION-BOUND: EDU min-delay (25 = 2.5 s) and `stat_fire_delay` barely change the real rate (tested by modders; only re-timing the firing animation works). Real cycle is ~10 s per volley for longbowmen, "6 volleys per minute" (FOLKLORE tier but consistent).
- All ranks fire; no rank-limit parameter exists anywhere (verified absence). Default archer formation is 4 ranks.
- No minimum range parameter exists (verified absence); close-in behavior falls out of the angle limits and melee engagement.
- Foot archers must HALT to shoot; only missile cavalry fires on the move (corroborated folklore; no config governs it). Cantabrian circle is cavalry-only (manual).
- High ground: manual, hills give archers "a range bonus and a clearer shot", which is just the ballistics.

## Friendly fire and shields

- Friendly fire is real and unmodified: arrows hit whatever they land on, there is no friendly-fire damage modifier anywhere (verified absence). Withwnar: "when flanking you will still inevitably hit your own troops".
- Shields work against arrows directionally: full from the front, 50% weaker on the sides, nothing from the rear; armour counts from all directions (Withwnar, tested).

## Skirmish

- Manual, verbatim: skirmish mode "will try to keep a safe distance between itself and the enemy (usually the range of its missile weapons)". No numeric trigger distance documented anywhere.

## Flaming arrows (deferred)

- `delay flaming 15.0` (vs 0.0 standard) plus worse accuracy (0.07) explains the ~3x slower, weaker-killing flaming volleys; manual confirms extra fire damage on connect and an explicit morale penalty for being under flaming fire. Skipping for the first archer milestone.

## Folklore corrections worth remembering

1. Bodkin/composite are NOT more accurate in vanilla (dead config lines).
2. Accuracy is range-independent.
3. `stat_fire_delay` does nothing useful; rate lives in the animation.
