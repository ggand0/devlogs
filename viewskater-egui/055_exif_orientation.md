# EXIF orientation: the fix, the JXL exception, what the turn costs

Date: 2026-09-19
Context: branch fix/exif-orientation, three commits on top of main
915f9ea. 6b062cb is the fix and was pushed to origin the same day.
9443449 puts the panel's Orientation row back (section 2) and was
pushed by the owner. 0b02d6c adds one comment (section 9). PR #46,
opened by the owner from tmp/drafts/2026-09-20_pr_exif_orientation.md,
merged into main on 2026-09-20 as 060fb12. Numbers from gota-home
(Linux, RTX 3090, 144 Hz). Started from
tmp/handoffs/2026-09-19_metadata_panel_merged_orientation_next.md.
Section 7 lists which of its claims held.

## 1. The bug

Photos with an EXIF orientation tag were shown as the file stores the
pixels. A phone held upright stores a landscape buffer plus tag 6
("turn 90 CW to show"), so those photos came out on their side.
Nothing in src/ read the tag. The only mention was the panel's
Orientation row, which said in words which turn would fix the picture.

The owner remembered fixing this before. That was the iced version,
commit 3b2a479 "feat: EXIF auto-rotation" (2025-12-30, PR #77):
decoder.orientation() plus img.apply_orientation() in
src/exif_utils.rs. The egui version never had it.

The folder the owner reported,
/home/gota/Pictures/profile_pics/ALL_gigafile-0831-09c9546c37663d81a4b24e83ea82e05f,
361 JPEGs and one MOV, by `identify -format '%[EXIF:Orientation]'`:

| orientation | stored size | files |
|---|---|---|
| 6 | 4032x3024 | 198 |
| 1 | 3024x4032 | 149 |
| 1 | 4032x3024 | 14 |

The 198 were the sideways ones. The other folder, profile_pics/051723,
has orientation 1 or no EXIF and always showed correctly.

## 2. What was built

One value travels from the file to the conversion:
image::metadata::Orientation, the image crate's enum for the eight EXIF
values (NoTransforms, Rotate90, Rotate180, Rotate270, FlipHorizontal,
FlipVertical, Rotate90FlipH, Rotate270FlipH).

Reading it, src/file_io.rs:

- LoadedImage has a new field `orientation` (line 178). `image` still
  holds the pixels as the file stores them.
- decode_into (206) now returns ImageResult<(DynamicImage,
  Orientation)>. After the existing exif_metadata() call it asks
  exif_orientation(&mut decoder, format) (219). For a PNG whose eXIf
  chunk follows the pixel data, the bytes png_trailing_exif already
  fetches for the record also go through Orientation::from_exif_chunk
  (233).
- The trailing-EXIF lookup still runs when the pixel decode failed.
  The decode result stays a Result until the last line,
  `image.map(|image| (image, orientation))`. An early `?` there would
  have lost the record's EXIF for a broken PNG.
- load_image (192) unpacks the tuple. On a decode error the
  orientation is NoTransforms.
- exif_orientation (246) returns NoTransforms when `format` is None
  (section 3), else decoder.orientation(), with NoTransforms on an
  error. decoder.orientation() is per format inside the image crate.
  JPEG and WebP cache the value during exif_metadata(), which
  decode_into has just called. PNG uses the trait default, which calls
  exif_metadata() again and parses the chunk. TIFF reads the
  Orientation tag from the first IFD, so TIFF is turned although the
  panel has no EXIF for TIFF yet.

Applying it, src/decode.rs:

- image_to_color_image(img, orientation) (23): downscale_if_needed,
  then img.apply_orientation(orientation), then convert_image.
  convert_image is untouched.
- image_to_thumbnail(img, orientation) (86): resize to 400 px, then
  apply_orientation, then convert_image. The turn runs on the 400 px
  image. resize() fits a 400x400 box, so turning after it gives the
  same size as turning before it.
- The argument is required. A caller that forgets it does not compile.

Callers:

- cache.rs 691 (background decode thread), cache.rs 839 (decode_sync),
  pane.rs 183 (load_sync): pass loaded.orientation. Nothing else
  changed there.
- cache.rs 50, the slider preview worker: was image::open(path), now
  load_image(path) and image_to_thumbnail(img, loaded.orientation).
  Section 4.
- animation.rs 92 and 119: open_animation_frames (file_io.rs 334) now
  returns (Frames, Orientation) and every frame is converted with it.
  GIF is always NoTransforms. WebP uses exif_orientation. APNG takes
  the eXIf chunk from the decoder, else the trailing chunk, the same
  two places load_image looks. The reason: the first picture of an
  animated file comes from load_image and the player then swaps in its
  own frames. If the two disagreed the picture would flip when the
  animation starts.

Dims (metadata_panel.rs) and the footer resolution (menu.rs) read
texture.size(), so they show the turned size without a change.

The Orientation row. 6b062cb removed it with
ExifSummary::orientation, metadata::orientation_text, the test
orientation_as_a_turn, one assert in camera_section_from_a_block and
file_section's `exif` parameter. Plan 009 and devlog 054 (lines 152
and 342) said the row goes away once the turn is applied, and that
note was followed without asking. The owner on 2026-09-19: "why?
orientation is one of the most important exif params ppl care".
9443449 puts all of it back as it was on main. What the removal lost:
ALL EXIF has the tag only in kamadak's words, "row 0 at right and
column 0 at top", and Dims now reads 3024x4032 for a file that stores
4032x3024 with nothing in the panel saying why. Against main the two
files differ in three comments only. They said the app does not apply
the tag, and now say the row names the turn that shows the stored
pixels upright and that Dims is the size after it. The row is still
hidden for value 1. For a JXL it shows the Exif box's tag, which the
app does not apply itself (section 3). LABEL_W and its comment are as
on main.

## 3. JXL is not turned

The handoff did not have this. A JXL file has an orientation field in
its codestream. jxl-oxide 0.12.5 applies it while rendering:
lib.rs 527 and 533 (width and height "with orientation applied"),
lib.rs 721 (`.apply_orientation(&self.image_header)`), and the image
integration reads `render.stream()`, documented as "Orientation is
applied". So the pixels that reach us are upright already.

jxl-oxide's ImageDecoder implements exif_metadata() and not
orientation(). The trait default would therefore hand back the tag from
the Exif box. cjxl keeps that tag next to the codestream field.
`jxlinfo -v` on both files made from one orientation-6 photo:

    transcoded.jxl (cjxl in.jpeg, lossless JPEG transcode)
      Intrinsic dimensions: 4032x3024
      Orientation: 6 (90 degrees clockwise)
      box "brob": Brotli-compressed Exif metadata
    pixels.jxl (cjxl --lossless_jpeg=0 -d 2)
      same three lines

Applying the tag would turn these files a second time. In a JXL file
the codestream field is the one that counts. libjxl says so in
/usr/include/jxl/decode.h, lines 1321 to 1323, on the "Exif" box: "The
Exif orientation should be ignored by applications; the JPEG XL
codestream orientation takes precedence". The standard itself was not
read.

The test in code is `format.is_none()`. ImageReader::format() is None
exactly when the reader resolved to a decoding hook
(io/image_reader_type.rs: Format::Extension("jxl") has no ImageFormat),
and jxl-oxide is the only hook registered. format_name already leans on
the same fact. If a second hook decoder is ever registered, it gets no
orientation until this function learns about it.

## 4. The slider preview worker now uses load_image

It was the one decode site outside load_image, which made the sentence
in load_image's doc comment ("Every place that decodes an image for
display goes through here") untrue. Side effects of the switch:

- image::open picks the decoder by file extension only. load_image
  uses with_guessed_format(), so a JPEG named .png now gets a preview
  like it gets a main image.
- The worker builds a MetadataRecord and drops it. Devlog 054 section
  1 has that at under 0.1 ms per image.
- A PNG without EXIF costs the preview worker the 128 KB tail read too
  (79 µs median in devlog 054 section 10).
- ensure_image_decoders_registered() now also runs on this path.

## 5. What the turn costs, and the loop that was thrown away

First measurement, wrong. A scratch crate (image 0.25.9, release,
system allocator) on 30 of the orientation-6 photos, medians: decode
94 ms, apply_orientation 66 ms, plain RGB8 to Color32 conversion 31 ms.
A loop that writes the Color32 output in turned order, in 32 px
blocks, did conversion plus turn in 40 to 46 ms. On those numbers I
wrote that loop into decode.rs (one index formula, origin + x * step_x
+ y * step_y, for all eight orientations, tested against
apply_orientation on RGB8, RGBA8 and Luma8).

Second measurement, inside the app: temporary #[ignore] tests in
file_io.rs, `cargo test --profile opt-dev`, which links mimalloc like
the binary. Same photos, medians, two runs each:

| step | ms |
|---|---|
| load_image (decode plus record) | 84.7 |
| plain conversion, no turn | 8.0 to 8.6 |
| apply_orientation, then plain conversion | 34.9 to 35.8 |
| the loop in decode.rs | 38.0 to 38.6 (42.3 in the first run) |
| 64x64 local buffer: rows converted, then written turned | 34.7 to 35.0 |

And from a separate run of 12 files with loops written for the quarter
turn only: strided reads in 32 px blocks 32.2, strided writes in 64 px
blocks 40.5, plain row order with no blocks 31.9, filling 12 M Color32
4.7.

The scratch numbers were mostly page faults. The system allocator maps
fresh pages for every 36 or 48 MB buffer, mimalloc reuses them. The
plain conversion alone went from 31 ms to 8 ms. In the app every way
of turning a 12 MP image lands between 32 and 40 ms for turn plus
conversion, so the turn itself is about 27 ms whoever does it, and the
general loop was the slowest of them. It was deleted. decode.rs calls
apply_orientation. The best hand-written variant would save about
3 ms of 35 and would need one loop per orientation.

So per photo with a quarter turn: 85 + 8 = 93 ms before, about 120 ms
after. On decode threads in skate mode, on the UI thread in load_sync
while the slider is dragged. Photos without the tag or with tag 1 run
the same code as before plus one match arm. A 180 degree turn and the
two flips are done in place by the image crate and were not timed.

The one way to make it free is to upload the pixels as stored and draw
the quad with turned texture coordinates. Then every reader of
texture.size() has to know the orientation: cache.rs (LRU bytes, which
would not care), app.rs (slider preview), pane.rs (fit, zoom, the
painter.image call, which would become a four-vertex mesh because
painter.image only takes an axis-aligned uv rect), metadata_panel.rs
(Dims), menu.rs (footer resolution). Not done, not planned.

## 6. Verification

Tests, 87 pass at 6b062cb, 3 ignored, clippy clean with --all-targets:

- file_io orientation_comes_from_the_exif_tag: a 3x2 image with a
  26-byte EXIF block holding only tag 0x0112 = 6, written as a JPEG
  (JpegEncoder::set_exif_metadata), as a PNG with eXIf before IDAT
  (PngEncoder::set_exif_metadata) and as the same PNG with the chunk
  moved in front of IEND. A moved chunk keeps its CRC. All three
  report Rotate90, have a Present record and convert to 2x3. A PNG
  without EXIF reports NoTransforms and stays 3x2.
- decode conversion_applies_the_orientation: red left of blue, turned
  a quarter clockwise, is red above blue.
- decode thumbnail_is_turned_after_the_downscale: 1000x500 gives
  400x200, and 200x400 with Rotate90.
- metadata orientation_as_a_turn was removed in 6b062cb and is back in
  9443449. 88 tests pass after it.

Real files, through a temporary probe test (not committed). One
orientation-6 photo, and copies made with ImageMagick 6 (`convert
in.jpeg -resize 25% out.webp|tiff|png`) and cjxl:

| file | decoder gives | turn applied | shown |
|---|---|---|---|
| rot6.jpeg | 4032x3024 | Rotate90 | 3024x4032 |
| rot6.webp | 1008x756 | Rotate90 | 756x1008 |
| rot6.tiff | 1008x756 | Rotate90 | 756x1008 |
| rot6.png (trailing eXIf) | 1008x756 | Rotate90 | 756x1008 |
| transcoded.jxl | 3024x4032 | NoTransforms | 3024x4032 |
| pixels.jxl | 3024x4032 | NoTransforms | 3024x4032 |

The probe also wrote the converted pixels to PNG. `compare -metric
RMSE`, normalised value:

- JPEG against `convert rot6.jpeg -auto-orient`: 0.0005. Two JPEG
  decoders, same picture.
- PNG against `convert rot6.png -rotate 90`: 0, exact.
- transcoded.jxl against the auto-oriented JPEG: 0.002.
- For scale, the PNG against the wrong quarter turn: 0.25.

The animation path, checked after the push with the same kind of
probe. An animated WebP, two lossless frames of 200x150 (`img2webp`),
with the EXIF block of rot6.webp put in by `webpmux -set exif`.
load_image reports Rotate90 and the still converts to 150x200.
open_animation_frames reports Rotate90 too, and both frames convert to
150x200. With NoTransforms the frames stay 200x150, which is what the
player would have shown without the change in animation_worker: the
still upright, then on its side from the first frame on.

ImageMagick 6 here does not take the orientation from a PNG eXIf
chunk. `identify -format '%[orientation]'` prints Undefined for
rot6.png and `-auto-orient` leaves it 1008x756. It is no reference for
PNG. An explicit `-rotate 90` is.

Not done: nobody has looked at the branch in the app window yet.
`--bench-nav` on the reported folder, main against the branch, waits
for the owner's OK to open a window. No Windows or macOS check. The
change has no platform code.

The photos in the reported folder have no Make or Model, so the
ignored test real_photos_have_camera_fields fails on them at its
camera assert. It was left as it is. Its "orientation:" line now
prints the tag text and loaded.orientation.

## 7. Handoff claims, checked on 2026-09-19

- "Nothing in src/ reads the tag": true.
- image 0.25.9 API (ImageDecoder::orientation at io/decoder.rs 53,
  apply_orientation at images/dynimage.rs 1161, from_exif and
  from_exif_chunk public, the JPEG decoder caches during
  exif_metadata()): all true. WebP caches the same way.
- "The one place to change for the main image is decode_into, then
  apply it to the decoded image": the read is there. The turn is in
  decode.rs after the downscale, so that thumbnails are turned small.
- PNG trailing chunk needs from_exif_chunk on the fetched bytes: true.
- Other decode sites, cache.rs 50 and open_animation_frames: true,
  both changed.
- "Dims and the footer follow by themselves": true.
- Missing from the handoff: JXL must be skipped (section 3).
- "ImageMagick `-orient right-top` should set the tag without touching
  the pixels": not needed and not tried. The owner's folder has 198
  real files.

## 8. Process

- The scratch-crate timing led to 55 lines of index math that were
  written, tested and deleted. Time pixel-buffer code inside the app's
  build. Memory: feedback_measure_inside_the_app_build.
- `git switch -c` for the branch. decode.rs was reset to main's
  content by writing the file, with a copy of the loop version in the
  session scratchpad, which is gone with the session.
- Temporary probe and timing tests lived between TEMP-PROBE markers in
  file_io.rs and were removed before the commit.

## 9. After the first push, 2026-09-19 and 2026-09-20

The two EXIF reads. parse_exif (kamadak) parses the whole block for
the panel and the image crate scans the same bytes for tag 0x112. The
file is read once. Timed in the app's build on the 8604-byte block of
an orientation-6 iPhone photo: Orientation::from_exif_chunk 19 ns,
PNG decoder.orientation() (fetches the chunk from the decoder a second
time) 98 ns, parse_exif 4506 ns, the decode about 85 ms. JPEG and WebP
store the value during the exif_metadata() call that decode_into makes
anyway, on main too. The value is not taken from our own parse because
the record holds display strings, the picture would then only be
upright when the panel's parse succeeds, and TIFF has no record EXIF.
0b02d6c puts one line above the call in decode_into saying so.

The slider preview and wrong extensions. The content sniffing is
with_guessed_format() in decode_into (file_io.rs 207, from d9c14ce in
PR #45), not new code. image::open is ImageReader::open(path)
.decode(), extension only. One of the owner's JPEGs copied as
jpeg_named.png: image::open fails with "Invalid PNG signature",
load_image gives 4032x3024, format JPEG, Rotate90. So on main such a
file has a main image and no preview, on the branch both. It is the
same change as the iced fix a71aaf3. A JXL named .jpg already worked,
jxl-oxide registers its two signatures with the image crate's format
detection (integration/image.rs 487 and 488).

Still open in that area: may_have_animation (file_io.rs 29) goes by
extension. A two-frame GIF copied as gif_named.jpg loads as a still
(format GIF), may_have_animation is false, and open_animation_frames
would return both frames if it were asked. No error, it just never
plays. The record already has the real format, so start_animation
could ask that.

Iced issue ggand0/viewskater#65 "Error on mismatched file extensions"
(eye-wave, 2025-11-30, open, the owner answered with a71aaf3). Left
in the iced repo on 2026-09-11 on purpose. Decision on 2026-09-20:
transfer it to the egui repo after the PR for this branch is open,
then add "Resolves #N" to the PR body. The iced README's "Issues moved
to the egui version" table (data-viewer clone) needs a row then.

More pixel checks for the PR text. rot6.tiff against `convert
-auto-orient`: RMSE 0. rot6.webp against `convert -rotate 90`
(ImageMagick 6 reports no orientation for the WebP either): RMSE 0.

Windows. The owner tested in the app on Linux and macOS and skips the
Windows test. The diff has no cfg, no path handling and no new file
system call. `cargo check --target x86_64-pc-windows-gnu
--all-targets` passes on a scratch export of c0333cd with the winres
`if` in build.rs skipped. c0333cd was the comment commit before the
owner reworded the comment. It was amended into 0b02d6c, which has the
same code.

The rule "read every file in tmp/drafts before a draft" is now "the
five most recent", in docs/internal/pr_draft_guidelines.md and in
memory, at the owner's request (17 drafts by now).

The cost sentence, checked on 2026-09-20. The owner asked whether
"adds about 27 ms to the 93 ms" was measured. It was put together from
two runs of section 5: 93 is load_image 84.7 plus the plain conversion
8.6, and 27 is 35 minus 8 from the run where the test called
apply_orientation by hand. The committed image_to_color_image had not
been timed as a whole. Timed now, same 30 photos, opt-dev build, load
average 7.5, three runs, medians in ms:

| | run 1 | run 2 | run 3 |
|---|---|---|---|
| load_image | 81.6 | 80.6 | 84.4 |
| image_to_color_image, NoTransforms | 8.6 | 8.2 | 8.4 |
| image_to_color_image, Rotate90 | 37.6 | 35.9 | 38.2 |
| per photo, load plus convert, no turn | 89.8 | 88.9 | 93.3 |
| per photo, load plus convert, turned | 119.4 | 117.9 | 124.5 |

The turn adds 29 to 31 ms on 89 to 93 ms. The PR draft now says about
30 ms on 90 ms. The commit message of 6b062cb on origin says 27 and
93 and was left alone. Still not measured: skate mode images per
second and the slider's UI block on a folder of turned photos, main
against the branch (--bench-nav and --bench-slider, which open a
window).
