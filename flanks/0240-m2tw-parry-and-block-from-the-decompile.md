# 0240 M2TW parry and block, read from the decompile

Written by Claude Fable 5.1.

Date: 2026-10-12. Gota asked for M2TW's parry and block behaviour and a proposal for flanks (the proposal is `docs/plans/024-parry-and-block.md`). This entry is the research record. No code changed.

## Sources

- The verified decompile in `/home/gota/ggando/gamedev/m2tw-decompile`, `docs/BATTLE.md`, section 9 (combat resolution) and "The melee action". Every function cited there as *verified* was reimplemented and checked against the machine code in an emulator, so these are not forum theories.
- `descr_skeleton.txt` (the unpacked copy in `work/research/m2tw-extraction/data/`): the attack and defend clip pairs, their impact frames and flags.
- The animation pack (`assets_dev/_m2tw_refs/m2anim.py`, data in `/data/ggando/m2tw/data`): clip lengths.
- `strings medieval2.exe`: the reaction enum (`ART_*`) and the descr_skeleton flags (`-prob`, `-parry`, `-evade`).
- Web: the community formulas (attack over attack plus defence, the 1.2 per point kill curve of devlog 0031) are superseded by the decompile; the "a successful hit kills outright; parry is a weapon animation" summaries agree with it in outline. Sources: [TWC damage calculation](https://www.twcenter.net/threads/damage-calculation.245980/), [totalwar.org attack values](https://forums.totalwar.org/vb/showthread.php/128953-How-do-attack-values-work-Attack-lethality-armor-shield-etc), [Steam unit guide](https://steamcommunity.com/sharedfiles/filedetails/?id=909957531), [GameFAQs defence breakdown](https://gamefaqs.gamespot.com/boards/931592-medieval-ii-total-war/40948216).

## 1. There is no matching step

M2TW never pairs two soldiers into a shared duel. Each man runs his own melee action (`ACT_ATTACK_MELEE`) on a 0.1 s tick: pick a foe, swing, recover, react. The only synchronised pair is the *fatality*: when a blow kills and the dead man faced the attacker within 75 degrees, the two play a paired kill clip (6 to 9 s long) in sync. Everything else, blocks and parries included, is each man's own clip on his own clock. That is the difference from Warscape (ETW, NTW and later), where pairs lock into matched duel animations. Gota's preference for the M2TW way is also the cheaper one for a 200k sim: no pairing, no shared state.

## 2. The blow: one roll per strike

A swing lands on one frame of its animation, the impact frame (`-if:N` in `descr_skeleton.txt`, 13 to 26 at 20 fps, so 0.65 to 1.3 s into the clip). At that frame the game picks the man nearest the swing's line within reach and rolls once (`Combat_rollHit`):

- d = attack minus defence. Attack plus charge bonus minus armour (halved by armour piercing), then the sector: front (within 67.5 degrees) minus shield minus skill; shield side (left) minus shield minus half skill; open side (right) plus 5 minus half skill; rear plus 10 and armour only. Plus height, fatigue, formation, ground and general terms.
- chance per mille = 20d + 200 for d at or above minus 6 (80 at minus 6, capped 990), 10d + 140 down to minus 12, 4d + 68 below, floor 1. Scaled by the attacking unit's factor when not negative (a hypothesis: this is the EDU `stat_pri` last field, 1 for swords, 0.6 for spears, 1.33 for the axe of the Dismounted English Knights, 1.8 for the archers' knife), then by `combat-balancing/melee-hit-rate`, default 1.75.
- one roll r in 0 to 1000: r within the chance is a hit (one hit point; almost every man has one, so a hit is a kill); within 50 above it a knockdown; within 100 a knockback; else nothing.

At parity (d = 0) a blow kills 35 percent of the time, knocks down 5, knocks back 5, and does nothing 55 percent of the time. Each attack point is worth 3.5 percentage points near parity, half that from 6 behind, a fifth from 12 behind. The "nothing" outcome is what M2TW shows as a blocked or turned blow: the attacker plays the attack's *fail* clip, the victim his reaction. The reaction enum in the exe names the classes: `ART_HIT`, `ART_MISS`, `ART_DODGE`, `ART_FIENT`, `ART_DEFENDED_SHIELD`, `ART_DEFENDED_ARMOUR`, `ART_KNOCKBACK_LIGHT`, `ART_KNOCKBACK_HEAVY`, `ART_KNOCKDOWN`, `ART_STEPBACK`. So "shield block" and "armour turned it" are presentation classes of the one roll's failure, not separate mechanics.

What defence stats do, then: they lower the land chance. There is no damage scaling; a landed blow takes a hit point.

## 3. The active parry: a timed reaction that stops the blow outright

This is the mechanic Gota means by parry. From "The melee action" in `BATTLE.md` (functions `Melee_swungAt`, `Melee_scheduleReaction`, `Melee_reactionState`, `Melee_attackStage`, all verified):

1. When a man starts a swing, his target is **told** (slot 54, "swung at").
2. If the target can fight and **sees the attacker** (within 90 degrees of his facing), a reaction is scheduled to meet the attacker's strike frame. Its countdown is half the strike's key frame; it starts two frames later when a draw beats his readiness (his eagerness plus 1, so only a man drained below zero is ever late by this path).
3. The reaction clip is the answer to the attack: high blows get high parries (the attacker's strike height sorted into four classes), otherwise a weighted draw among up to seven answers. The reaction stage is entered with a counter that makes the answer's key frame meet the swing's strike frame, and, **unless a random draw falls below the answer's own skill value** (a percentage stored per clip; doubled in guard mode), two more ticks of delay: a late defence.
4. The man's **reaction state** is 0 while advancing or running, 1 while winding up his own swing, 6 after it and while recovering, 5 in a reaction until its key frame, and then a **defensive window** by the clip's kind: kind 0 gives 7 frames, kind 1 three then two, kind 2 five frames (frames are the 0.1 s ticks).
5. At the strike, when the swing has an alternate record, a target in the window **who faces the attacker within 45 degrees stops the blow outright**, whatever the stats. In state 2 the attacker's own swing is turned into its alternate (fail) clip. A kind 4 alternate turns the outcome into a very near miss instead.

So defence is a matter of timing animations. Three consequences:

- A man who is winding up, following through or recovering **cannot parry** (states 1 and 6). Attacking commits him. A man who is parrying is in a reaction for the clip's length and is not swinging. Outnumbering wins by tempo: the second attacker's blow finds the man already busy with the first.
- The clip's skill value sets how often the parry is on time. In `descr_skeleton.txt` the sword and shield set (`MTW2_Mace`, which `MTW2_Swordsman` inherits) lists twelve `eager_defend_*` clips: the slash defends carry `-evade -prob 10`, the stab defends and the right to left slashes carry no `-prob`. The exe parses `-prob`, `-parry` and `-evade` as flags. Whether a clip without `-prob` defaults to always on time or never is not read yet (one check: the descr_skeleton parser in the decompile).
- **Guard mode** is this: a tenth of the swings, and the on-time share doubled.

M2TW staggers (knockback) do not cancel a scheduled parry by themselves, but a knocked man goes to the recover stage (state 6) and spends eagerness, so he swings less and parries nothing until he is back.

## 4. The swing rate

M2TW has no cooldown. Each chance to attack (every tick while an enemy touches him, otherwise every 5 to 10 ticks) a man swings with probability 0.025 + 0.975 × quality × eagerness. Eagerness is a per-man reserve: a swing spends a random 0 to 1.2 of it, a knockback 0 to 0.35, and it comes back by 1/n per tick, n = (the EDU minimum delay, 25 deciseconds for every foot unit checked, doubled in guard mode, plus a fatigue delay) × 0.5. A tired or knocked man swings less often; a fresh one about every other second once the clip is done. The flanks cooldown with its 25 percent jitter plays the same role; nothing to adopt here now.

## 5. The clips

Lengths from the pack (20 fps, odd frame counts), impact frames from `descr_skeleton.txt`:

| clip | length | note |
|---|---|---|
| Swordsman stab, success | 1.40 to 1.80 s | impact frame 13 (0.65 s) or 18 (0.9 s) |
| Swordsman stab, fail | 1.00 to 1.30 s | the blow stopped; shorter return |
| Swordsman slash, success | 2.30 s | impact frame 25 (1.25 s) |
| Swordsman slash, fail | 1.50 s | |
| Mace (sword and shield) defend, stab high/mid/low | 0.80 to 0.95 s | guard up at frame 3 or 4 (0.15 to 0.20 s) |
| Mace defend, slash left-right / right-left | 0.95 to 1.05 s | guard up at frame 3 to 7 (0.15 to 0.35 s) |
| Spear defend, stab high / mid | 0.85 to 0.95 s | guard up at frame 6 (0.3 s) |
| Spear defend, slashes | 0.95 s | frame 6 or 7 |
| Spear stab, success / fail | 1.50 to 2.60 s / 0.50 to 1.05 s | |
| Knockback from front/back/left/right | 1.65 s | the stagger clips |
| Fatality pairs (victim and killer) | 6.2 to 9.0 s | synced; skipped for flanks |

Pattern: every attack has a success and a fail clip and the fail is a third shorter; every defend clip raises the guard within a third of a second and spends the rest returning; a defend costs the man about a second in which he does not swing.

The M2TW reference pack still lacks these clips as sheets (plan 020, "defend and parry, hit reactions"); `assets_dev/_m2tw_refs/render_clips.py` renders them the way the basic set was done.

## 6. What flanks has today

`sim/damage.rs`: every swing that reaches its target lands; the combat factor (attack minus skill, armour and shield, by sector, M2TW's sectors at 60 degrees) scales the damage, 1.13 per point, clamped at 12. Hit points are 160 / 90 / 100 / 80 by kind, base damage 23 (sword) and 32.5 (spear), so a kill takes about 4 to 9 landed blows at parity. A landed blow also rolls a stumble whose chance rides the same factor (0.65 at parity, 0.05 per point), the only place "the defence stopped it" exists. The audio already draws a blow's material (parry steel, shield wood, armour) in proportion to the points each defence put up. The swing byte has a free state value (3; 0, 1, 2 are ready, wind-up, recover) and the clip table has four free slots (8 of 12 used). The parallel kernel sees other men only through the tick snapshot (`Field` has no swing or target columns), so a victim can be told of a swing only through a chunk event applied serially, like `DamageEvent` today.

The proposal that follows from this is in `docs/plans/024-parry-and-block.md`.
