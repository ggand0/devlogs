# PR #26: Update window title only when changed

**Author:** YelovSK (Patrik Hampel)
**Status:** Tested on Linux and macOS, ready to merge

## Summary

Prevents `send_viewport_cmd(Title(...))` from firing every frame by caching the last-sent title string and only issuing the command when the computed title actually differs.

## Change

- Extract pure `current_title(&self) -> String` from the old `update_title`
- Add `title: Option<String>` field to `App` struct
- New `update_title(&mut self, ctx)` compares against stored value, only sends viewport command on change

## Why it matters

Window managers that subscribe to title-change events (Komorebi on Windows, some tiling WMs on Linux) treat every `set_title` call as a real change event, even if the string is identical. This causes freezes/lag when multiple instances are running.

## Prior art in iced version

Same bug existed in the iced version of viewskater. Fixed in commit `d1f35f9`:
- Issue discussion: https://github.com/ggand0/viewskater/issues/52
- Original fix used the same pattern: compare `new_title != *last_title` before calling `window.set_title()`
- Earlier attempt used a `moved` flag which broke fullscreen transitions; the string-comparison approach was the final fix

## FPS baselines (measured on sorting-options branch, same results on main)

### macOS (M1 MacBook)

| Dataset | Key nav FPS | Slider FPS | Notes |
|---|---|---|---|
| 4K 10MB PNG | 50-55 | 14-16 | Occasional single stutter dropping to 30 in skate mode |
| 1080p 3MB | 60 | 40-45 | Consistent |
| small_images | 60 | 55-58 | |

### Linux (RTX 3090)

| Dataset | Key nav FPS | Slider FPS | Notes |
|---|---|---|---|
| 4K 10MB PNG | 60-64 | 14-15 | 60+ when lucky |
| 1K 3MB | 120+ | 30-40 | Reaches end quickly, uncertain ceiling |
| small_images | 145-146 | 61 | |

## Test plan (verified)

1. Open directory - title shows filename
2. Navigate forward/back - title updates
3. Open dual pane - title shows both filenames
4. Close one pane - title reverts
5. Open empty (no args) - shows "ViewSkater"
