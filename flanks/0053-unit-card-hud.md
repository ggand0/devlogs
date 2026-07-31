# 0053: Unit card bar + minimal control panel

TW-style battle HUD, the first step of the "playable by actual people"
roadmap after the game shell (0046). All new code in `src/unit_cards.rs`
(`UnitCardsPlugin`); pure presentation over existing state.

## Unit card bar (bottom center)

One card per player regiment, spawned `OnEnter(Battle)` after
`setup_battle`, cleaned by `DespawnOnExit(Battle)`.

- Flex compression instead of scrolling: cards have
  `flex_basis 44px / min_width 3px` and shrink to fit the window. At 20
  regiments they read as proper cards (kind letter, strength fill,
  morale strip); at 100+ (200k battles, reg_size 1000) they compress to
  slivers where kind color + strength fill is the whole message. The
  entire army is always on screen.
- Card anatomy: kind letter (H/S/L, clipped away on slivers), bottom-up
  strength fill (`count / initial_count`, whole-percent steps so relayout
  stays off the per-frame path), 4px morale strip on top
  (green -> yellow -> red, bright red routing, dark shattered), white
  `Outline` when selected, brightened on hover, near-black when dead.
  Fill drains to the broken-flag gray when the regiment breaks.
- Click selects the regiment, shift/ctrl-click toggles it into the
  selection; the mask is the same `Selection` resource the lasso writes,
  so both paths are one system.
- A hovered card writes `Hover.own` (after `update_hover`, overriding
  its terrain raycast), so the inspect plaque shows the regiment. The
  plaque moved up (`bottom: 76px`) to clear the bar.

## Control panel (bottom left)

Halt / Wall / Loose / Blob / Hold buttons: thin UI over the exact
hotkey code paths. `formation_keys` was refactored into
`FormCmd` + `apply_formation_cmd(cmd, selection, groups)` and the halt
body into `halt_selected` (orders.rs); keys and buttons call the same
functions. Toggle buttons light up when the whole controllable
selection is in the mode (matching the hotkeys' toggle-as-a-set
semantics: lit = next press turns it off). Disabled look with nothing
selected.

## Input plumbing

- Map input no longer fires through the HUD: `drag_select` (LMB lasso)
  and `issue_order` (RMB) skip press starts while the pointer is over
  any `Interaction` node (`unit_cards::pointer_over_ui`). Drags that
  started on the map may still pass over the bar.
- HUD buttons opt out of the global `button_hover_style` via the
  `CustomStyled` marker; they paint their own state-dependent colors.
- `card_clicks` is ordered `.after(drag_select)`: the over-UI guard
  reads hover state that lags synthetic same-frame move+click input
  (xdotool) by one frame, and if a click ever slips through both paths
  the card selection must deterministically win.

## Verified

20-regiment and 200-regiment battles on X: card click (outline +
count), shift-click multi-select, Wall button (plaque shows SHIELDWALL,
button lit active-blue), morale strips yellowing under pressure,
strength fills draining, sliver compression at 100 cards. No lasso
leak-through in logs. `FL_VOLUME=0` mutes test runs.

## Icon pass (second commit, owner-directed)

Panel moved to the right end of the bar; the five buttons became
round M2TW-style icon buttons (border_radius: BorderRadius::MAX,
which in Bevy 0.19 is a Node FIELD, not a component). Pictograms are
composed from plain UI nodes: stop square (Halt), three shields
(Wall), 2x3 square grid (Loose), scattered mob (Blob), planted shield
with boss (Hold). The Loose icon is stateful per the owner's spec:
spread squares in close order, tight squares once the whole selection
is loose; the icon previews what pressing yields. Both variants are
spawned stacked and refresh toggles Visibility.

Cards show node-drawn kind icons when wider than ~30 logical px
(knight's helm / leaf spear / arming sword), falling back to the kind
letter on slivers. Width is read from ComputedNode
(size() * inverse_scale_factor()); letter/icon swap via Display so
the hidden one leaves the flex layout.

## UI testing lessons (cost an hour, remember these)

- The owner's display runs a 1.25 scale factor: bevy UI logical
  coords * 1.25 = physical pixels for xdotool. A 1600x900 window is a
  1280x720 UI. Don't hand-compute layout targets from screenshots.
- The owner's near-fullscreen window silently eats synthetic clicks
  on DISPLAY=:1 (raise doesn't stick while they work). Do NOT fight
  the live display. Protocol that works: `Xephyr :2 -screen 1600x900`
  on :1, run the game with
  `VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.json` (llvmpipe;
  NVIDIA won't present into Xephyr) and drive it with
  `DISPLAY=:2 xdotool` — isolated cursor, no interference with the
  owner, no WM so the window is exactly 1280x720 at 1:1 scale.
- At llvmpipe frame rates use `mousedown 1; sleep 0.6; mouseup 1`,
  not `click` (press+release in one frame can be missed), and click
  IMMEDIATELY after spawn — FL_TEST_FRONT battles decay fast and
  card_clicks correctly ignores dead regiments.
- `FL_VOLUME=0` mutes test runs (owner asked for silent debugging).
- Full path verified end to end on :2: card click -> selection ->
  Loose button -> "1 regiments to LOOSE order" + tight icon + active
  blue button.

## Icon polish (third commit, owner feedback)

- Owner: "each unit card icon looks slightly different by pixels."
  Root cause: node-composed icons round every rect to the pixel grid
  independently, and at 125% display scale the 45-logical-px card
  stride is 56.25 physical px, so each card sits on a different
  subpixel phase and its icon rects round differently (slit shifts,
  dome width +/-1px). Fix: kind icons are rasterized ONCE per kind
  into a 40x44 texture (2x logical, software rounded-rect SDF with
  1px edge AA, straight-alpha src-over) and every card samples the
  same image via ImageNode. Identical by construction at any scale;
  the only residual is whole-quad +/-1px scaling, which reads as
  uniform softness, not warped geometry. Shared handles live in the
  KindIcons resource, built lazily on first battle spawn.
- Control buttons show a hover tooltip (dark pill, "Wall (F)" etc.),
  anchored to the button's right edge so it stays on screen.
- Verified on the Xephyr protocol: tooltip renders, all same-kind
  card icons pixel-identical at zoom. (Xephyr ignores
  WINIT_X11_SCALE_FACTOR and Xft.dpi, so fractional-scale conditions
  could not be reproduced there; the shared texture makes the fix
  scale-independent regardless.)

## Icon shape rework (fourth commit, owner feedback round 2)

- Pixel identity PROVEN numerically, not by eye: parchment-pixel
  clusters extracted from a fresh capture, same-kind icons diffed on
  icon pixels only -> max channel diff 0 for all pairs (helm x4,
  spear x3, sword x3). Earlier nonzero diffs were the cards'
  strength-fill backgrounds bleeding into the crop windows.
- Spear redrawn (was a capsule head = "mace/match"): pointed triangle
  leaf head on a longer 1.5px shaft. Sword redrawn (was a cross):
  longsword with tapered tip triangle, dark fuller, narrow guard,
  grip, pommel. Rasterizer gained a triangle SDF (iq's formula).
- Icon canvas 20x22 -> 20x24 logical (texture 40x48): both dims are
  integer physical at 125/150/200% scale, killing the last residual
  (quad height rounding 27 vs 28 phys px on neighboring cards).

## Button icon polish (fifth commit, owner feedback round 3)

- Blob icon: symmetric quincunx (was an irregular scatter).
- Wall icon: row of three heater shields with dark bosses (was three
  plain rects = "fence"). Second stateful variant like Loose: when
  ALL selected regiments are spears, it shows three spears braced at
  18 degrees (UiTransform rotation on UI nodes, works in 0.19) over
  a lower shield row — the spearwall the press would form. Toggled
  by WallIconVariant visibility in refresh_control_buttons.
- Verified on the Xephyr protocol including the live flip when a
  spear regiment card is selected.

## Button icons rasterized (sixth commit, owner feedback rounds 4-5)

Owner: wall icons bad ("fence", spearwall = "extra diagonal lines"),
tooltip must say Shield Wall / Spear Wall. Root cause: node-composed
pictograms can't hold detail at 22px. ALL button icons moved to the
SDF rasterizer (44x44 textures, ImageNode, ButtonIcons resource);
node-drawing helpers deleted. Rasterizer gained a Seg (capsule)
shape. Shield Wall = three heater shields (rounded top, tapering
point, boss) shoulder to shoulder; Hold = one big heater shield;
Spear Wall = five dense spears with leaf heads braced over a low
line (three sparse ones read as "stakes" per owner; five parallel
at ~68 degrees read as a bristling rank). Wall tooltip text swaps
"Shield Wall (F)" / "Spear Wall (F)" with the variant
(WallTooltipText marker). Verified per-state on the Xephyr protocol.

## Centering + cleanup pass (seventh commit, 0d6a688)

- Spear wall glyph recentered (bbox sat low-right); loose grid
  vertical recenter.
- Four-angle review (reuse / simplification / efficiency / altitude)
  applied: Selection::recount + picked_controllable de-dupe the
  mask/count invariant and the controllable predicate;
  MapInputSet -> HudInputSet named sets replace cross-module .after()
  on pub systems; CustomStyled + BTN_*/TEXT_COLOR palette live in
  game_state; banners::FLAG_BROKEN and unit_cards::BAR_HEIGHT shared;
  ControlButton = {Halt, Form(FormCmd)}; StatefulIcon and CardArt
  merge the variant marker pairs; refresh systems allocation-free
  with set_if_neq; is_spearwall_kind shared by wall_kind and the
  icon preview; icon textures rebuilt per battle entry (memoization
  resources dropped). Net -64 lines. Deliberately skipped:
  change-detection gating of refresh (risk > benefit at ~1000 UI
  entities), hover caching, rasterizer bbox clipping (one-time
  sub-ms), PointerOverUi resource (two call sites).
- PR draft: tmp/drafts/pr-unit-card-hud.md. Feature-complete pending
  owner play test.

## Next

Deployment phase + army picker (roadmap #1), then archers.
