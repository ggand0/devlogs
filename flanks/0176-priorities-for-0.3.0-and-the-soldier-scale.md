# 0176: priorities for 0.3.0, Gota's item lists, and the soldier scale

Written by Claude Fable 5.1.

2026-10-03, Linux, main 25ec337 (0.2.1 plus the README commit). A planning session after the 0.2.1 release. No game code changed. This devlog keeps Gota's three prompts word for word, the scope Gota set for 0.3.0, what the exploration found, and the first look at the soldier scale.

## Gota's prompts, as typed

### Prompt 1

```text
Read 
- 018-roadmap-to-early-access.md
- 019-release-0.2.1.md
and other relevant plan docs and devlogs if needed

I just released 0.2.1 and want to plan high priority items for 0.3.0.
A thing I really wanna fix soon is the animation bug (detached arm on shield wall/spear wall), and how soldiers end up doing slo penguin walk when units are blobbed up and soldiers get blocked by others (can be fixable by introducing the m2tw shuffling animations I think, I have one for the footwork branch created by Astra for side shuffle animation but never incorporated). and i wanna add a few extra variations for attacking or something. 

Another thing is soldier spacing. right now the gap between soldiers is wide like m2tw's loose formation since when it was tightened in pre-0.1.0 weapons clip through the soldier in front of another. Need a fix for this

Then, probably a bigger map but idk the size of m2tw maps. 2048x2048 first sounds good? Then optimization that comes with it. Then I wanna create more interesting like ones that contain hill to defend, or one looking like battle of Agincourt maybe. Then pathfinding?

It's already 5-6 items or more, but I'd like to learn your thoughts on these plus what would you do other than / instead of these items. Take your time to explore and let me know.
```

### Prompt 2

```text
Right, but I have lots of things I wanna do, need to prioritize. For example these are things I forgot to mention in the initial prompt
  - audio: death sfx regen never done, though overall current sound atomosphere is pretty good, m2tw charge voice sfx like "chaaaarge!", "attack!" or something. theres probably more instances like this
  - audio: optional random english screaming / jeering like "hold the line!" depending on the situation (play one suitable for each situation) for more immersiveness. i like the current "retreat!" like voice clips I hear sometimes, since they make the battlefield more authentic
  - battle phase based music like m2tw: deployment, marching, intense battle, etc. I generated some on eleven labs music and they're really good. music volume slider in settings
  - soldiers shouting confirmation voice "yes!" "<unit wide crowd sfx sounding like hai!!." (hard to explain), this is pretty important for immersiveness and make the marching units more interesting
  - speaking on marching, I think soldiers walk / run so fast and not realistic. it's fine as of 0.2.1 obviously tho. sometimes some soldiers run reaaaaly fast for example when MAA is attacking but the path is blocked by another unit. soldiers who made it through run reallly fast, like i feel maybe there's a logic that makes soldiers who are behind the unit center go faster or something
  - commander and optionally flag bearer per unit. Make commander really smart so if you zoom in the camera you could see interesting movements and fun to watch. more health than regular soldier
  - PARRY mechanic: almost forgot: when theres only 10k-20k soldiers they die off pretty quickly since TTK is pretty fast without parrying / blocking mechanism. I feel this is kind of a must for 0.3.0. adding relevant animation. Do we need some kind of matching algo for this??
  - animation: when charging: soldiers should raise the sword like they would do in IRL not just running in the last minute of charging. Spearman should hold the spear vertically when standing / marching. (theres probably more I forgot)
  - when to add cavalry / general units? obviously they'd take time to implement and test, so no rush
  - map: currently outside the boundary of map is mid-air, empty and while I think this is totally fine for a while (maybe 0.4.0), we should render far landscape like other games and forbit the camera going beyond the actual map area to make the view immersive
  - a nice medieval town map after pathfinding.
  - a small fort defense map: later
  - balancing spearwall / shieldwall combat, and also consider preventing them from running during these formations
  - slope combat / fatigue like you mentioned
  - arrow hit / melee hit / better death fall animations. in the future: killing blow animations
  - for animation, I had Claude mine m2tw animation and generate nice reference gif / image / visualization for Astra (side shuffle animation), I gave it to Astra and it generate pretty nice animation one-shot. So I think for m2tw ish animation generating this for all relevant animation is definitely priority. Forgot to mention. But I forgot where did the reference data go. Canyou find it?
  - forgot to add skybox but the game looks great without it, low priority maybe. I didn't even notice until now
  - some optional nice-to-have graphics stuff like clouds, conditional fogs, rain, night, ambient occlusion and other common tricks
  - In the future can I add DLSS to generate frames to improve performance? I remember I used "CNN" option for the finals game and it made the game pretty fast
  - SHINY armor and swords, they looked great in m2tw and really awesome
  - when to add crossbow men, artillary, gun powder units (future ofc, but I remember I really enjoyed penetrating ranks with balista, canons and it was a very fun minigame. this was when I was sallying out as the siege defender, since the enemy is defensive and static, I moved arty units all the way from castle gate to the far flanks of enemy line since Ai was stupid, and then fire projectiles to enjoy bawling and this was also very effective at killing so many soldiers. Something to note and remember. Balista is worth adding earlier after cav and generals I suppose
  - whether to let soldiers in teh rear some occasional taunt animation or something for immersiveness rather than just slowly walking toward the frontline (optional)
  - optionally, as a minigame, add a gamemode where you kill lots of soldiers in some satisfying way to have fun (arty, muso TPS shit, etc.): just something to note, obviously im not gonna add it (potential pivoting idea in the unlikely scenario etc.)
  - future: expose stuff so we can edit / adjust / mod things of the game as much as possible (yaml?)
  - better AI: right, so I also felt at some point that, while the current chad AI just attacking the nearest enemy units, we could create a smarter AI conditioned by some traits like being more defensive, flexible, using different type of formation variants. This can be done before adding generals
  - UNIT GROUPING: forgot, but when I actually want to control units, it's inconvenient without this feature. Something like m2tw, when grouped you keep the formation as is, and group attack order, units inside the group attack the unit accordingly based on the ordered unit; it probably attacks the nearest enemy close to the attack ordered enemy or something. should research
  - UI: current red arrow indicating attack order is hard to see since enemy flag is orange. something better needed
  - feature to toggle the army position in the map. Blue is playable but always deployed to one side of the map and players can't play the other half

There's probably more but enumerated stuff that came out from my brain right now
```

### Prompt 3 (the scope for 0.3.0)

```text
No, I'll do the 2048px map in 0.3.0, the current map is too small for 200k or more units and I find it a problem.also this means i can't run good benchmarks if you can't fit 300k-500k soldiers in the map in a battle situation
- animation
- bigger map
- pathfinding
- + some other stuff as the game progresses like audio

all in 0.3.0


I just run different agents in parallel
You make it work first and then polish later
I want to ship things somewhat fast, sicne as a player I really wanna play 0.5.0 0.6.0 myself
and siege battle in the future is also a must. imagine how you can reproduce the battle of helm's deep in this engine?

can you record the last two prompts in a devlog for the record as is? I'll likely copy paste them at some point.

> The scale decision,
let's do this now. explain intuitively and clarify this point
```

## What Gota decided

- 0.3.0 holds animation, the bigger map (2048 by 2048 m), pathfinding, and other work such as audio as it comes up. Claude had proposed moving the map to a later release; Gota turned that down. The reasons: today's field is too small for 200k, and 300k to 500k cannot be put into a battle on it, so the larger benchmarks cannot be run.
- Several agents work in parallel. Make a thing work first, polish it later. Ship fast.
- Siege battles are a must in the future (Helm's Deep as the picture).

## What the exploration found

All from reading the code and the records. Nothing below was changed.

- The M2TW reference renders are in `refs/animation/m2tw_shuffle_refs/` (16 sheets and GIFs: mace, spear and bowman shuffle left and right, mace side step). Handoff 037 and note 011 still name `resources/animation/`, the folder's old name. The readers are `work/scripts/m2tw_anim/m2anim.py` and `render_shuffle.py`. Astra's knight side shuffle from those references is in `assets_dev/knight/shuffle_v1` and was never wired in.
- Detached arm in the shieldwall and spearwall (not reproduced in play in this session): the wall turns the whole shield part 60 degrees about the body's centre axis and lifts it (`put_simple` and the part 5 branch of `place` in `src/shaders/unit_pose.wgsl`). That was written for the code-built shield slab. On the GLB models the part is the whole left arm with the shield, so the shoulder end leaves the socket.
- The slow penguin walk has two causes. The gait never drops below two steps a second (`gait.rs`: 1.0 + 0.16 x speed cycles a second, two steps a cycle), so a man creeping at 0.3 m/s takes 14 cm steps. And the sim lets a blocked man creep at 0.1 to 0.6 m/s, where M2TW has no locomotion under 0.75 m/s (devlog 0120). The legs also only pitch, so a sideways step plays a forward walk.
- Weapon clipping in tight ranks: readiness is one value per regiment, so every rank levels its weapon. The sim already knows per man whether a comrade stands close ahead (`SIGHT_AHEAD` in `sim/soldier.rs`).
- Men running very fast after being held up: `drive` in `sim/soldier.rs` sends every man to his own slot at his kind's full speed (9.5 m/s for men-at-arms) and slows him only inside the last 35 m in proportion to the distance left (`ARRIVE_RADIUS`). A man who was held up is far from his slot and runs at full speed while the rest of his unit, near its slots, moves slowly.
- Every blow lands today; the defence stats only scale the damage (`sim/damage.rs`). There is no block or parry.
- M2TW's battlefield size, measured from the `descr_battle.txt` files of eight battles in Gota's install (agincourt, hastings, pavia, tannenberg, arsuf, otumba and two multiplayer maps): deployment and unit coordinates reach 845 to 860 m from the centre on both axes, so the playable area is about 1.7 by 1.7 km. M2TW ships Agincourt with its deployment areas and unit positions in plain text.
- Bigger-map costs already on record: the collision grid caps each axis near 3,072 m, the grid rebuild and the density field scale with map area (plan 013 stage 1 fixes the rebuild), the terrain is 393,216 triangles at 1024 by 768 m and would be 2.1 million at 2048 by 2048 with 2 m cells, and the map size is compile-time constants in `terrain.rs`. The game has pause and no fast-forward.
- Attack clips authored and not wired: the knight diagonal slash v4 (note 010) and a man-at-arms sword rig in `assets_dev/man_at_arms/textured_v3_sword_rig`.

## The soldier scale, seen in the game

The scale mismatch was recorded in devlog 0086 and parked ("animation now, scale later"): the world is in true metres (1.4 m between slots, 1.8 m of sword reach, 120 m of arrow range, a 1.7 m shrub) and the soldiers are 1.0 to 1.1 m tall. `FL_UNIT_SCALE=1.64` draws a knight at 1.8 m.

Two runs of `target/opt-dev/flanks` (built 2026-10-02 23:24), `FL_TEST_FRONT=1`, 200k, a 1600x900 window, the flank view (`FL_CAM_X=-470 FL_CAM_Z=0 FL_CAM_DIST=15 FL_CAM_PITCH=0.35 FL_CAM_YAW=-1.5708`, locked), shots through `work/scripts/shots.sh` at 2, 10, 25 and 40 s. Raw shots and logs: `tmp/runs/scale/`. The comparison with a to-scale drawing of one rank: `work/notes/vis/026-soldier-scale.png`.

- Today the gap between two neighbours is about three bodies wide (1.06 m against a 0.34 m body). At 1.64 it is about one and a half (0.84 m against 0.56 m), and the standing lines read as ranks.
- At 1.64 the levelled swords reach into the man ahead. This is the clipping that stopped the tighter spacing before 0.1.0; it needs the guard per soldier.
- The fps overlay in the shots read 236 and 237 at 10 s (both scales the same) and 224 against 162 at 40 s with the lines standing in contact. These are overlay readings in a small window while a screenshot was being taken, not a measurement by the perf rules. They say only that the same camera spot costs more with bigger men, as expected: the levels switch by on-screen pixels.
- The soldier counts at 40 s were 192,803 and 192,781, so the fight itself ran about the same.

## Gota's try-out at 1.64, and where the fps went

Gota played 200k at `FL_UNIT_SCALE=1.64` in the maximized window and saved four screenshots in `refs/unit_scale_1.64/`. Verdict: it looks more realistic, shoulder to shoulder, and it fixes the gap and the running speed. Two problems: the weapons clip through other soldiers (`gap_clip_situation.png`: every rank holds its sword level into the back of the man ahead), and the frame rate dropped a lot: 84 fps in the mid-air spot (`mid-air_spot_200k.png`), 106 in the ground spot. The fallen also float a little (`bodies_float.png`), as the shader's fixed 0.5 for the feet predicted. Gota also said some aspects of the original scaling are missed; which ones is still open.

The drop is the detail levels, not the size itself. `LodBands::new` in `render_units.rs` puts each switch where a soldier is a set number of pixels tall (`LOD_PX_DEFAULT` 28, 12, 3), so a man 1.64 times taller keeps each detailed model 1.64 times further out. Multiplying the thresholds by 1.64 puts every switch back at today's distance, and `shadow_level` follows the same thresholds.

Four runs of `target/opt-dev/flanks` (main 25ec337), `FL_TEST_FRONT=1`, 200k, `FL_WINDOW=2494x1363`, the mid-air view (`FL_CAM_X=-400 FL_CAM_Z=0 FL_CAM_DIST=140 FL_CAM_PITCH=0.28 FL_CAM_YAW=-1.5708`, locked), 50 s each, the median of the log's last eight fps lines (34 to 48 s, lines in contact). Runner and logs: `tmp/runs/scale/midair/`.

| Run | fps | Men at L0 / L1 / L2 / L3 at the end |
|---|---|---|
| scale 1.0 | 164 | 11 / 13,280 / 87,502 / 89,809 |
| scale 1.64 | 99 | 5,168 / 25,158 / 146,809 / 13,488 |
| scale 1.64, `FL_LOD_PX=45.9,19.7,4.9` | 154.5 | 14 / 13,284 / 88,010 / 89,315 |
| scale 1.0 again | 163 | 11 / 13,279 / 87,501 / 89,810 |

Not a measurement by the perf rules: the desktop was not checked for other drawing and the window's focus was not controlled. The two baseline runs agree within 1 fps and the level counts explain the gap, so the direction is safe: the scaled thresholds return all but about 6% of the frame rate, and that last part is the bigger men covering more pixels. The price is that each simpler model is shown larger than before (L1 up to 46 px tall instead of 28). Crops of the same second with and without the scaled thresholds, `tmp/runs/scale/midair/cmp_near.png` and `cmp_mid.png`, are hard to tell apart at this distance; the far men read a little flatter.

## Gota's check with the scaled thresholds

With `FL_UNIT_SCALE=1.64 FL_LOD_PX=46,20,5` Gota read about 150 fps or a little less on the ground and 130 to 135 in the mid-air spot at 200k (`refs/unit_scale_1.64/with_FL_LOD_PX=46-20-5.png`: 131 fps, 188,883 soldiers). The look holds up. Gota would make the scale the default on a feature branch.

What Gota asked for with it:

- The gap between soldiers as its own parameter, apart from the physics that pushes overlapping men apart. The roomy look of the old scale was liked too, where each man has room to move without clipping; not every unit has to stand as tight as a phalanx, and skirmishers and archers would stand wider. Gota may tune the gaps later.
- A note that the camera's lowest height may need adjusting after more play. Today the zoom is clamped to 15 to 900 m and the pitch to 0.25 to 1.45 rad (`camera.rs`).

Facts for those two:

- The code already keeps the two apart. The slot pitch is `FormSpacing::pitch()` in `formation.rs` (wall 1.05 by 1.15 m, normal 1.4, loose 2.66), one set for every kind. The physics is `SEP_RADIUS` 1.4 m (a soft push) and `HARD_RADIUS` 0.9 m (true overlap, corrected in position) in `sim/soldier.rs`. A slot pitch under the soft push distance cannot be held, which is why the wall has its own rest distance.
- The old roomy look at the new scale is a pitch of about 2.3 m (1.4 times 1.64). Loose order (the L key) is 2.66 m. At 2.3 m, 200k men need about 1.06 km2 of slots and today's field is 0.79 km2, so wide gaps at 200k need the 2048 map.
- All four models are built 1.80 m tall to the top of the helmet. The game draws the knight at 1.002 of that and the other three at 0.911 (1.64 m), from the type table's half heights of 0.55 and 0.50.
- The comrade-ahead sight bit (`SIGHT_AHEAD`) is worked out only for men on their way to a fight, every 8 ticks. A standing rear rank does not have it, so a guard chosen per soldier needs the test added to the scan every man runs each tick.

Open: the heights per kind (all four at the built 1.80 m, or the knight taller), the branch for the scale, the per-kind gap, the guard per soldier (its own proposal), the camera's lowest height.
