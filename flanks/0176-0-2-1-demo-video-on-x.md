# 0176: The 0.2.1 demo video on X

Written on Windows 10 in the NTFS tree at `D:\ggando\gamedev\flanks`, for copying into the Linux devlogs. 2026-10-03. Windows clone `C:\Users\gotag\projects\flanks` on main at 42cbba4 (the `0.2.1` tag). Follows devlog 0174 and the release of 0.2.1 from Linux. Numbered 0175 in the D tree; 0176 here, since the Linux tree's 0175 is the Linux test of main and the go for 0.2.1.

## Summary

- 0.2.1 is out with all three files: the AppImage and the dmg uploaded from Linux and the Mac (2026-10-02 14:50 and 16:21 UTC), the Windows exe (`flanks_0.2.1.exe`, 193,922,048 bytes, assets embedded) on 2026-10-03 02:45 UTC.
- Gota recorded a 94 s demo in OBS on Windows, cut 12.5 s to 60 s out of it, and posted it on X: https://x.com/gtgando/status/2106225238266777837
- The posted file is `demo_recs/100326_0.2.1_demo_smoothcam_0012-0100_x.mp4`, 47.5 s, 355 MB. X refused the first cut, a 1.44 GB copy of the recording's own stream, as too large; the posted file is a two-pass x264 re-encode at 60 Mbps.
- For the next recording: set OBS's keyframe interval to 1 or 2 s, so cuts without re-encoding can start close to the wanted time.

## The recording

`demo_recs/100326_0.2.1_demo_smoothcam.mp4`, recorded 2026-10-03 at about 11:55, on main at the 0.2.1 tag:

- 94.35 s, 1920x1080 at 60 fps, H.264 High at about 206 Mbps (OBS with NVENC), AAC-LC 48 kHz stereo at about 176 kbps. 2.43 GB.
- A slow, smooth camera move along the two armies before contact, 200k soldiers. The stats line at the top left reads 204 to 215 fps at 200,000 soldiers between 8 and 10 s.
- 1920x1080 is the size X plays and 16:9; the maximized 2K window (2560x1360) is not. Gota asked for the `FL_WINDOW=1920x1080` launch just before recording.

## The cuts

Both cuts copy the recording's streams without re-encoding (`ffmpeg -c copy`), so they are bit-identical to the recording over their range.

- OBS put a keyframe every 250 frames, 4.17 s at 60 fps. A copy can only start on one, so 00:10 was not possible: the choices were 8.333 s and 12.5 s.
- With B-frames, each keyframe is decoded one frame before it is shown (the keyframe at 8.333 s has its decode time at 8.317 s). ffmpeg's output-side `-ss` compares decode times, so `-ss 8.333` skipped that keyframe and the cut started 4 s late at the next one, with only audio before it. That happened on the 2026-10-02 cut; the start has to sit between the decode times of the frame before and the keyframe (`-ss 8.31`, `-ss 12.475`).
- The end has the same problem the other way: a P-frame is decoded before the two B-frames shown ahead of it. Ending at 60.0 s kept the P-frame shown at 60.033 s and dropped the B-frames at 60.0 and 60.017, a two-frame jump in the last frame. Ending just before that P-frame's decode time keeps every frame through 59.983 s.
- Checked by frame hashes (`-f framemd5`) against the recording at both ends, and a full decode with no error.

| File | Range | Length | Size |
|---|---|---|---|
| `100326_0.2.1_demo_smoothcam_0008-0100.mp4` | 8.333 to 59.983 s | 51.7 s | 1.52 GB |
| `100326_0.2.1_demo_smoothcam_0012-0100.mp4` | 12.5 to 59.983 s | 47.5 s | 1.44 GB |
| `100326_0.2.1_demo_smoothcam_0012-0100_x.mp4` (posted) | the same, re-encoded | 47.5 s | 355 MB |

## X's upload limit

X refused the 1.44 GB cut as too large and took the 355 MB re-encode. Its published limit for a standard account is 512 MB per video (2 min 20 s). The recording's 206 to 240 Mbps puts that at about 17 to 20 s of 1080p60, so any posted clip from OBS at these settings needs a re-encode. X re-encodes every upload to a few Mbps anyway.

The posted file, from the 12.5 s cut, video only re-encoded, audio copied:

```bash
V="-c:v libx264 -preset slow -b:v 60M -maxrate 90M -bufsize 120M -profile:v high -level 4.2 -pix_fmt yuv420p -g 120 -bf 2"
ffmpeg -i in.mp4 -map 0:v $V -pass 1 -an -f mp4 NUL
ffmpeg -i in.mp4 -map 0:v -map 0:a $V -pass 2 -c:a copy -movflags +faststart out.mp4
```

Two passes give the size up front: 47.5 s at 60 Mbps is 355 MB. 84 s to encode on the 3900X. H.264 High at level 4.2, 1920x1080, 60 fps, AAC-LC, the moov atom at the front: the format X asks for. All 2850 frames, a full decode with no error. Gota's verdict after posting: looking good.

## The post

The text as posted (`work/drafts/x-post-0.2.1.md` in the D tree), with the release link in a reply:

> Still working on this game. After a lot of optimization, it can finally simulate and render 200k soldiers smoothly at 2K resolution. It's still missing lots of things, but I love watching mass battles like this lol #bevy

Reply: https://github.com/ggand0/flanks/releases/tag/0.2.1

"2K" is the maximized 2560x1360 window of the perf measurements (158 to 163 fps at Gota's usual ground spot in Gota's own battle after PR #23, devlog 0173). The clip itself is the 1920x1080 window.

## Notes from the same session

- The release exe needs `cargo build --release --features embed_assets`; a plain `cargo build --release` gives an exe that only runs where the repository's `assets` folder exists, and on the build machine it hides the mistake. The feature is opt-in because with it every asset edit recompiles the game crate and every exe carries 115 MB more; the AppImage and the macOS bundle carry `assets/` themselves. Committed on main, not pushed: 25ec337 "Describe the build profiles in the README", one line under `cargo run --profile opt-dev`: opt-dev for development and playing, `--release` for published builds, `--features embed_assets` to embed the assets. A `cargo release-exe` alias in `.cargo/config.toml` was proposed, not made.
- Fat LTO did not raise fps in 0.2.0 against opt-dev, as expected: at 200k the GPU is busy 95 to 98% of the frame, and LTO only speeds up CPU code. Its gain on the sim tick is unmeasured; likely a few percent at most, since opt-dev already builds at opt-level 3 with the crate's own codegen units optimized together, and the hot loops are in-crate and memory bound. Measuring it: release against opt-dev with `FL_LOD_PX=28,12,10` (GPU work taken away), tick job times at 300k.

## Index row for devlogs/README.md

| 0176 | 2026-10-03 | [The 0.2.1 demo video on X](0176-0-2-1-demo-video-on-x.md) | 0.2.1 complete with the Windows exe (assets embedded). A 94 s OBS recording at 1080p60 cut to 12.5-60 s without re-encoding (keyframes every 4.17 s; B-frame decode order decides where a copy can start and end), then re-encoded in two passes at 60 Mbps to 355 MB because X refused the 1.44 GB copy (512 MB limit). Posted: https://x.com/gtgando/status/2106225238266777837. README line on opt-dev, --release and embed_assets committed (25ec337, not pushed) |
