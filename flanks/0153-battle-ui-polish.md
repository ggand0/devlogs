# Battle UI polish (feat/ui-polish)

Written by Claude Opus 5.5.

2026-09-27. A UI branch before 0.2.0, off main at 57d8090 (PR #12). Everything here is presentation: no sim change, and the pile-on scenario spawns the same regiments in the same order as before.

## What was built

- **Settings tabs.** General (audio, camera, video), Interface (Battle HUD, Unit panel, Debug overlay, Hit flash, Front line) and Controls (drag select and every key binding, in two columns). Each tab is one column at a fixed body size, so switching tabs never resizes the panel. The last tab opened is remembered.
- **F1 to F3.** F1 hides the battle HUD (card bar, control panel, balance bar), F2 the unit panel, F3 shows the debug overlay. Each key flips the saved setting, so the key and the Settings row are one state. The debug overlay now defaults off; the periodic fps log runs either way. With the HUD hidden, a hidden node takes no pointer input, so the map is clickable where the bar was.
- **Debug tools behind F3.** The X crater tool only works with the overlay on. The unit panel's morale level, formation line and factor breakdown only show with it.
- **Hit flash toggle.** Off packs every flash as zero, in both render paths.
- **Front line toggle.** Hides the front line gizmo; G still hides it (and the banners, move markers and lasso stroke) as before. What else G should cover is undecided.
- **Unit panel after M2TW's** (resources/ui/m2tw_unit_panel.png): army in the team colour, `name [class] (men)`, the activity, the morale word, the fatigue word. No border, no icons.
- **Title screen.** Tagline "Hold the line. Turn the flank." Version from Cargo.toml. The test scenarios sit behind one small Test Battles button that opens a panel; Wide vs Narrow and Two on One (the footwork scenarios of `pile-wide.sh` and `pile-two.sh`) join them. Their setup is a `PileSetup` passed from the scenario; the FL_PILE_* envs still override it.
- **Balance of power bar** (top center): the player's share of the men still fighting (alive, regiment not broken), muted slate blue against burnt orange in the unit panel's dark glass. Hover shows each side's losses and M2TW's verdict line. The deployment banner moved below it.
- **Selection rings** replace the body tint: see below.

## M2TW's own words

The display strings come from M2TW's `battle.txt.strings.bin` (localized.pack, unpacked with the official unpacker under Proton 8 wine; decoder in the session scratchpad, format: u32 header, u32 count, then u16-length UTF-16 strings).

- Morale: Heroic, Impetuous, Eager, Steady, Shaken, Wavering, Broken. Seven words for the engine's seven states (berserk, impetuous, high, firm, shaken, wavering, routing, devlog 0056), in the same order.
- Fatigue: Fresh, Warmed up, Winded, Tired, Very tired, Exhausted (ours were lower case).
- Activity: Idle, Hiding, Ready, Reforming, Marching, Firing missiles, Reloading, Charging, Fighting, Running amok, Routing, Fighting to the death, Pursuing, Dead, Left battle, Waiting to enter battle, Taunting.
- Classes: Heavy Infantry, Light Infantry, Skirmishers, Spearmen, Missile, Heavy Cavalry, Light Cavalry, Skirmisher Cavalry.
- Balance: seven verdicts from "Victory seems certain. Only a fool could lose this battle" to "Defeat seems certain. Only a military genius could win this battle", and "Percentage Allies Killed:" / "Percentage Enemies Killed:".

Eager starts at +12, the measured median of `high` (devlog 0057), the way Shaken and Wavering start at their states' medians. Impetuous and berserk never occurred in the captures, so they get no band. `band()` is unchanged (the render stance reads it); the word is a separate `state_word()`.

Our mapping: Knights [Heavy Infantry], Men-at-Arms [Light Infantry], Spearmen [Spearmen], Bowmen [Missile]. The panel and the picker now share `unit_types::kind_name`; the panel used to say Heavy Knights and Archers.

The balance verdict is the seventh of the bar the split falls in. M2TW's own rule for it is unknown.

## Selection rings

A see-through disc with a brighter rim and a facing notch, under every living, visible soldier of a selected regiment (the move preview's slot green, same 0.42 m radius as the preview's circles), the player's regiment under the cursor or its card (the same green, fainter) and the enemy under the cursor (red, the old hostile tint's hue). Broken regiments keep the gray body and get no selection ring.

- **GPU path.** The build pass appends each ringed soldier to a ring list in the tail of the index list (record index, style, kind in one u32), and `finalize` turns the count into one more indirect draw argument. One draw of six corners per ring reads the soldier's interpolated record, so a ring never trails its soldier. No per-soldier CPU work.
- **CPU path.** The sweep collects the same entries with their records; they are uploaded and drawn directly. Same shader.
- **On the ground.** Each corner samples an uploaded copy of the height field (513 x 385 floats), re-sent when the terrain changed (craters) while rings show. When the soldier's feet are more than 0.3 m off the field (the bridge deck) the ring lies flat at his feet. The lift above the ground grows with distance.
- **Order.** Drawn in the Transparent3d phase, depth tested, never written, so feet stand over their ring. The item is added per frame (`add_transient`) only while something is ringed.

## Checks

- Build and clippy clean.
- GPU path, 20k, army selected with `FL_SELECT_ALL=1` (new knob, selects the player's army at battle start for screenshots), camera locked at 22 m: rings under every soldier, notches toward the enemy, feet over the rings (tmp/runs/rings/close_14s.png).
- GPU path, default camera at 280 m: each selected block reads as a green field under its banner, as M2TW's far view does (tmp/runs/rings/wide_10s.png).
- CPU path (`FL_GPU_SYNC=0`), same close camera: rings drawn the same way. The window opened under the mouse and a real click reselected one regiment 13 ms after FL_SELECT_ALL, so only that block is ringed; the unit panel shows it as "Blue / Spearmen [Spearmen] (500) / Idle / Eager / Fresh" (tmp/runs/rings/cpu_10s.png).
- Not checked here, input only: hover rings (own and enemy), F1 to F3, the tabs, the Test Battles panel, the balance tooltip. No fps run: with nothing ringed the build pass adds one flag test per soldier and no draw; the heaviest case is Ctrl+A at 200k (100k rings, 600k corners), for Gota's counter.

## Feedback round 1 (Gota)

- Rings too bright where formations are dense (reference resources/ui/etw_unit_selection0.jpg): the rim is gone; one even fill at 0.45 in a muted green (sRGB 0.42, 0.74, 0.46), hover red muted to (0.80, 0.30, 0.24); the notch became ETW's teardrop, the two tangent lines from a tip at 1.45 radii (f41dc27, tmp/runs/rings/close2_crop.png).
- Controls ran past the panel: every tab body scrolls with the wheel, with a thin thumb (3ebf85c).
- Banners cast shadows that grew with the zoom: flag and pole no longer cast (13e12b2).
- Move markers piled up for every regiment: only the selected regiments show theirs (5afe1dc).

## Engaged regiments ignore move orders (diagnosis, not fixed)

A regiment in melee has a fight point (frontline.rs:541) whatever its order: its target under an Attack order, else the nearest formed enemy, so under a Move order too. The men read "in melee" from the fight point (sim/soldier.rs:357); a man who swung once is out of formation, and `decide_to_join` zeroes his march for as long as the melee lasts (sim/soldier.rs:888). The men in reach keep fighting, which keeps the regiment engaged, which keeps the fight point: nobody leaves until the enemy dies or breaks. Came in with the footwork commits e7dc9e1 and 3f941b4 (2026-09-25, PR #9); the earlier design (devlog 0037) had "Move orders never freeze (withdrawal from melee)", and M2TW has a WITHDRAW unit task and an ACT_WITHDRAW soldier action (devlog 0120).

A new attack order on another regiment while fighting: the contact frame stays (frontline.rs:556 keeps it while attacking and engaged, whatever the target) and a regiment holding a frame gets no destination (sim/job.rs:454), so its slots stay at the old fight; men with nobody in reach jog to the new target, men in reach keep fighting the old one. Same stall.

## Move orders take a regiment out of melee (a2 of the diagnosis above)

Approved by Gota. Under a Move order a regiment has no fight point (frontline.rs), is not a pressing regiment for the far look (sim/job.rs `press`), and its men close on no enemy (`moving`, sim/soldier.rs `close_in`). They walk to their slots at the destination; the swing stage still lets a man strike an enemy in reach in front of him, and a wind-up slows him to a quarter pace, so a block walking into enemies fights its way through as far as the bodies allow (hard core 0.9 m, soft band to 1.4 m, formation pitch 1.4 m: a formed line is a wall, a loose or broken one lets men through). Everything under Attack, at ease and hold is unchanged. Skirmishers' auto withdrawal is a Move order and gets the same.

The scripted scenarios that walk attackers into the enemy with Move orders (DIR, Surround, Rout, Rout Pass) are now that push-through case. DIR smoke run, 32 s: no panic, per-hit damage front 22.1 / side 28.3 / rear 49.0 (devlog 0123: rear 48.5, front 20.6), test pair wiped at 20 s. Fingerprints will differ from the baselines; gate after Gota's play check.

Not done: a new attack order mid-fight still stalls (the contact frame holds whoever the enemy is, by design since devlog 0042).

## New attack target mid-fight (672937d) and fight point markers (80449f1)

Gota liked the Move behaviour and approved the retarget rule. A regiment whose attack order names a new regiment while it is engaged breaks off (`GroupData::retarget`): no fight point, no contact frame, no melee clock, and it counts as `moving` for its men, so the block marches on the new target and its men strike only whoever is in front of them. It ends when the men striking a man of the target reach the lock threshold (3% of the regiment, at least 4; new `target_fight_n` in the frontline sums) or the regiment disengages; then the footwork and the any-enemy frame apply again. The melee clock also stops under a Move order, so a regiment still engaged on arrival starts a fresh fight with its own join wave.

Check: 60 s AI battle at 20k without input: 36 engagements, 0 break-offs (the AI and at-ease engagement only order unengaged regiments), no panic. The break-off path itself is only reachable from player input.

F3 now also draws each selected regiment's fight point: a magenta ring on the ground joined to the regiment's centre, or a dashed line to the new target while it breaks off.

## Retarget round 2 (af8778b, test hook 89c853f)

Gota's play check: the target routed and the fight point jumped to a nearby formed enemy; and after a retarget half the regiment went to the new target while the half next to the old one kept fighting it (resources/combat/debug0.png).

- Routing target: the fight point used the ordered target only while unbroken, else the nearest formed enemy within watch range, while the order and the march kept chasing it. Men still cutting at routers kept the regiment engaged, so its men headed for the formed enemy; with none near, it pursued, hence "not always". The fight point now stays on the ordered target while it has men.
- The split: the break-off ended once the lock threshold of men struck the new target; the rest then fought anyone they saw. Now the break-off lasts while the order stands and the regiment is engaged. Before the melee with the target starts it is Move-like; after, `focus` restricts closing, the far look and the remembered enemy to the target's men, a man in reach of the target picks him over anyone else, and the join to the fight point goes on while he fights another enemy in his own defence (quarter pace in a wind-up). Only target strikers start that melee and lay its frame. With no focus every gate reduces to the old code.
- A first, strict version (no self-defence) left men standing while the old enemy cut them down: regiment 0 lost 144 men in 35 s, its old enemy 11.

`FL_TEST_RETARGET=35`, 20k AI battle, 8 regiments summed, old (80449f1 plus the hook) vs new; the AI battle is not deterministic, so indicative only:

| | 50 s | 60 s | 75 s |
|---|---|---|---|
| striking the old target, old | 68 | 58 | 35 |
| striking the old target, new | 41 | 29 | 21 |
| touching the old target (2.5 m), old / new at 75 s | | | 267 / 195 |

In the old version 7 of 8 had ended the break-off and fought anyone; in the new one the remaining contact is where the old enemy follows the regiment. Logs tmp/runs/rings/retarget-old.log and retarget-test3.log.

## Branch split

feat/ui-polish now ends at 978dde9 (the nine UI commits); fix/melee-orders carries those plus 3d99575, 672937d, 80449f1, af8778b and 89c853f, stacked, to be rebased onto main after the UI PR merges. Backup: branch backup/ui-polish-before-split and work/backups/ui-polish-before-split.bundle. The UI branch alone, built in a scratch worktree: clippy clean, fingerprint gate against main6 equal in all four scenarios (dir 19/19, arch 19/19, pilewide 29/29, pile2 29/29). PR draft: work/drafts/pr-ui-polish.md.

## Merged

feat/ui-polish merged as PR #13 (main = 0a41859), with 3aa78bc (comment fixes from the pre-PR audit) and 87ec8a6 (Blob button and its key removed, Hold moved to B). fix/melee-orders rebased onto it: 13217d2, dadd8c9, 8511c93, 5efb0e4, bc98607, clippy clean. Handoff for the next thread: work/handoffs/HANDOFF-melee-orders-2026-09-27.md.
