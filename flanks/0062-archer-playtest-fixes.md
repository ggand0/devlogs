# 0062: archer playtest round 1 fixes

Date: 2026-07-27. Owner feedback on febfc22: archers never fired (zero enemy casualties, no arrows visible) and the art missed (mesh "bomberman/peasant", hood needs to be a proper ranger hood, bow "a stick", card icon reads crossbow).

## The no-fire bug: one flag, all three symptoms

Diagnosed from [archery] scenario logs (arrows-in-flight stuck at 0 while ammo drained, blue archers 500 -> 90). The ammo drain was EXACTLY 30 x deaths — zero arrows were ever loosed; the archers died in plain melee.

Root cause: frontline.rs's ground-truth contact check counts any soldier in SWING_WINDUP as fighting, and a bow DRAW is a wind-up. The first draw marked the archer regiment ENGAGED, the fire-solution precompute (step_sim) treats engaged as "in melee -> no volleys", so every draw aborted before its loose. The false engaged flag also disabled skirmish (engaged regiments don't back-step), which is why the demo attacker walked straight in and slaughtered them. Fix: the fighting[] check now excludes SWING_RANGED wind-ups.

Two more real fixes uncovered by the same investigation:

- Self-hit at launch (latent, masked by the no-fire bug): arrows launched at +0.55 over mid-body, INSIDE the body-hit band, with the shooter at lateral distance zero at flight-time zero. Launch is now +0.75 (above every kind's hit ceiling) and the body-band top margin shrank 0.15 -> 0.05.
- Skirmish triggered on CENTROID distance (35 m) — two 500-man blocks' fronts touch at ~30 m of centroid separation, so the back-step started with swords already arriving. Trigger is now the EDGE gap (centroid distance minus both footprint radii), 22 m trigger / 40 m reopen.

Verified live (FL_TEST_ARCHERY + import screenshots): first volley 171 arrows in flight at t=12, second 337, orange bleeds from range, the demo attacker breaks and routs off the field from arrow fire + the screen fight. Arrows render as pale streaks arcing over the field; flying arrows now draw at 1.35x scale so a volley reads at battle zoom (ground litter stays true-scale).

## Art round 2

- Mesh rebuilt as a RANGER (32 cuboids): deep pointed hood swept back in four tiers with a drooped tip, jutting brim with a cowl-shadow block swallowing the upper face, shoulder-mantled cloak falling to the thighs (team color pulled toward black — CLOAK blends 80% team over near-black so armies still read), gambeson + team tabard under it, leather bracer on the bow arm. The bow is a real longbow now: thick grip, mid limbs and recurved nocks each stepping forward to draw the curve, pale linen string tip to tip, ~1.06 m tall.
- Card icon redrawn as a longbow at FULL DRAW, arrow pointing left: stave is a 6-segment arc, string a V pulled to the nock, broadhead + fletching on the shaft. Unmistakably a bow.

## Process note

The screenshot loop that cracked this: FL_TEST_ARCHERY=1 in the background, `xdotool search --name '^flanks$'` + `import -window` timed into the volley window, read the PNG. Logs alone had me chasing a friendly-fire ghost; one screenshot of the standing-but-silent archers plus the exact 30-per-death ammo arithmetic found the engaged flag.

Owner feel pass still pending on the new look; damage calibration (missile::BASE_DMG = 12) untouched this round.
