# 0022 — Corpses, banners, charge momentum (2026-07-10)

`feat/battle-feel` continued (`ccd384e`), while the owner generates the
ElevenLabs audio batch (asset list + prompts + durations in
`work/audio/audio-plan.md`).

## Persistent corpses

The dead used to vanish after their 0.6 s topple — two-hour battles left
a spotless lawn. Now `process_deaths` freezes each body's final pose
(`fx = 2.0`, the shader's fully-toppled state) into a `Corpses` resource:
per-kind static instance lists, ring-buffered at 25k/kind. Two extra
instance-bucket entities (same meshes, `CorpseBucket` marker) draw them;
a sync system copies only when dirty. Escaped routers leave nothing;
R-restart clears the field. Cost: 2 draw calls + up to ~2.4 MB/frame in
the existing extract copy — nothing measurable. The front now leaves a
carpet of bodies that maps where the line stood and where routs got run
down. Biggest immersion-per-line-of-code item so far.

## Regiment banners

TW-style flag per regiment (~200 plain bevy entities + a handful of
shared unlit materials/meshes — the no-entity rule is about UNITS):

- Team color; KIND is the flag shape (square standard = knights, long
  pennant = men-at-arms).
- Red MORALE bar + yellow STRENGTH bar, left-anchored fills (scale +
  translate trick, zero material churn).
- Billboards to camera yaw, scales with camera distance
  (`dist * 0.013`, clamp 0.8–3.2 — first pass at 0.018/5.0 dominated
  wide views).
- Wavering (<25 morale): the banner trembles (scale pulse). Broken:
  gray flag material swap. Selected: white diamond marker. All hidden
  by the G debug-viz toggle and for dead regiments.

Morale stops being an invisible stat: you can watch a line segment's
bars drain and pre-empt the break — that's new gameplay, not just UI.

## Charge momentum

Swings started above 60% of max speed set a charge bit (packed into the
`swing` column beside the state) worth 1.75× damage and a 1.35× lunge.
Emergent effects: the front rank of a marching regiment hits like a
hammer on first contact; standing grinds stay attrition; the jam yield
means you can't "charge" inside a press. Pairs with the war-horn audio
when it lands.

## Verification

Screenshots: banner field over a 200-regiment battle (bars draining
along the front, gray broken flags), ground-level shot of the melee with
corpse carpets between the standing fighters. Casualty rate at first
contact up vs pre-charge runs, as intended. Clippy clean; sim timings
unchanged (banners are Update-side transform writes on 200 entities).

## Audio wired (`da8f988`)

Owner generated 34 ElevenLabs assets (mp3, in `assets/`). `audio.rs`
mixes them from aggregate sim signals only:

- 4 looping beds crossfaded per frame (far din / mid din / close melee /
  marching drums) from engaged-regiment counts, camera proximity, and
  hits/tick; ~0.35 s smoothing; hearing radius grows with zoom-out.
- Clang/shield one-shots budgeted from hits/tick × proximity² through a
  fractional accumulator (2/frame cap); death screams on kill deltas,
  rate-limited. Volume + pitch jitter on everything.
- Event cues on state transitions: charge horn (new own orders), rout
  wail + alarm horn (breaks), rally cheer, selection click, outcome
  stings. War-cry trigger wired but waiting on a usable crowd-roar asset.
- FL_VOLUME master; missing assets degrade to silence; bevy `mp3`
  feature; AssetPlugin anchored to the repo root so the direct binary
  finds assets (bevy defaults to the exe dir).

Failed generations documented with retry prompts in `work/audio/audio-plan.md`
(bed_march, bed_wind_field, vox_warcry, ui_order + a play-twice trick
for the two-blast rout horn). Rule that emerged: ElevenLabs latches onto
the first concrete noun — put crowd/scale words first, keep them plural,
avoid "gentle" (near-silence) and singular phrases like "a battle cry".

## Next on this branch

March-in-step walk phase, controls bundle (halt key, control groups,
hover highlight, edge pan), dust if wanted; regenerate the four failed
audio assets.
