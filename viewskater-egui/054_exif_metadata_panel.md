# EXIF metadata panel: design change, implementation, viewer research

Date: 2026-09-16 to 2026-09-18
Context: plan 009 branch 1 on feat/metadata-panel, seventeen commits
24538c8 to 47d09d2 on top of main d7ccd27. Pushed to origin on
2026-09-18 after a message-only rewrite (bodies wrapped at 72 columns
like main, five phrases reworded); same trees, authors and dates. The
hashes before the rewrite are under
backup/feat-metadata-panel-before-reword. Numbers from
gota-home (Linux, RTX 3090, 144 Hz). Claims not verified are marked.
Plan 009 was rewritten on 2026-09-16 before building; the version
before that is tmp/backups/2026-09-16_009_exif_metadata_panel_before_
non_retention_rewrite.md. Where this devlog and plan 009 disagree
(sections 3, 4 and 10 of the plan), this devlog is what was built.

## 1. Design change before building

The plan review on 2026-09-16 changed the storage and added seven
smaller points. All of them are in the rewritten plan 009 section 6.

Storage. The first plan kept every record in a
HashMap<PathBuf, MetadataRecord> on the app, capped at a few thousand
with oldest-first eviction, looked up by path every frame. Dropped:
the only consumer is the panel (and later the footer line) for the
image on screen, and every way an image reaches the screen already
produces or carries its record. Keyboard navigation moves only when
the sliding window slot has the texture, and the texture and record
arrive in the same DecodeResult. Slider drags go through load_sync,
which hits the texture LRU (entry carries the record) or decodes on
the spot. Folder open and slider release go through initialize, which
decodes the centre image synchronously. A record kept from an earlier
visit is only reachable by revisiting, and a revisit either hits a
texture that still has its record or re-decodes. So the record lives
next to the texture:

    pub struct Loaded { texture: TextureHandle, record: Arc<MetadataRecord> }

in the sliding window slots, the upload queue and the DecodeLruCache
entries, plus Pane::current_record next to current_texture. No table,
cap, eviction, per-pane ownership question or trash bookkeeping.
A named struct, not a tuple: later per-image facts derived from the
file bytes (applied orientation, ICC name) go in as fields. Editable
state (ratings) and anything needed for images not on screen (sort
keys, search) do not belong there; those need a folder scan.

The other points:

1. Bounded formatting on the worker. kamadak formats a whole value
   before it can be cut (tag.rs d_sub_hex, d_sub_comma). Nikon D750
   JPEGs carry a thumbnail IFD whose StripOffsets and StripByteCounts
   are arrays of 4016 numbers and a 5.6 KB private text tag;
   to_string() then truncate took 235 to 633 µs per image, ten times
   the rest of the record. Fix: a fmt::Write target that returns Err
   after 200 characters, which stops the formatter, and binary values
   over 64 bytes plus MakerNote shown as a byte count. 31 to 66 µs on
   every measured file.
2. catch_unwind around the whole decode thread body. Before, a panic
   dropped the sender unsent; poll never removed the path from
   running_decodes, request_decode skipped that image forever and one
   decode slot stayed busy until a slider jump rebuilt the window.
3. The record is built before the pixel decode, so it exists when the
   pixels fail (File rows and the footer size next to "Failed to load
   image").
4. load_sync (pane.rs) is a third decode site besides spawn_thread and
   decode_sync; all three call file_io::load_image.
5. All EXIF draws only the rows in the scroll viewport.
6. into_decoder() replaces decode(), so the allocation check decode()
   made (Limits::reserve of total_bytes) is repeated.
7. An "Exif\0\0" prefix is stripped before read_raw.

Measured cost of the record (scratch program, same crate versions,
release, files in the OS file cache, median of 5), microseconds:

| Step | Nikon D750 24 MP, 46 KB EXIF, 64 tags | Canon, 5.8 KB, 56 tags | iPhone 15, 12 KB, 67 tags |
|---|---|---|---|
| exif_metadata() | 17 | 13 | 11 |
| std::fs::metadata | 3 | 3 | 2 |
| kamadak read_raw | 8 | 7 | 4 |
| format all tags, bounded | 63 | 46 | 31 |
| format all tags, naive | 235 | 16 | 57 |
| pixel decode, for scale | 170,936 | 112,425 | 11,164 |

JPEG: image 0.25.9's JpegDecoder reads the whole file into memory at
construction and exif_metadata() re-parses headers on that buffer.
PNG: the eXIf chunk is in the decoder's info after the header read
(only when it precedes IDAT). WebP: one seek and one read. JXL:
through the image hook, jxl-oxide feeds bytes until the Exif box is
complete. TIFF: none yet, File rows only.

## 2. What was built, by commit

- 24538c8 src/metadata.rs. MetadataRecord { file_size, modified,
  format, exif: ExifData }, ExifData::{None, Unreadable,
  Present(Box<ExifSummary>)}, formatters, the Bounded writer,
  parse_exif(bytes) with continue_on_error and
  distill_partial_result so one bad tag does not lose the block.
  Tests build EXIF blocks with kamadak's experimental::Writer.
- d9c14ce file_io::load_image(path) -> LoadedImage { image:
  ImageResult<DynamicImage>, record: Arc<MetadataRecord> }. stat,
  ImageReader::open, with_guessed_format, format name (hook formats
  such as JXL have no ImageFormat, the extension names them),
  into_decoder, exif_metadata, Limits reserve, from_decoder.
  open_image is gone.
- 4f4bc85 Loaded in slots, PendingUpload, DecodeLruCache entries;
  current_texture_for became loaded_for; initialize and jump_to return
  the centre record; Pane::set_current, set_current_failed,
  show_sync_result. poll_animation swaps current_texture directly and
  never touches the record.
- 77f1c48 catch_unwind, always send; panic_message for the log.
- 30247dc Box the summary (clippy large_enum_variant).
- b82c27f the panel: SidePanel::right added after the menu bar and
  before footer and slider, default 260, range 200 to 480, width saved
  when ctx.read_response(panel_id.with("__resize")).drag_stopped(),
  two-tab strip in dual pane, Pane::take_image_click switches the tab,
  action row with trash_button, settings show_metadata_panel,
  metadata_panel_width, metadata_all_exif_open, View menu switch,
  Preferences > Display row, ctx.wants_keyboard_input() guard at the
  top of handle_keyboard, footer size from the record instead of
  std::fs::metadata per frame per pane.
- 9db881b tag list height fix, see section 4.
- 4d92275 text selection colour #264F78 with light text instead of the
  accent; also the selected item in the Preferences sort dropdowns,
  the only other user of egui's selection colour here.
- 698570d ignored test real_photos_have_camera_fields:
  VIEWSKATER_EXIF_FILES=a.jpg:b.jpg cargo test real_photos -- --ignored
  --nocapture; VIEWSKATER_EXIF_TAGS=1 prints every tag.
- 14e34f7 the I key (modifier ignored, so Cmd+I from issue #40 still
  works), panel follows the toggle in fullscreen, fullscreen FPS
  overlay sits left of the panel.
- 33a4274 tag rows became labels (drag select and copy), f-number and
  focal length keep two decimals, zero exposure compensation hidden,
  hover names on values (removed again in 864eae3 when labels became
  visible).
- 864eae3 dense label and value rows, section 6.

- 889fae1, 62f349e, c366244: section 10.

84 unit tests pass, 3 ignored. Clippy clean with --all-targets.

## 3. Decisions that differ from plan 009 as written

- Shortcut is I, not Ctrl+I. Every other toggle is a bare key (Tab,
  1, 2) and a hand is on A and D while culling.
- Fullscreen: no edge reveal; the toggle applies there too.
- Panel state persists across runs (settings field, like Footer).
  Intended.
- No cards and no values-only rows. Label and value rows, 12 px
  proportional, 18 px pitch, flat headings.
- Capture card dropped. Program, metering, white balance, flash are in
  the tag list only; their fields and formatters were removed from the
  record.
- Orientation is a row under Dims in FILE, shown only when the picture
  is turned, and goes away when orientation is applied on decode.
- Focal length leads with the 35 mm equivalent when it exists and
  differs: "24 mm (6.86 mm)", "180 mm (120 mm)".
- f-number as stored: f/1.78, as Preview and Photos show it.
- Exposure compensation only when not zero, labelled "Comp.".
- Folder row cut from the left at a path separator.
- "Open in map" stays: one Hyperlink with an OpenStreetMap URL.

## 4. The twitch at the end of long tag lists

Reported on nikon_d500/1.jpg and the iphone14_pro folder. The list
draws only visible rows and stands in for the rest with add_space.
Drawn rows got egui's item spacing (3 px), the space did not, so the
list height was n * row_h + 3 * drawn. Near the end fewer rows are
drawn, the content shrinks, the scroll area clamps its offset, the
next frame draws a different count. Headless test before the fix,
heights at offsets 0, 300, 1500, 3500, 4200, 9000:
[4089.75, 4089.75, 4089.75, 4068.75, 3993.75, 3993.75]. Fix: zero item
spacing inside the list scope. The test now checks the height is the
same at six offsets.

egui lays out without a display: Context::default() plus ctx.run with
a RawInput screen_rect. The panel tests use that for the heading
buttons' right edge, exact row height for long, empty and elided
values, the list height, and elide_start.

## 5. Verification

Real app, 2026-09-17, scratch XDG_CONFIG_HOME so the real settings
were untouched, Nikon D750 JPEG and a WebP: four sections filled,
"No EXIF data", tag list scroll with hover on cut rows, filter 64 to
4 tags, typing "a" and "d" in the filter without navigating, toggle
off and on, resize drag saving 338.4 to settings, dual pane tabs,
click on the right pane switching the tab. Screenshots were in the
session scratchpad, not kept.

Not exercised by me: fullscreen toggle, copy buttons, the map link, a
drag selection across tag rows, and the dense layout of 864eae3, which
has only the headless geometry tests behind it.

Two false alarms. "copy path" looked cut off in a downscaled
screenshot; pixel rows showed it ends at the content edge under egui's
2 px floating scrollbar. 243 navigation steps in the log began the
instant a script released focus from the filter box after xdotool had
typed "a" and "d"; xdotool left the keys repeating, the guard had
blocked them while the box had focus.

Ground truth: ImageMagick identify -format '%[EXIF:*]' against the
record for nikon_d500/1.jpg, iphone14_pro/IMG_3613.jpeg and
IMG_6005.jpeg. All curated fields match (1/80 s, f/4.5, ISO 100,
120 mm with 180 eq., aperture priority; 1/60 s, 89/50, ISO 160,
343/50 with 24 eq., +09:00; iPhone 7 GPS 35.6802, 139.7630, 10 m).
exiftool is not installed here. On the MacBook: exiftool -a -G1,
Preview's Inspector, Photos Cmd+I, XnView MP.

## 6. Viewer research and the dense layout

What photographers' tools put in their primary info view. Verified
from the programs' own field definitions:

- darktable, metadata_view.c _labels: model, maker, lens, aperture,
  exposure, exposure bias, exposure program, white balance, flash,
  metering mode, focal length, 35mm equiv focal length, crop factor,
  focus distance, ISO, datetime, width, height, latitude, longitude,
  elevation (file and tag lines around them).
- digiKam, itempropertiestab.cpp Photograph Properties: Make, Model,
  Created, Lens, Aperture, Focal, Exposure, Sensitivity, Mode/Program,
  Flash, White balance. Image Properties: Type, Dimensions, Aspect
  Ratio, Bit depth, Color mode, Sidecar.
- Capture One Metadata tool (imagealchemist.net): groups EXIF-Camera,
  EXIF-Exposure, EXIF-GPS with Show on map, Vendor Specific.
- Lightroom Classic (Adobe's description): Default view is file name,
  copy name, folder, rating, label and a subset of EXIF; EXIF view
  adds path, dimensions, exposure, focal length, ISO, flash.
- Apple Photos (support page): device, date, lens and camera
  settings, file size, location with a Maps link. On iOS the six from
  issue #7: camera, ISO, focal length, EV, aperture, shutter.
- From memory, not checked: Google Photos and Windows Photos show the
  same set plus dimensions with megapixels.

Everyone shows make and model, lens, aperture, shutter, ISO, focal
length with the equivalent, exposure compensation, date taken. The
editors add program, metering, white balance, flash and GPS. Nothing
else appears in a primary view.

The owner's screenshots in resources_dev/ (digikam_exif.png,
lightroom_exif_ss.jpg, photo_mechanic_exif_ss.png, xnview_exif.png,
xnview_properties.png):

- Lightroom and XnView Properties are curated views. Both start with
  file facts (name, folder, size, type; rating and label directly
  with them), then image facts, then exposure before make and model.
  XnView's Camera group: Exposure Time, F-Number f/1.78, True ISO,
  Model, Date taken, Lens model, Focal length. Lightroom ends with
  GPS, Altitude, Direction.
- digiKam EXIF, XnView EXIF and Photo Mechanic Exif are raw tag lists
  (digiKam alphabetical in two groups, the others in file order) and
  correspond to All EXIF. digiKam prints "Binary data 1,526 bytes" and
  cuts long values with an ellipsis.
- Every row in all five is label plus value.

Density, measured off the screenshots:

| | Font | Row pitch | Rows in about 700 px |
|---|---|---|---|
| Lightroom | ~11 px proportional | 16 px | about 40 |
| Photo Mechanic | ~12 px proportional | 20 px | about 40 |
| digiKam | ~13 px proportional | 21 px | about 42 |
| XnView | ~13 px proportional | 22 px | about 36 |
| ours before 864eae3 | 13 px monospace, padded cards | 22.5 px plus card chrome | 14 |
| ours, 864eae3 | 12 px proportional | 18 px | about 36 |

Layout as built: TEXT_SIZE 12, ROW_H 18, LABEL_W 68 right-aligned
muted labels, LABEL_GAP 8, headings 10.5 px, the tag list with a 45%
name column in the same rows.

    FILE                         copy name  copy path
            Name  IMG_3613.jpeg
          Folder  …/Pictures/profile_pics
            Size  561.7 KB · JPEG
            Dims  1536x2048 · 3.1 MP
     Orientation  Rotate 90° CW           only when turned
        Modified  2023-05-17 17:24
    CAMERA
        Exposure  1/60 s  f/1.78  ISO 160
           Comp.  +0.3 EV                 only when not zero
           Focal  24 mm (6.86 mm)
          Camera  Apple iPhone 14 Pro
            Lens  iPhone 14 Pro back dual wide cam…
           Taken  2023-05-17 12:37:22 +09:00
    LOCATION                              only with GPS
          Coords  35.6802, 139.7630
        Altitude  10 m
                  Open in map
    ALL EXIF (49)

Ten rows for a typical photo, fifteen at most; the tag list starts
about 250 px down. Not added on purpose: colour depth and ICC profile
name. The profile name is worth showing when the app does something
about colour management.

## 7. Benchmarks, 2026-09-17

docs/benchmarks.md command, 3 runs, settings as the baseline
(cache_count 5, decode_threads 10, lru 1024, gpu Performance). The
machine was not idle: load average about 8, a browser process at 86%
CPU. Summaries in benchmarks/20260917_09*_gota-home_summary.md, each
with its label. Skate, right pass, images/s and decode median:

| Folder | Baseline idle 09-16 | main same session | branch, panel off | branch, panel on, All EXIF open |
|---|---|---|---|---|
| small_images 4318 JPEG 800x800 | 144.7, 6.1 ms | not run | 145.3, 5.8 ms | 145.4, 5.6 ms |
| 1080p_PNG_3MB | 131.1, 30.3 ms | 129.1, 31.6 ms | 114.2, 35.0 ms under the heaviest load | 131.6, 31.6 ms |
| 4k_PNG_10MB | 60.5, 76.1 ms | 62.1 and 61.2, 74.9 and 77.0 ms | 55.2, 58.6, 60.1; 79.5, 78.4, 78.3 ms | 58.4, 78.3 ms |
| nikon_d500, 50 JPEG 24 MP, 63 tags | none | not run | 21.1, 205 ms | 23.4, 200 ms |

JPEG and 1080p: no cost. Panel drawing: no cost in images/s on any
folder; nikon_d500 frame p99 42 to 48 ms with overlapping ranges.
Slider on 1080p and 4K within a few percent of main (4K UI block per
load 63.8 vs 63.0 ms, click to image 176 vs 178 ms).

Open: 4K PNG trends 2 to 4% slower on the branch over five runs while
main sits at the baseline. The old and new decode paths alone are
identical (4K PNG 56.6 vs 56.4 ms, 1080p 18.0 vs 18.1, scratch
program alternating the two). Nothing in the change touches pixels and
1080p shows no shift, so the reading is load sensitivity of ten 33 MB
decodes sharing memory bandwidth. Not proven. Close it with the
baseline command on an idle machine on main and on the branch. main
was built from git archive into the scratchpad, no checkout.
devlogs/FPS_BASELINES.md was not updated.

## 8. Process

- xdotool on the live display. The app window was closed from the
  desktop mid-session; the next script sent a click, Ctrl+A,
  BackSpace and Escape to whatever had focus, which was the owner's
  browser. No input injection on the live display again. No Xvfb on
  this machine; Xephyr would still open a window. Benchmarks drive
  themselves and send no input, so they are safe to run; screenshots
  with import -window send no input either, but a new window can take
  focus while the owner types, and Delete trashes.
- rm -rf on a scratch folder made a minute earlier. Against the rule
  regardless of whose folder it was.

## 9. Open

- Orientation is not applied anywhere (nothing in src/ reads the tag;
  decode() never asked the decoder). Next branch: decoder.orientation()
  and DynamicImage::apply_orientation in load_image, the same in the
  thumbnail cache, then drop the Orientation row.
- Single pane keyboard navigation dead after deselecting pane 1 in
  Ctrl+3 and switching to Ctrl+1. Likely set_single_pane only
  truncates, the mode stays Independent and pane 0 stays unselected.
  Own branch. In memory.
- Checked by the owner on 2026-09-18: the dense layout, a drag
  selection across tag rows with Ctrl+C, I in fullscreen, the map
  link, the two copy buttons, the label column at 200 px.
- 4K idle rerun, sections 7 and 10. Optional now.
- A setting for the map site (OpenStreetMap, Google Maps, Apple Maps),
  one URL template each. Own PR. OpenStreetMap is the panel's choice in
  Location::map_url, not egui's; the Hyperlink opens any URL.
- No JXL file with EXIF to test; cjxl is not installed here.
- Pressing I while Preferences is open toggles the panel behind the
  backdrop, as Tab does for the footer.
- Branch 2 of plan 009, the footer exposure line.
- TIFF and RAW need the IFD walker; .tif shows File rows only.
- resources_dev/ is not ignored and holds two AppImages (330 MB,
  109 MB).

## 10. After the owner's first look, 2026-09-18

Double tooltip (889fae1). Hovering a cut tag value showed two
overlays. egui's Label already shows the full text on hover when its
galley was elided (egui 0.31.1 widgets/label.rs line 256), and the
cells added their own on_hover_text plus a layout_no_wrap to decide
when. Both removed, one layout per cell less. The folder row is cut
from the left by the panel, so it keeps its own tooltip and uses
Label::extend() so egui does not cut it again.

Other containers. The iPhone 7 JPEG converted with ImageMagick 6 to
WebP and PNG (25% size, scratchpad). WebP: camera, lens, shutter, GPS,
63 tags. PNG: nothing. Chunk order of the ImageMagick PNG:

    IHDR iCCP cHRM bKGD pHYs tIME zTXt(app10) zTXt(xmp) IDAT...
    eXIf(10963) tEXt(exif:...) x60 IEND

eXIf after the pixel data. image's PngDecoder answers exif_metadata()
from the chunks before the first IDAT, and from_decoder consumes it,
so it cannot be asked again. Plan 009 called this case rare; for
ImageMagick output it is the norm. c366244: when a PNG gave no EXIF up
front, read the last 128 KB after the decode (the file is in the OS
cache by then) and search backwards for "eXIf". Chunks can only be
walked forwards and compressed pixel data can contain the same four
letters, so a candidate counts only if the chunks after it, walked by
their length fields, arrive at an empty IEND. No CRC check. Cost on a
PNG without EXIF: 79 µs median, 131 µs p95 (10 MB 4K PNG, 200 calls),
about 0.1% of a 4K decode. Misses an eXIf block larger than the window
or one followed by more than 128 KB of other chunks.

Also in c366244: record_exists_when_the_pixels_fail, 18 bytes of text
named broken.jpg: image is Err, file_size and modified are set, exif
is None.

README shortcuts table has an I row (62f349e).

Benchmark of the final code c366244, skate right pass only, 3 runs,
panel off then on (All EXIF open), load average about 8 again:

| Folder | panel off | panel on |
|---|---|---|
| small_images: img/s, frame p99, CPU s | 145.3, 10.4 ms, 36.77 | 145.3, 10.0 ms, 37.12 |
| 4k_PNG_10MB: img/s, decode p50 | 57.4 (56.6 to 58.8), 79.1 ms | 61.8 (59.3 to 64.8), 76.4 ms |
| nikon_d500: img/s, frame p99, CPU s | 24.0, 45.8 ms, 9.03 | 23.9, 48.0 ms, 8.93 |

The dense panel with selectable rows costs nothing measurable, the
Nikon folder included, where it draws 63 tag rows per image.

The 4K row is the same binary twice, minutes apart: 57.4 and 61.8
img/s, 79.1 and 76.4 ms. That spread is wider than the gap between
main and the branch in section 7 (61 to 62 against 58 to 60), and the
panel-on run is the faster one. So the 4K trend in section 7 is within
what one binary does on this machine under this load. An idle rerun
would still be the clean statement; it is no longer needed to decide
anything. Summaries: benchmarks/20260918_224805 (off) and
20260918_224950 (on).
