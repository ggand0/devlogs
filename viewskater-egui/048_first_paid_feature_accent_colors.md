# First paid feature: accent color presets

PR: https://github.com/ggand0/viewskater-egui/pull/35
Merged into main 2026-09-11 as 3896941. Branch feat/official-accent-presets.
Ideas and rejected options: devlog 047.

## What shipped

Preferences > General gets a Theme section with an accent color picker:
Teal (the default), Cyan, Amber, Violet, and Custom, which opens egui's
own color picker (`ui.color_edit_button_srgb`). The accent is the color of
toggles, tab underline, links, radio dots and the navigation slider. It
applies on the next frame and is saved to settings.yaml as `accent_preset`
plus `custom_accent` (three sRGB bytes).

Only the paid build has the section. It is compiled with a new Cargo
feature:

    cargo run --release --features official

A plain build shows no Theme section and stays teal. The two settings
fields exist in both builds, so a settings file written by either loads
in the other; the free build ignores the value. Both configurations
compile without warnings.

## How it is wired

- `AccentPreset` enum in `src/settings.rs`, serde snake_case. `ALL`,
  `label()` and `fixed_color()` are behind `cfg(feature = "official")` so
  the free build has no dead code.
- `AppSettings::color_of(preset)` resolves a preset to a color, using
  `custom_accent` for `Custom`. `accent_color()` is the current one.
- `App::update` copies `settings.accent_color()` into `theme.accent` each
  frame before `apply_to_visuals`, behind the same cfg. One comparison per
  frame.
- The radio rows reuse the GPU Memory Mode row drawing code, renamed
  `radio_row` and made generic, with an optional color swatch after the
  label. The Custom row shows a "Pick a color" button when selected.
- The default teal moved to `theme::DEFAULT_ACCENT` with a `theme::rgb`
  helper; the theme, the Teal preset and the custom default read it.

## Why the feature is called `official`

Checked 2026-09-11. Krita, LosslessCut and Pixelorama ship identical code
in their store builds, so they have no flag to copy. The two precedents
for a build-time switch are Firefox, whose `official` branding set is the
trademarked artwork with `unofficial` for source builds, and Chromium,
where `is_official_build` means optimization level and the Google bits
are `is_chrome_branded`. `official` matches Firefox's use and the words
on viewskater.com and the Polar page. `paid`, `pro`, `premium` read wrong
for a flag anyone can enable. Renaming later is a search and replace over
Cargo.toml and the cfg attributes.

## Distribution state

- 0.3.0 free builds (AppImage, dmg, exe) on GitHub Releases, 2026-09-10.
- Polar product now lists only the 0.3.0 official builds, built from
  main at 3896941 with the feature. They report version 0.3.0 although the
  0.3.0 tag predates the feature. Next release number undecided (0.3.1
  with PR #20, or 0.4.0 with culling).
- cargo-appimage: `cargo appimage --features official` does pass the flag
  through. Verifying with `strings` on the binary does not work (the
  labels were not found even though the running app shows them); verify by
  running the AppImage.

## Tested

Ubuntu, X11, opt-dev and the release AppImage. Picked every preset and a
custom hue from the picker; toggles, tab, radio and slider recolored at
once; the Saved indicator fired; the value survived a restart; the free
build showed no Theme section. Captures in tmp/shots/accent_*.png.

## Open

- Cyan is close to Teal; drop or replace it.
- Whether the free build should show the Theme section with only Teal, so
  the paid build reads as "more options" rather than a new section.
- Empty-state trail animation stays on local branch
  feat/official-empty-state (a7087fd); too noisy for a utility. May be
  swapped for a small pixel-art idle or About-modal animation.
- Next paid cosmetics from devlog 047: app icon variant, About screen art
  with a thank-you line. Both need artwork.
