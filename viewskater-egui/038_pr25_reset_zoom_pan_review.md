# PR #25 Review: Reset Zoom/Pan on Navigation

**Date:** 2026-06-14
**PR:** https://github.com/ggand0/viewskater-egui/pull/25
**Author:** YelovSK
**Branch:** `reset-zoom-pan-on-navigation`

## Summary

Adds a setting to reset zoom and pan when navigating between images. Defaults to `true`. All image viewers the contributor has used reset the view on navigation, and they argue it's better UX.

## Files Changed

- `src/settings.rs` - New `reset_zoom_pan_on_navigation` field on `AppSettings`, default `true`, toggle in settings UI, included in `SettingsChanges::pane_settings`
- `src/pane.rs` - New field on `Pane`, `reset_view()` helper, called in `navigate()`, `jump_to()`, `apply_slider_target()`
- `src/app.rs` - Pass new setting to `Pane::new` at both construction sites
- `src/app/handlers.rs` - Pass to `Pane::new` in `set_dual_pane`, propagate in `apply_settings_to_caches`

## Clippy

Clean pass, no warnings.

## Review Findings

### Fix before merge

1. **Missing space in settings UI** (`src/settings.rs:455`): `ui.horizontal(|ui|{` should be `ui.horizontal(|ui| {` to match surrounding style.

### Observations (not blocking)

2. **`Pane::new` growing parameter list** (now 6 params): Pre-existing pattern. Every new pane-level setting adds another positional arg. Future consideration: pass `&AppSettings` or a subset struct instead.

3. **`reset_view()` in `apply_slider_target`**: Resets zoom/pan on every slider drag tick, not just on release. If a user scrubs the slider while zoomed in, they lose zoom each frame. Matches the intent ("navigation resets view") but worth verifying the feel.

4. **`cache.summary()` hoisted before `reset_view()`** in `navigate()`: Harmless since `reset_view()` only touches `zoom`/`pan`, not cache state. Slightly unnecessary change but no issue.

5. **`reset_view()` extraction**: Good DRY improvement. Replaces inline `zoom = 1.0; pan = ZERO` in the double-click handler and reuses across 3 navigation paths.

## Verdict

Solid PR. One formatting nit to fix, then merge.
