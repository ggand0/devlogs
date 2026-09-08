# PR #28: Animation Support (gif, animated webp, apng)

PR: https://github.com/ggand0/viewskater-egui/pull/28
Author: YelovSK (Patrik Hampel)
Merged: 2026-09-08 (main a8d6cf4, merge of e0bfcba on top of YelovSK's 275793b)

## Summary

Adds playback of animated gif, animated webp and apng. Three commits on the branch:
3a0cb54 and f9e133a "Add animation support for gif, webp, png/apng" (2026-05-24) and
275793b "Fix animation frame pacing" (2026-07-31). Nine files, +255/-35.

New module `src/animation.rs` with an `AnimationPlayer`. Every place in `pane.rs` that
used to assign `current_texture` now goes through `set_current_texture`, which drops any
previous player and starts a new one if the current path has an animation-capable
extension (gif, png, apng, webp). `App::update` calls `pane.poll_animation()` every frame.
`navigate`, `apply_slider_release` and `load_sync` gained a `ctx` parameter for this.

`file_io.rs` adds `apng` to the supported extensions, a `may_have_animation` check, and
`open_animation_frames`, which guesses the format with `ImageReader` and then builds the
gif, png or webp decoder from the `image` crate by hand, because the crate's generic
`decode()` only returns still images. Non-animated png and webp return `None` and the
player thread exits immediately. The .desktop files and the macOS plist gain the gif,
webp and apng MIME types.

## How playback works

The caches are unchanged. The sliding window cache, the decode LRU and the thumbnail
cache still decode only the first frame of a gif or webp through `open_image`, same as a
jpg. That is why skate mode is unaffected.

Playback is not from the cache. When an animation becomes the current image, the player
spawns a thread that reopens the file and decodes frames one at a time. Frames go over a
zero-capacity `sync_channel`, so the worker always has exactly one frame decoded ahead and
blocks until the UI takes it. The first frame uploads a fresh texture that replaces the
cached one on screen; later frames call `texture.set` on the same handle. When the worker
reaches the end it reopens the file and loops, unless the file had fewer than two frames.
Nothing holds more than two frames at a time, so memory is flat regardless of clip length.

Costs: every visit to an animation re-decodes from disk starting at frame one, every loop
re-decodes all frames, and frame one is decoded twice per visit (once by the cache, once by
the player). Dropping a player does not stop its thread instantly; the worker notices on
its next blocked `send`, so it lives at most one more frame decode.

## Merge with main

The branch had been force-pushed by YelovSK onto 570eccd (merge of #34), so it already sat
on top of the thumbnail cache and settings work. Only PR #27 (jxl) and the license commit
9fcc48b were missing. Merged main into the branch (not rebased, so the contributor's hashes
survive) as e0bfcba. Three conflicts, all one-line lists where jxl and apng were added in
the same place: `SUPPORTED_EXTENSIONS` in `file_io.rs`, the file dialog filter in
`handlers.rs`, and the MimeType line in `assets/viewskater-egui.desktop`. Resolved as the
union. `resources/macos/Info.plist` auto-merged with both. Build, 6 tests and clippy clean.

Not done: `cargo-appimage.desktop` gets a new MimeType line from this branch that has no
`image/jxl` entry, because main never had a MimeType line there. Minor follow-up.

## The frame pacing thread

hml-pip reported on 2026-07-31 that animated webp played slower than in other viewers.
YelovSK answered the same day that the next frame was only decoded after the current
frame's delay had elapsed and the timing was never corrected, so a 70 ms clip showed frames
72 to 73 ms apart, and pushed 275793b. Nobody had confirmed the fix, so we did.

### Reproduction, first attempt

Owner tested the merged branch: loop timing matched the browser, gif and webp panes stayed
in sync in dual pane. Then tried to reproduce the bug "at the old commit" but was running
`cargo run` in the repo, which was still on `animation-support`; the old commit had only
been checked out in a scratch worktree. Lesson: when the owner needs to run an old commit,
check it out in the repo itself, or say very clearly that the repo is not on it.

### Reproduction, by measurement

Built f9e133a (pre-fix) and 275793b (fix) in two scratch worktrees, each with the same
added log line printing the interval between consecutive frame arrivals in `poll()` next
to the frame's nominal delay. Ran each for 25 s on `giphy_DFRghyjqRDkJO` (130 frames,
80 ms each, 10.4 s per loop), gif and webp, on the Ubuntu desktop.

```
build            format  frames  nominal  actual   slow by  median interval
old f9e133a      webp    268     21.44 s  24.23 s  +13.0%   90.3 ms
old f9e133a      gif     270     21.60 s  24.39 s  +12.9%   90.3 ms
fixed 275793b    webp    305     24.40 s  24.40 s   +0.0%   82.8 ms
fixed 275793b    gif     305     24.40 s  24.40 s   +0.0%   82.5 ms
```

The old code adds a flat ~10 ms to every frame, identical for gif and webp, so it is not
decode time (which would differ between formats) but the round trip of deadline wake-up,
request to worker, decode, repaint request, next poll. The fixed build jitters per frame
(individual gif intervals ranged 37 to 123 ms) but the running total lands exactly on the
nominal timeline because each deadline is the previous deadline plus delay, so lateness is
paid back by the next frame.

Because both formats lose the same amount in the old build, comparing gif against webp in
dual pane does not show the bug. Only an external reference does: a browser tab or the
fixed build. That is consistent with hml-pip noticing it "compared to other viewers".

After checking out f9e133a in the repo itself, the owner confirmed by eye: the webp
`giphy_Cdkk6wFFqisTe` played slower than the browser, while gif and webp in dual pane
stayed together. On the fixed build the owner saw 60 fps skate on mac, 100 to 120+ fps on
the Ubuntu desktop, sync in dual pane, and webp at browser speed on both machines.

## Test data

`/home/gota/ggando/rust_gui/data/test_data/animations/`

- `gif/` 50 Giphy cat clips, `webp/` the animated webp of each same clip plus three
  samples (Google's animated Rubik's cube, a static webp, the Nyan cat animated-webp test
  image). Same Giphy ID in both filenames.
- `timing_loop/` uniform-delay clips for stopwatch loop counting, README has expected loop
  times (e.g. gstatic_animated_1.webp 100 frames x 100 ms = 10 s).
- `timing_pairs/gif` and `timing_pairs/webp` six clips numbered 01 to 06 in the same
  order, each gif with identical per-frame delays to its webp, for dual pane sync tests.

Sourcing notes: Giphy's public beta API key is banned and giphy.com returns 403 to curl,
but `media.giphy.com/media/<id>/giphy.gif` and `.webp` serve fine. IDs came from the
giphy.com search pages fetched through the WebFetch tool. Four IDs were single-frame and
one was a "content not available" placeholder, all dropped. Frame delays were read from
the files (GIF graphic control blocks, WebP ANMF chunks); Pillow reports 0 for webp
delays. Giphy's webp encoder rounds some delays differently from the gif (66/67 ms vs
60/70 ms), so only pairs with identical delays are usable for sync tests.

## Browser behaviour, for the follow-up

Firefox, verified in `modules/libpref/init/StaticPrefList.yaml`: pref
`image.animated.decode-on-demand.threshold-kb`, default 20*1024, "The maximum size (in kB)
that the aggregate frames of an animation can use before it starts to discard already
displayed frames and redecode them as necessary." Chrome's Blink keeps decoded frames in a
decoding store with a heap limit and can purge all frames except the current one; the exact
threshold at which it stops retaining animation frames was not pinned down.

Follow-up idea noted in the merge comment, low priority since playback is correct and
memory is bounded: keep decoded frames of small gifs/webps under a memory budget so loops
after the first are free and revisits do not restart from frame one.

## Review notes not acted on

- Animation decode goes through the codec decoders directly, not `open_image`, so the jxl
  registration hook is irrelevant to it. Fine.
- Dual pane: each pane owns its player, works.
- Skating through a folder of animations briefly stacks player threads, each dying after
  at most one more frame decode. Not observed as a problem at 50 files.
