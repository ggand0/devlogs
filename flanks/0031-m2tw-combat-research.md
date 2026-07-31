# 0031 — M2TW combat research record (2026-07-14)

Owner: formations don't feel like they matter; research M2TW's engine
properly and propose a design. Full proposal: docs/plans/
formation-combat-v2.md (gitignored, like this file). This entry is the
research record. Code landed alongside: only the standing-facing fix
(units dress to the ordered facing; FL_TEST_FORM facing err 0.00-0.01
rad).

## Sources

- totalwar.org "Combat M2TW" wiki + stat threads (kill formula,
  directionality).
- TWC threads: "Armour, defense skill, and shield and HP functions",
  "How does Charge bonus work" (braced reflection), "Medieval 2
  Mechanics" (mass, engine lineage).
- HeavenGames EDU modding guide (field semantics: soldier mass,
  formation line, stat_pri/_attr, stat_mental).
- VANILLA export_descr_unit.txt from the m2tw.narod.ru mirror —
  primary calibration data (516 KB, full unit roster).
- Local Steam install (owner pointer): Feral Definitive Edition
  repack. See "pack format" below — index parsed, payloads not.

## The M2TW model (what the community reverse-engineered)

- **Kill roll, not damage**: every landed strike rolls; combat factor
  = attack + modifiers − (defence skill + armour + shield). Factor 0 =
  1.9% kill chance, ±20% per point (~1.2^f), clamp ±20. Attack rate is
  per-weapon (min delay field ~25 ticks + a cycle factor: spears 0.6 =
  slower).
- **Directional defense — the load-bearing mechanic**: defence skill
  applies to front/side attacks, NOT rear. Shield applies front +
  LEFT side only (shield arm), nothing rear. Armour applies from all
  directions; `ap` weapons halve it. Per SOLDIER: getting behind a man
  strips skill+shield — flank/rear attacks kill several times faster
  before any morale effect. Formations in M2TW matter because facing
  is a per-soldier defensive resource.
- **Weapon attributes**: `spear` (anti-cav bonus, brace bonus, small
  anti-infantry penalty), `light_spear` (brace only), `spear_bonus_x`
  (+x atk vs cavalry, x ∈ 2..12), `long_pike` (reach, first strikes),
  `ap`. Charge bonus adds to attack while the charge state lasts.
- **Bracing/reflection**: braced spear-type units REFLECT the
  attacker's charge bonus back at the charger ("do not charge braced
  spears").
- **Mass**: per-soldier scalar (EDU `soldier` line, 4th field);
  cavalry mass from the mount. Drives melee shoving and charge
  knockback ("higher mass may blast people back on the charge").
- **Formation is unit data**: close order 1.2 x 1.2 m spacing, loose
  2.4 x 2.4; default rank depth per unit (militia 5, pikes 8, elite
  swords 3); special formations (schiltrom/phalanx/shield_wall/wedge)
  are per-unit abilities. `stat_mental` = morale + discipline (shock
  response) + training (formation neatness).

## Vanilla EDU calibration rows (from the mirror)

| unit | atk | chg | armour/skill/shield | mass | attrs |
|---|---|---|---|---|---|
| Town Militia | 5 | 2 | 0/1/6 | 0.8 | light_spear, spear_bonus_4 |
| Spear Militia | 5 | 2 | 0/1/6 | 1.0 | spear, spear_bonus_8 |
| Armored Sergeants | 8 | 3 | 5/4/6 | 1.2 | spear, spear_bonus_8 |
| Pike Militia | 7 | 2 | 0/1/0 | 1.0 | spear, long_pike, sb_8 |
| Dism. Feudal Knights | 12 | 5 | 6/7/7 | 1.5 | — |
| Dism. English Knights | 21 | 6 | 8/5/0 | 1.2 | ap (axe) |
| Feudal Knights (cav) | 11 | 14 | 8/2/3 | mount | lance |

Reading: defence is SPLIT and directional (militia are all shield;
knights are armour+skill and keep most of it from any front-arc
attack), attack spread is wide (5..21), infantry charge is minor
(2-6) vs lance 14, and depth/spacing/mass are unit identity.

## Feral DE pack format (the unpack attempt)

The Steam install has no plain-text data; everything lives in
packs/data_N.pack. Findings (packprobe/m2unpack.py in the session
scratchpad, worth preserving if we ever want assets):

- Header: `PACK`, u32 version 0x30000, u32 entry count, u32 record-
  region size, u32, u32.
- Records: `[u32 data_offset][u32 index][u32 uncompressed_size]
  [u32 compressed_size]` then name (null-terminated, padded to 4
  relative to region start). Offsets absolute; adjacent entries'
  offset+csize = next offset (verified).
- Entries with csize == usize are stored raw (e.g. .cas animations);
  text/xml entries are compressed with a CUSTOM LZSS variant — NOT
  zlib/gzip, NOT raw LZ4, not snappy (all tested). Streams are byte-
  aligned with 2-3 byte match tokens interleaved with long literal
  runs; `export_descr_unit.txt` is 513,960 -> 73,171 bytes. Started
  known-plaintext reversing (the match encoding didn't yield in
  reasonable time) and STOPPED: the same files are mirrored as plain
  text online, so reversing buys nothing for stats. The official
  unpacker.exe is Windows-only (no wine on this box).
- IMPORTANT SCOPE NOTE: even a full unpack only exposes DATA — EDU
  stat tables, formation configs, animation skeletons, models,
  textures, sounds, battle_config.xml. The combat CODE (kill-roll
  implementation, collision solver, pathing) is compiled into
  medieval2.exe and is not in any pack. The community formulas above
  come from black-box testing and disassembly, not from data files.

## Engine comparison: are we already M2TW-shaped?

Foundations we ALREADY share (no rework needed):
- Barrel bodies: 2D circle collision, mass-weighted separation +
  positional correction == their collision cylinders and mass shoving.
- Regiments of individual soldiers in rigid slot formations, orders as
  points, per-regiment morale/rout, engage/charge states.
- Swing-timer melee with reach — comparable cadence to their timed
  attacks (our cooldown_ticks ~= their min-delay + cycle factor).
- Per-unit facing (yaw) already simulated and now formation-dressed —
  the prerequisite for directional defense is in place.

What M2TW has that we lack — all MODEL-level, all additive:
- Stat structure (attack/skill/armour/shield/ap) instead of flat
  damage: swap the kind table + a factor computation at damage apply.
- Directional defense: a sector test per damage EVENT (dot/cross vs
  victim yaw) in the serial apply pass — few hundred events/tick.
- Charge as physical impulse + stagger: reuse the positional-
  correction channel + swing-state reset; no new solver.
- Rank discipline (only front ranks fight; the anti-mush): per-unit
  slot-depth dot product against per-group vecs, hot-loop cheap, but
  changes pacing — gated + rebalanced (plan phase C).

One M2TW thing we deliberately DON'T copy: melee duel-pairing with
synchronized attack/defense animations (their soldiers lock into 1v1
exchanges; it's why their battles cap ~10k). Our free-for-all
reach-based swings are the thing that scales to 200k; the stat model
bolts onto it without the pairing.

Verdict: no fundamental rework. The engine is already the "barrels
with mass" foundation; what's missing is the stat/directional model on
top of the damage pass, physics impulses through existing channels,
and the rank-fighting behavior rule. See the plan for phases and
acceptance gates.
