# 0165: Vanilla M2TW's compiled sound events decoded (events.dat)

Written by Claude Opus 5.5.

Follows devlog 0161 (M2TW evidence from the SSHIP text configs) and 0164 (charge phase rule). Research only: no game code, builds or game runs. Tools and dumps in work/research/m2tw-sound-events/.

Why: after the charge phase fix a 200k battle lost its mass atmosphere. Every M2TW sound fact so far came from the SSHIP mod's text configs. This entry reads what vanilla 1.52 itself ships, compiled into data/sounds/events.idx and events.dat, and what triggers the per-soldier sounds.

## Result in one paragraph

Vanilla has no group sound for units that wait or stand: the unit_taunt and unit_idle banks are present with 0 entries, and war_cry, druid_chant and screeching_women are not defined. What a waiting unit sounds like comes from its soldiers' animations. The high-morale variants of the ready stance (`ready_LF_high_morale`) fire Individual_Battle_Scream (-10 dB, mindist 2, probability 0.2, 1.0 within 10 m of the camera). Taunt animations fire Individual_Taunt (0 dB, mindist 2, probability 0.08, 1.0 within 10 m). Idle stands fire quiet fidgets (-40 dB). Every value checked is identical to SSHIP's except two global delays. So SSHIP's text was a faithful source, and the per-soldier voice layer on the animations is what vanilla uses for its atmosphere.

## Tools and output

- `evtpack.py`: decodes events.idx/events.dat to JSON (`events.json`) and to text in the descr_sounds layout (`events.txt`, 113k lines). Run `python3 evtpack.py events.json events.txt`.
- `skelsounds.py`: lists the sound markers of every animation in data/Animations/skeletons.dat (`anim_sound_markers.tsv`, 19,550 markers in 1,775 animation entries). This is where the triggers of the per-soldier sounds live.
- `hardcoded_event_names.json`: the 313 hard-coded event names in id order, read from the exe.

The layouts come from the pack writer in medieval2.exe (disassembled with objdump; function addresses are in the evtpack.py docstring and comments). The exe also holds the descr_sounds*.txt parser, and its sound setup (a6bd0e) parses descr_sounds.txt when present and can call the pack writer, one path gated by the `debug.snd_save_events` preference. Whether the vanilla data packs contain descr_sounds*.txt is not checked: the pack indexes are not plain text, and the official unpacker was not run because a game was running.

## The format

- events.idx: `EVT.PACK`, u32 0, u32 0x49 (version), u32 1026, then 1026 records {u32 4, u32 offset, u32 size, u32 type}. events.dat starts with the same header; the records are contiguous.
- Record types: 1 globals (1 record), 2 hard-coded events by id (313, UI and campaign), 3 named events (661: ANIM_*, narration, SM_* strat map), 4 banks by id (51).
- An event is groups of samples. Each sample carries the full parameter set: path, 3d, probability (u8, x255), volume (i16 dB), min/max pitch (u16, x32767.5), priority, fadein/fadeout/randomdelay/delay (u16 at 218.45 per second), looped, rndvolume, pref, ignore_pause, then mindist, maxdist, distancepriority, probradius, effect_level (3d) or pan and dry/wet levels (1d). Fixed-point values are read back to the shortest decimal the parser maps to them (2.0, not 1.996).
- A bank is a list of nodes. Each node has 5 event slots with a lod threshold, then class-specific conditions. The node class per bank comes from the exe's bank factory: UNIT_MOVE for unit_idle, march, run, charge, fighting, ambient, reform, collide, taunt, retreat, celebrate; UNIT_VOICE for unit_voice; AMBIENT_ANIMS for ambient_anims; and 27 other classes.

Decoded: every byte. All 1026 records parse to their exact idx sizes, globals included. Resolved from the exe: bank names, hard-coded event names, vocal types (Individual_* and the order voices), unit category (infantry 0, cavalry 1, siege 2, non_combatant 3, ship 4, handler 5), unit class (heavy 0, light 1, skirmish 2, spearmen 3, missile 4), ambient anim names (stretch ... taunt_hit), the unit_voice accent and class (read off the sample names), and the UNIT_MOVE terrain mask, season mask and climate list.

Not interpreted (read, kept raw in the dump): the condition fields of the siege, strat map, weather, water, building, cursor and music nodes; WEAPON_HIT's weapon and material fields (the material is clear from the sample names: metal, leather, shield, flesh, ground); the EAX environment block; the unit_voice flag `b70` (1 on order voices, 0 on individual ones); the event `kind` word and the slot flag, both 0 everywhere in 1.52. For 3d sounds the parser ORs the streamed/ducking keywords into mindist's two lowest mantissa bits, so those two flags cannot be read back for 3d sounds.

## 1. Soldiers and units not in melee

| Sound | Trigger (vanilla) | Vanilla | SSHIP | Differ |
|---|---|---|---|---|
| unit_taunt (group taunt sheet) | none | bank present, 0 entries | bank present, all entries commented out | no |
| unit_idle (group idle sheet) | none | bank present, 0 entries | bank present, no entries | no |
| war_cry, druid_chant, screeching_women banks | none | not defined | war_cry and druid_chant commented out, screeching_women absent | no |
| Individual_Taunt | taunt animations (34 in 16 skeleton families: infantry, missile, mounted; 1 to 7.5 s), one marker each | probability 0.08, 1.0 within 10 m (probradius 10), 0 dB, mindist 2, priority 100, pitch 0.9 to 1.0, rndvolume -20, distancepriority -2; 9 lines per accent and class | same values; SSHIP has 10 accents (adds Jerusalem and EE_Nordic) | no |
| Individual_Battle_Scream | `ready_LF_high_morale` animations (13, in the 2H sword, 2H axe, halberd, mace, pike, spear and knifeman skeletons, 1.75 to 6 s), marker in a frame window around 8 to 34 | probability 0.2, 1.0 within 10 m, -10 dB, mindist 2, priority 100, rndvolume -20; 44 lines per accent and class | same | no |
| taunt_hit (shield bash, ambient_anims) | no vanilla animation carries a taunt_hit marker | probability 0.5, -35 dB, mindist 2, pitch 0.7 to 1.2, priority 0, randomdelay 0.1, rndvolume -25; classes General, Heavy, Light; Taunt_01 to 11 | same values | no (and nothing plays it in vanilla) |
| Stand idle fidgets (ambient_anims) | stand idle animations, e.g. spear `stand_A_lf_idle_1`: armour clink and cloth at frames 31 and 56 | heavy and spearmen: armour clink p 0.25, cloth p 0.25; light, skirmish, missile: cloth p 0.45, no armour clink; cough, sniff, spit, throat p 0.1; all -40 dB, mindist 1, priority 50, rndvolume -15 | same | no |
| ANIM_Human_ready | ready idle animation, frame 0 | throat, sneeze, cough, yawn, spit, sniff, belch at p 0.004 each, whistle 0.01; -35 dB, mindist 1, priority 60 | not compared | |
| unit_ambient | engine: played only when the unit's state field (offset 0x2a0) is not 0, 1, 2 or 4, at probability 3.5 / unit size | marching jingle, grass, leaves, bush, rock, sand, water splash, armour clink by terrain; lod 1 / 10 / 30 at p 0.2 / 0.4 / 0.6; -20 to 0 dB; mindist 1, priority 60 | same in the nodes checked; the winter climate list differs (vanilla 4 climate ids, SSHIP 7 climates) | climate list only |

Notes:

- probradius is "distance to the camera inside which the probability is set to 1.0" (SSHIP header comment), so near the camera every taunt and every high-morale ready animation voices.
- How often the engine picks the taunt or high-morale ready animations for a waiting soldier is engine code, not in the data, and not decoded.
- The unit_ambient routine at a2f710 plays animal_idle for mounts in states 0 and 4 and nothing for the men; which unit states those numbers are is not decoded. unit_ambient's content is movement sound.

## 2. unit_fighting

| | Vanilla | SSHIP | Differ |
|---|---|---|---|
| Samples | Group_Fight_Small / Medium / Large | same | no |
| lod thresholds | 3 / 40 / 80 | 3 / 40 / 80 | no |
| volume | -40 dB | -40 | no |
| mindist | 10 | 10 | no |
| fadein / fadeout | 2 / 2 s | 2 / 2 | no |
| priority | 220, distancepriority 0 | 220, 0 | no |
| pitch | 0.9 to 1.1 | 0.9 to 1.1 | no |
| Units | infantry and cavalry of every class, siege missile/heavy/light (13 nodes) | same classes | no |

## 3. unit_charge and the charge yells

| | Vanilla | SSHIP | Differ |
|---|---|---|---|
| unit_charge samples | infantry_group_charge_small_01 / medium_01 / large_01, 02 (cavalry, handler and siege use the infantry sheets) | same | no |
| unit_charge lods | 5 / 10 / 30 | 5 / 10 / 30 | no |
| unit_charge volume, mindist | -25 dB, 10 | -25, 10 | no |
| unit_charge fadein / fadeout | 0 / 1 s | 0 / 1 | no |
| unit_charge delay, randomdelay | 2 s, 0.5 s | 2, 0.5 | no |
| unit_charge priority | 170, distancepriority -1 | 170, -1 | no |
| Individual_Charge samples | 47 Yell_Charge + 3 accent and class lines per node | same set per accent | no |
| Individual_Charge values | probability 0.2, 1.0 within 2 m, -30 dB, mindist 4, priority 80, rndvolume -20, distancepriority -2 | same | no |
| Individual_Charge trigger | the charge run cycle of each soldier (14 animations, 11 to 13 frames = 0.55 to 0.65 s at 20 fps), marker window frames 0 to 11 | not in the text | |

How many per unit: the data has no per-unit count. Each charging soldier rolls 0.2 whenever its charge cycle passes the marker. If markers fire on every loop, as the footstep markers of the walk and run cycles must, a 60-man unit starts about 60 x 0.2 / 0.65 = 18 yells a second before the priority 80 and the voice limit cut them. That per-loop firing is an inference, not a reading.

## 4. Individual combat voices (unit_voice, vanilla = SSHIP for all)

All 24 nodes (8 accents x General, Heavy, Light) of each type share one parameter set. Common: pitch 0.9 to 1.0, distancepriority -2, rndvolume -20, effect_level 0.5.

| Vocal | Trigger (animation markers) | Probability (probradius) | Volume | Mindist | Priority | Samples |
|---|---|---|---|---|---|---|
| Individual_Attack_Grunt | attack success animations (375 markers) | 0.4 (2 m) | -20 | 0.75 | 120 | M_Attack_Grunt + accent lines |
| Individual_Attack_Scream | charge_attack and charge_jump_attack, fatality attacker | 0.25 (10 m) | -15 | 0.75 | 120 | M_Attack_Yell, Attack_Yell_M/R + accent lines |
| Individual_Battle_Scream | ready_LF_high_morale | 0.2 (10 m) | -10 | 2 | 100 | accent Battle_Scream lines |
| Individual_Charge | charge run cycle | 0.2 (2 m) | -30 | 4 | 80 | Yell_Charge + accent lines |
| Individual_Death | die animations, fatality victim | 1.0 | 0 | 1.5 | 130 | Individual_death + accent lines; pitch 0.9 to 1.1, rndvolume 0 |
| Individual_Grunt | knockback, knockdown launch, fatality victim | 0.25 (10 m) | -20 | 0.75 | 120 | G_Grunt, M_Roman_Grunt, RV_Grunt, grunt |
| Individual_Groan | knockdown lying and recover | 0.25 (10 m) | -20 | 0.75 | 120 | Groan + accent lines |
| Individual_Fall_Grunt | fall landing | 1.0 | -20 | 1 | 120 | grunts |
| Individual_Fall_Scream | die_flailing_cycle | 1.0, looped | 0 | 3 | 160, distancepriority 0 | fall screams |
| Individual_Celebrate | celebrate animations | 0.08 | 0 | 2 | 100 | accent lines |
| Individual_Taunt | taunt animations | 0.08 (10 m) | 0 | 2 | 100 | accent lines |
| Individual_Retreat, Individual_Confirm | no animation marker (engine-triggered or unused) | 0.08, 0.1 | 0 | 2 | 100 | accent lines |

SSHIP's text gives the same probability, volume, mindist, priority and probradius for every row. There is no separate "battle scream" event beyond Individual_Battle_Scream.

## 5. ANIM_SWOOSH and ANIM_STAB

| | ANIM_SWOOSH vanilla | SSHIP | ANIM_STAB vanilla | SSHIP |
|---|---|---|---|---|
| Samples | swoosh_01 to 10 | same | Death_Hits_01 to 19 | same |
| Volume | -25 dB | -25 | 0 dB | 0 |
| Mindist | 0.75 | 0.75 | 0.75 | 0.75 |
| Priority | 80, distancepriority -1 | 80, -1 | 80, -1 | 80, -1 |
| Pitch | 0.9 to 1.1 | 0.9 to 1.1 | 1 to 1 | 1 to 1 |
| Fadeout | 2 s | 2 | 2 s | 2 |
| Trigger | 814 markers in attack animations | | 43 markers in 8 killing animations (fatalities, one axe hit, a rider killing a mount) | |

No difference. Also on the attack animations: ANIM_SCRAPE (heavy_clangs_01 to 13, -35 dB, mindist 0.75, priority 80, pitch 0.6 to 1.1) fires in successful attacks (471 markers). The weapon_hit bank's melee entries equal devlog 0161's table: death_hit on flesh 0 dB, mindist 1.3, priority 180; hit metal -20 dB, 0.75; shield and flesh 0 dB, 0.75 and 1.0; priority 90, distancepriority -2, probradius 7.

## 6. Globals and the voice limit

| Global | Vanilla | SSHIP | Differ |
|---|---|---|---|
| rolloff_factor / stratmap / doppler | 1 / 3 / 0.5 | 1 / 3 / 0.5 | no |
| volume_cutoff | 0.01 | 0.01 | no |
| priority_floor | -1000 | -1000 | no |
| pitch_offset | 0.2 | 0.2 | no |
| cam_cull_radius_unit / engine | 100 / 2000 m | 100 / 2000 | no |
| ducking on, fade in, fade out, amount | 1, 1, 1, -40 | same | no |
| unit_collide_threshold | 10 | 10 | no |
| unit_under_attack_delay | 120 s | 25 | yes |
| unit_warhorns_delay | 15 s | 9 | yes |
| unit_idle / unit_ambient probability scale | 3.5 / 3.5, both on (applied as 3.5 / unit size) | same | no |
| do_unit_anim_switch, unit_anim_switch | 0 (off), 2 m | same | no |
| unit_start_delay, unit_proximity_distance | 0, 0 | 0, commented out | no |
| music timeout, retrigger, fade in, fade out, fade out timeout | 60, 5, 0, 3, 10 | same | no |

Voice limit: none in the data. The exe asks the Miles 3D provider for "Maximum supported samples" and allocates 3D sample handles until Miles refuses (a193f0), so the cap is the provider's (medieval2.preference.cfg selects "Miles Fast 2D Positional Audio"). The number was not measured.

## Open

- How often the engine plays taunt and high-morale ready animations for waiting soldiers, and the unit state numbers in the unit_ambient routine: engine code, not decoded.
- Whether animation markers fire on every loop of a cycle (inferred from footsteps) and whether cam_cull_radius_unit applies to soldier markers.
- Whether vanilla's data packs hold descr_sounds*.txt: one run of tools/unpacker (about a minute of disk reads under Proton wine) with a sounds filter, when no game is running. If they do, diffing that text against events.txt also cross-checks the decoder.
- The marker flag (0 or 1) in skeletons.dat and the uninterpreted node fields listed above.
