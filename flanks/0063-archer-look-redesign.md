# 0063: archer look — hooded longbowman redesign

Date: 2026-07-27. Continued from 0062 (art round 3+). Owner verdicts along the way: sallet read as "not an archer", stacked-box cervelliere read as "a poop", octagon-cross cone was better but the bare cube face under it read weird; "completely redesign, historically accurate".

## What landed

- **Head: wool hood that WRAPS the cube** — crown slab overhanging the brow, back panel to the neck, cheek flaps framing a face opening, chin wrap, shoulder cape, liripipe tail. This is the historically accurate headwear for levy/yeoman longbowmen (hoods and coifs, not helms) AND it solves the engine's core problem: a bare cube face always looks like a cube with a hat perched on it, so the head has to be *clothed*, not *hatted*. Color: muted Lincoln green — no other kind wears cloth on the head, so the kind reads at battle zoom even when the bow is edge-on.
- **Body: one wool tunic** in two-tone team livery (bright TEAM chest, TEAM_MUTED skirt) cinched by a belt, team sleeves, diagonal quiver strap, dark hose, knee boots. The leather bracer is the only armor — an archer's piece. The previous tabard + pale linen skirt + gambeson sleeves were leftovers of the steel-helm "professional" concept and mismatched the hood.
- **Shoulders belong to the hood's cape**, not to team rolls — cape and hood read as one garment.
- Kept from the earlier rounds (owner-tested): the longbow with the stave/string gap opened SIDEWAYS (~10 cm in X, so the two lines stay separated from front and behind — a gap in depth projects onto one line, which is what made the bow "a stick"), the slim back quiver with the 3-arrow fletch fan overtopping the head, the bright thin string.

## Failed iterations this session (for the record)

- Cervelliere from stacked shrinking boxes, even symmetric with 45°-crossed tiers (cuboid_y helper): reads as a swirl/ziggurat. The engine's winning headgear formula is ONE strong slab (kettle brim, knight's flared crown), never a tapering stack.
- frustum_y primitive (tapered N-gon cone, flat-shaded): the cone itself was fine, but it sat on the bare cube face — kept with #[allow(dead_code)] for future helmeted kinds.

## Round 3: the Sherwood body

Owner: the recolor rounds were "literally nothing has changed" — the tunic stack still read as the man-at-arms layout. Given `m2tw_sherwood_archers.png` as the target: plain ALL-GREEN forester (tunic green like the hood, with a whisper of team dye so the armies' archers differ), long sleeves, leather belt, brown hose, low shoes, back quiver with the fan, bracer. All ornamentation from the "cool archer" attempt (padded jack, quilt seams, sash, hem band, split hem, buckle, hip quiver, buckler, puttees, Sherwood hood peak) went in and came back OUT — the M2TW look is deliberately plain. Hood + cape from round 2 kept as approved.

## Debug tooling added

- camera.rs: FL_CAM_YAW at spawn, and FL_CAM_LOCK=1 to pin focus/yaw/pitch/dist to the FL_CAM_* values every frame. Without the lock, a cursor parked at a screen edge edge-pans the camera away before any screenshot.
- Screenshot loop notes: pkill must be `pkill -x flanks` (a -f pattern matches the invoking shell itself); background the game with setsid+nohup+disown or the tool timeout kills it; synthetic menu clicks didn't land, but FL_TEST_* scenarios auto-start anyway.
- The Archery scenario's "static" orange holders CHARGE at t≈12-16s unless FL_AI=0 (AI orders them; hold doesn't stop the AI).

Build + clippy clean. Not committed — owner feel pass pending.
