# 0068 — Controls, selection and audio QoL pass (2026-07-30)

Branch `fix/controls-audio-qol`, one commit per item. Owner's bugfix
list ahead of 0.1.0. Build + clippy clean at every commit.

## UI click sounds are now messages (`audio.rs::UiCue`)

The old feedback keyed on state diffs: selection click on a CHANGED
`count_units`, order click on a changed `GroupData::order`. Reselecting
the same regiments or re-clicking the same attack target stayed silent
(the owner's "doesn't play second time unless I deselect" bug). Input
systems now write a `UiCue` message (`Select` / `Move` / `Attack` /
`Deploy`) at the action site and `event_cues` plays one clip per cue
kind per frame. Writers: lasso release, control group recall, class
hotkeys, card clicks, RMB click/drag orders, deployment placements.
`Deploy` clicks like a move but never horns. Auto orders (skirmish
withdrawal, at-ease engagement) write no cues, so the old `auto_order`
silence rule holds by construction. The war cry retarget edge still
needs per-regiment `prev_order`, so that vec stays.

## One war cry per charge onset

The rolling roar (3 clips re-fired on the 2.5 s vox gate) read as the
clip looping. Removed; the edge cry (attack target newly inside
CHARGE_RANGE, devlog 0024 rule) is the whole behavior now. Mined M2TW
charge window is 2.8..7 s (devlog 0056); the ~4 s clip covers it.

## Drag-line facing follows the drag hand (M2TW)

`line_layout` faced the enemy side of the line unconditionally. With
the camera rotated (deployment surveying, mid-battle) that could face
the drawn line INTO the camera. Now the caller passes a facing hint
from the camera: a left-to-right drag on screen faces away from the
viewer, right-to-left toward it, the M2TW gesture. The player picks
facing with the hand that draws. Script callers (`line_order`,
FL_TEST_FORM) pass a zero hint and keep the enemy-side fallback.

## Fire-at-will holds against melee-locked targets

Fire-at-will picked the nearest enemy block, engaged or not, and
poured volleys into fights (the owner's friendly fire complaint).
Target selection now skips `engaged` enemy regiments: M2TW holds fire
rather than shoot into a melee with friends in it. An explicit attack
order still forces the shot, and the flight keeps zero team checks,
so forced volleys still hit whoever the shafts cross (devlog 0060
model untouched). Expect archers to go quiet once lines lock; that is
the M2TW read and the reposition-or-force choice it creates.

## Selection QoL

- Ctrl+A / Ctrl+I / Ctrl+M class selects (all / infantry / missiles),
  the M2TW keys. Camera WASD pan ignores keys while Ctrl is down.
- Enter clears the selection (skipped during deployment, where Enter
  is Begin Battle).
- Shift+click on a unit card selects the run of cards from the last
  clicked card; Ctrl+click keeps the single toggle. Anchor is a
  Local, bounds-checked against stale battles.
- Settings > Controls > "Drag select": Lasso (default) or Box. Box
  mode reprojects the four marquee corners to the ground each frame,
  draws the quad outline ungated (player-facing UI), encloses via the
  existing polygon test, and a stationary click still point-picks.
  `select_along_line` grew an `enclose` flag; lasso self-closing is
  unchanged.

## Verification

cargo build --profile opt-dev + clippy: zero warnings at every step.
Owner play-test pending for: drag-hand facing feel, archer hold-fire
feel in symmetric fights, war cry timing, box select feel.
