# The archer's shot clips in play

Written by Claude Opus 5.5. 2026-10-12, branch `feat/spearman-archer-clips` in the `flanks-gfx` tree, commit c29ab30 on 7fd69b9. Astra's handoff: `work/handoffs/111-archer-shot-clips-for-claude-2026-10-12.md`. Model and clips from `assets_dev/archer/release_aim_v2/`, byte for byte (the nine hashes of the handoff match).

## What changed

- `assets/units/archer.glb` is the 22-joint skinned model (LFS). Its parts and pivots are the old model's, so `archer.shoot.json` still poses the far levels. The eight clips sit beside it; the player loads stand, ready, draw, release, aim_up and release_up. Aim and hold_up are the last samples of draw and aim_up and ship as the contract's endpoints.
- The loader takes a skin and a bow rig together. The clip buffer holds the shot tables, then the skeleton's clips. A skinned archer's pose holds the bow rig's 45 part slots (his shadow and far levels draw by them), then his 22 joints.
- Four clip slots (draw, release, aim_up, release_up; 14 in all). The arrow is a joint role; a joint's pose slot carries a scale, and the arrow's is 0 while it is off the string, so its corners gather at the hand.
- An archer's blows and cheer swing his skeleton's arms the way his bow rig swings them.
- The loader logs a kind's missing clips in one line instead of one warning per clip, since every melee kind now lacks the four shot slots.

## The shot cycle

The sim is unchanged: 1.5 s of draw (`DRAW_TICKS` 45), the loose, then a reload of 6.4 to 8.9 s. The renderer plays after the loose a 0.6 s release, a 5.8 s reload and up to 2.5 s of hold, as before.

Astra's aim_up is the old far tables' raise: from the drawn level aim, 25° of chest lean plus 10° of bow lift. So the near archer plays the far cycle with the new clips:

1. **First shot of a volley**, from the bow down: the sim's draw plays the draw clip (ready to the level aim, the arrow at 0.40 s), the loose plays the level release.
2. **Every later shot**, from the held aim: the sim's draw plays aim_up (exactly 1.5 s), the loose plays release_up.
3. **The reload**: the release's end comes down to the ready pose over 0.5 s, he waits in ready, and the draw clip plays over the reload's last 1.5 s, so he reaches the held aim as the reload ends. He holds it until the sim's next draw.
4. When the hold runs out (the volley is over) or he walks, the bow goes down: the clips blend out over 0.5 s into his stance.

The anim z encoding gains three clips: 0 to 2 are the draw, release and reload of a shot from the bow down, 3 to 5 of one from the held aim, and the held aim is clip 3 at 0. A bow rig plays 3 to 5 as 0 to 2, so the far levels are unchanged. The renderer state gains two flags per soldier (`Smooth::up`, `Smooth::drawn`; GPU bits 512 and 1024 of `Smooth::swing`): whether this draw started from the held aim, and whether he holds it.

## Stopgaps

- **The lowering after a release is a blend**, release end to ready over 0.5 s (`SHOT_LOWER_S`). The reload clip (bow down, arrow from the quiver) does not exist yet; the arrow shows when the draw nocks it.
- **Every later shot goes up 35°**, as the far tables do, whatever arc the arrow flies (flat 0 to 15°, lofted 50 to 65°). Choosing by the arc needs the sim's arc before the loose.
- A draw cut short (the target gone) runs back over 0.25 s, as every wind-up does.

## Checks

- opt-dev build, strict clippy and the 36 tests pass. The loader reports the archer's skeleton of 22 joints with the six shot clips loaded.
- In play, `FL_TEST_ARCHERY=1`, 120 men a regiment, cameras pinned by `FL_CAM_*`, recorded with `work/scripts/clip.sh` (clips in `tmp/runs/archer_clip/`):
  - The first volley draws from ready and looses level; the second raises the held aim to 35° and looses from it; the third draws from ready again after the lowering. A temporary tint by shot clip (removed before the commit) confirmed each phase: the release_up plays its full 0.6 s, then the 0.5 s lowering, then ready.
  - The arrow shows on the string from the draw's nock through the held aim (its point past the bow hand) and is gone from the release on.
  - L0 and L1 archers hold the same clip pose. At the L1/L2 switch the L1 archers (clips) and L2 archers (tables) show the same 35° aim at the same moment.
  - With `FL_ARCHERY_ATTACK=1` the archers run off with the bow down in the gait.
  - `FL_GPU_SYNC=0` (the CPU path, parts only) shoots from the tables as before.
- A first build had a WGSL reserved word (`from`) in unit_build.wgsl, which stopped the build pass and hid every soldier; fixed before the commit.
- `soldier.rs` and the sim are untouched, so no hash run is needed.
- Not checked: an archer's melee blow and cheer on the skeleton (the attacker run did not frame the fight), and the pose pass cost of the skinned archer.
- Not done by me: watching the motion by feel at full speed.

## Open

- The reload clip (bow down, arrow from the quiver) replaces the lowering blend; Astra moves the quiver first (handoff 111).
- Level or 35° by the arrow's arc, once the sim decides the arc before the loose.
