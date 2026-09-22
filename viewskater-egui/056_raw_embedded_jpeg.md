# Camera RAW through the embedded JPEG: the walker, the EXIF block, a navigation bug

Date: 2026-09-20
Context: branch feat/raw-embedded-jpeg, nine commits on top of main
060fb12, none pushed. The research and the plan are in
docs/plans/011_raw_support.md. This is PR 1 of that plan. Numbers from
gota-home (Linux, RTX 3090, 144 Hz), files in the OS file cache.

| Commit | What |
|---|---|
| 22df0c1 | Show camera RAW files through their embedded JPEG |
| 2d3c0ba | Read the EXIF of camera RAW files for the metadata panel |
| 0cb91f0 | Let keyboard navigation pass a file that fails to decode |
| 27b9b42 | Move the TIFF walk into raw/tiff.rs |
| 7118d5e | Pick the embedded JPEG by its short side |
| 39551c0 | Show Canon CR3 and Fujifilm RAF files |
| 664a248 | Open KDC, NRW, PEF, RWL, SR2, SRF and SRW files |
| d44f0a4 | List the camera RAW formats for the file managers and in the README |
| 457ba0a | Show Olympus and OM System ORF files |

Sections 1 to 9 were written after the third commit. Their line
numbers are as of 0cb91f0, before raw.rs was split. Sections 10 to 14
cover the rest, with line numbers as of 457ba0a.

Decided with the owner today: PR 1 ships in every build, the rawler
decode (PR 2) is decided after PR 1, and only PR 1 goes into 0.4.0.
The message for a RAW file without a JPEG is "No embedded preview in
this RAW file".

Not tested in the app by me. The owner ran it on raw_all after the
third commit, found the navigation bug in section 6, and confirmed
after the fix that the folder can be browsed. 106 unit tests pass, 5
ignored, clippy clean with --all-targets.

## 1. What a RAW file gives us

A camera renders a JPEG when the shot is taken and writes it into the
RAW file next to the sensor data. The app finds that JPEG, reads those
bytes and decodes them with the JPEG decoder it already has. The
sensor data is never read and there is no new dependency.

ARW, CR2, DNG, NEF and RW2 are TIFF files: a chain of image file
directories (IFDs), each a list of 12-byte entries, and some entries
point to child IFDs. The JPEG is in a different place in each format.
Found with a throwaway Python walker on the owner's 16 files, then
confirmed by the Rust code:

| Format | File | Where the JPEG is | Size |
|---|---|---|---|
| ARW | DSC00116.ARW (a7 III) | IFD0, tags 0x0201 and 0x0202 (offset, length) | 1616x1080 |
| NEF | sample1.nef (D3) | child IFD 0 of IFD0, same tags | 2832x4256 |
| CR2 | cr2-sample-file.cr2 (6D) | IFD0, one strip with Compression 6 | 5472x3648 |
| RW2 | four Panasonic files | IFD0, Panasonic tag 0x002E, header magic 0x55 in place of 42 | 1920 wide |
| DNG | Pentax K10D, K-1 | child IFD with Compression 7, one strip | full size |

Seven of the 16 have no JPEG at all: two Blackmagic cinema DNGs, two
DNGs from CHDK firmware (only a 128x96 uncompressed thumbnail), a
GoPro GPR, and two PowerShot ".CR2" files that are bare sensor dumps
with no header. These are what PR 2 would cover.

## 2. src/raw.rs

One rule for all five formats: walk every IFD, collect every place
that may hold a JPEG, keep the ones whose frame header is a lossy
JPEG, pick one.

- `EXTENSIONS` (27): arw, cr2, dng, nef, rw2. Only formats checked on
  real files are in the list.
- `Source` (159) reads at absolute offsets through a `BufReader` and
  tracks the position itself, so a read close to the last one uses
  `seek_relative` and stays inside the buffer. `contains` checks
  offset plus length against the file length before every read.
- `read_ifd` (244) returns the entries of one IFD plus the four bytes
  with the next IFD's offset, or `None` when the count is 0, over
  `MAX_ENTRIES` (1024) or runs past the end of the file.
- `walk` (266) accepts "II" or "MM" with magic 42 or 0x55, then visits
  IFDs from a stack: the next IFD in the chain and the offsets in the
  SubIFDs tag 0x014A (`sub_ifds`, 393, one offset sits in the entry,
  more are an array elsewhere). A `seen` list stops loops and
  `MAX_IFDS` (64) bounds the work. Candidates per IFD:
  - the 0x0201 and 0x0202 pair,
  - a single strip (0x0111 and 0x0117 with count 1) when Compression
    is 6 or 7 and Photometric is not 32803 (color filter array, which
    marks sensor data),
  - tag 0x002E, only when the magic is 0x55.
  The orientation is tag 0x0112 of the first IFD visited.
- `lossy_jpeg_size` (436) walks the JPEG's segments to the first frame
  header. 0xC0 to 0xC2 give width and height. 0xC3 and the other frame
  markers give `None`. This is what keeps a CR2's sensor data out: it
  is a strip with Compression 6 too, a 20 MB lossless JPEG, and it has
  no Photometric tag to filter on.
- `pick_for_display` (414): the smallest JPEG with at least
  `MIN_LONG_SIDE` (1600) pixels on its long side, or the one with the
  most pixels when none is that large. Candidates are tried from the
  fewest bytes up, so with a mid-size JPEG present the header of the
  full-size one is never read.
- `read` (91) returns `RawContents { jpeg, orientation, exif }`.

The file is untrusted input. Nothing indexes with a value from the
file without a bounds check, entry counts and IFD counts are capped,
and the `broken_files_give_nothing` test covers an empty file, a
two-byte file, text, an IFD outside the file, an IFD that names itself
as next and as child, an entry count past the end, and JPEG tags that
point outside the file or at bytes that are not a JPEG.

## 3. Wiring

- src/file_io.rs: `is_supported_image` (29) also accepts
  `raw::EXTENSIONS`. `decode_into` (214) branches to `decode_raw_into`
  (261) by extension before `with_guessed_format`. The reason: a NEF
  or a DNG is a valid TIFF, and the TIFF decoder would decode IFD0,
  which is a 160x120 thumbnail. `decode_checked` (252) is the
  allocation check that was inline in `decode_into`, now shared by
  both paths. `supported_extensions` (34) feeds the file dialog filter
  in src/app/handlers.rs 69, which had its own copy of the list.
- `record.format` is the extension in upper case, like JXL.
- src/metadata.rs 34: `MetadataRecord::no_embedded_preview`. The
  record already travels to the pane when the pixels fail, so the pane
  reads the flag there (src/pane.rs 530) and picks the message.
- The slider preview worker and both cache decode paths call
  `load_image`, so they needed no change.

## 4. The EXIF block (2d3c0ba)

kamadak's `read_from_container` reads the whole file for anything
TIFF-based (plan 009 section 7), so the app reads the bytes itself.
Plan 009 had a stitched TIFF with rewritten offsets in mind. What was
built is simpler: offsets in a TIFF file count from the start of the
file, so a prefix of the RAW file is a TIFF block that
`metadata::parse_exif` reads as it is.

`exif_block` (355) visits IFD0 and the Exif, GPS and interoperability
IFDs it leads to, takes the furthest end of any IFD or out-of-line
value, and reads the file from 0 to there in one read. `parse_exif`
already runs kamadak with `continue_on_error`, so a value that is cut
off is skipped and the rest is kept.

Two limits:

- `MAX_EXIF_VALUE` (128 KB): a larger value does not extend the block.
  The first version read 4 MB for the G9M2's RW2 and 430 to 565 KB for
  the FX150's, because an RW2 has its JPEG (tag 0x002E, 739 KB) and
  sensor data (tag 0x0127, 5 MB) as tag values in IFD0. With the limit
  the blocks are 4288 and 864 bytes. The largest MakerNote seen so far
  is 68 KB (Canon 6D).
- `MAX_EXIF_PREFIX` (4 MB): the most that is ever read.

An RW2 block gets magic 42 written over 0x55 so kamadak accepts it.

Block sizes on the owner's files: ARW 42.8 KB, NEF 33.4 KB (Df:
133.5 KB), CR2 80.0 KB, K10D DNG 79.7 KB, K-1 DNG 162.0 KB. The
Pentax DNGs are the case a fixed "first N KB" would get wrong: a
57 KB thumbnail strip and the 78 to 160 KB DNGPrivateData sit between
IFD0 and the Exif IFD.

Checked with the ignored test `real_photos_have_camera_fields` on six
files: camera, focal length, aperture, shutter, ISO and date came out
for all six, the lens for the ARW and the CR2 only, and the CR2 has
its GPS position. 54 to 92 tags per file. Nothing was compared against
exiftool, which is not installed here.

Reading more than this was the owner's preference ("I'd just read as
much as I can if the cost is not a concern"). The block takes
everything the three IFDs point to, MakerNote included, and leaves out
only image data.

## 5. Orientation, verified on portrait shots

Open question from plan 011: is the embedded JPEG stored unrotated,
and is IFD0's tag the right turn? All 16 of the owner's files and the
first 73 sample dumps had orientation 1. The scan in section 8 found
69 turned files. Four were extracted by offset and length, turned with
ImageMagick by the IFD0 tag (6: -rotate 90, 8: -rotate 270), and
looked at before and after:

| File | IFD0 tag | Stored JPEG | After the turn |
|---|---|---|---|
| Nikon Df NEF | 6 | 1620x1080, a flower pot on its side | upright |
| Canon Rebel XT CR2 | 8 | 1536x1024, grass running diagonally | sky up, grass down |
| Panasonic TZ60 RW2 | 6 | 1920x1280, a balcony on its side | upright |
| Sony A450 ARW | 6 | 1616x1080 | flowers from above, cannot tell |

`walk` reports Rotate90, Rotate270, Rotate90, Rotate90 for them, the
same turns. The embedded JPEG of an ARW, CR2, DNG or NEF has no EXIF
of its own. The RW2's has, with the same value as IFD0. The JPEG
decoder is never asked for it, so the turn happens once.

sample1.nef is the odd one: a portrait JPEG stored upright (2832x4256)
with tag 1. Tag 1 means no turn, so it shows correctly.

Four more turned ARW files are in raw_all now (HX95, A58, 6400A, A99,
all tag 8). Not looked at yet. No turned CR3 exists on the sample
site.

## 6. Keyboard navigation stopped at a failed file (0cb91f0)

Found by the owner browsing raw_all: the arrow keys stopped in front
of the first file without a JPEG and the rest of the folder could not
be reached. Not a RAW bug. Any file that fails to decode did this on
main, a broken JPEG included. RAW files without a JPEG made it common.

Cause: `App::step_navigation` (src/app/handlers.rs) moves only when
`Pane::is_next_cached` is true, which asked
`cache.loaded_for(next).is_some()`. The slots of `SlidingWindowCache`
were `Option<Loaded>`, and `poll` dropped a result without pixels, so
a failed decode left `None` in its slot, the same as a decode that has
not finished. The key waited on it for good. The mouse wheel uses the
same check. The slider, Home, End and opening a file fall back to
`load_sync`, which handles a failure, so they worked.

Fix: the slots hold `Option<Decoded>` (src/cache.rs 225), where
`Decoded` is `Image(Loaded)` or `Failed(Arc<MetadataRecord>)`.

- `poll` writes `Failed` into the slot when a result has no pixels
  (449). It needs no upload, so it does not go through
  `pending_uploads` and `remove_index` has nothing new to reindex.
- `initialize` writes `Failed` for the centre image when its sync
  decode fails.
- `decoded_for` (672) is new. `loaded_for` (665) stays and returns
  images only, so the slider paths are unchanged.
- `Pane::is_next_cached` (312) and `Pane::navigate` use `decoded_for`.
  `show_decoded` (629) puts an image or a failure on screen.
- `jump_to` and the move after a trashed file use it too. Before, they
  decoded a failed file a second time through `load_sync`.
- The cache overlay has `COL_FAILED` and a legend entry.
- A RAW file without a JPEG is logged once at debug level by
  `decode_raw_into`. The four call sites that log a failed load skip
  it when `no_embedded_preview` is set. The owner's terminal had a
  WARN or ERROR line for each of these files on every decode.

Test: `keyboard_navigation_passes_a_file_that_fails_to_decode`
(src/pane/tests.rs 287) puts a text file named b.jpg between two PNGs
and navigates right twice and left twice through the pane, checking
the index, the texture and the record at each step. The cache tests
that fill slots by hand were adapted to the new type.

This commit does not depend on the RAW work and could be its own PR.

## 7. Measurements

12 RAW files that carry a mid-size and a full-size JPEG (four Nikon
Df, two Z 30, Sony RX1R III, a1 II, a7 V, a9 III, FX2, ZV-E10 II),
hard-linked ten times into a folder of 120. Small JPEGs are 1616x1080
or 1620x1080 (83 to 697 KB), large ones 3504x2336 to 6192x4128
(655 KB to 2.4 MB). The same JPEGs were also extracted into plain
.jpg folders. `--bench-nav --bench-runs 3`. The rule was switched with
a temporary environment variable in `pick_for_display`, removed
afterwards.

| Folder | images/s | frames stuck waiting for a decode | decode median | peak RSS |
|---|---|---|---|---|
| RAW, smallest JPEG with 1600 px | 142 right, 140 left | 3 to 5% | 18.6 to 19.4 ms | 580 MB |
| RAW, largest JPEG | 24.7, 24.2 | 78 to 79% | 164 to 167 ms | 1.5 GB |
| small JPEGs as .jpg | 146, 136 | 2 to 9% | 16.4 to 18.3 ms | 520 MB |
| large JPEGs as .jpg | 28.2, 31.1 | 70 to 72% | 104 to 119 ms | 1.6 GB |

- With the rule, skate mode runs at the display's frame rate. With the
  largest JPEG it is 24 images/s. The owner had asked for the IFD0
  JPEG for this reason before any number existed.
- Reading the JPEG out of the RAW costs about 2 ms over the same JPEG
  as a file, walk, EXIF block and turn included.
- The two large rows are not like for like. Six of the 12 files are
  portrait shots, the RAW path turns 16 to 20 MP of pixels, and the
  extracted JPEGs have no tag.

Slider, one run: UI blocked per load 14.9 ms against 14.4 ms, click to
image 41 ms against 36 ms, preview thumbnails per second while
hovering 14.1 against 14.9.

The slider preview needs no work. Thumbnails are 400 px
(src/decode.rs 11), the 160x120 JPEG is too small for them, and the
mid-size JPEG the rule already picks is the right source.

Open: loading the full-size JPEG after the user stops on an image, to
judge focus at 100%. The numbers rule it out for navigation and say
nothing about whether it is worth having. PR 2 would need the same
mechanism.

## 8. Test data

raw.pixls.us has an index of 2016 files, 1870 of them CC0, each with
a sha256 and an exiv2 text dump. No rsync, so HTTP on one connection.

- Orientation scan: the first 8 KB of each dump (the orientation line
  is about 330 bytes in). The first attempt fetched whole dumps and
  stalled on Panasonic's, which are 1.5 MB each.
- Result: 69 of 1744 files have an orientation other than 1. ARW 6,
  NEF 10, RAF 7, CR2 2, RW2 2, DNG 21, GPR 13, CRW 2, and one each of
  SRW, DCR, ORF, TIF, FFF, RWL. No CR3.
- Picked 255 files: every turned file (at most six per extension),
  then per extension recent bodies, old bodies and compression modes
  not covered yet. 252 downloaded, 6.2 GB, every checksum matched. One
  Kodak DCR has a "/" in its name and failed. Two were already there.
- Location: /home/gota/ggando/rust_gui/data/test_data/raw/raw_all, now
  269 files, 6.5 GB. Names have "_" where the site has ":", like the
  owner's earlier files. The manifest with license, checksum, source
  URL and the reason each file was picked is
  raw/raw_pixls_us_manifest.tsv.
- Benchmark folders: raw/bench/raw_two_sizes (hard links), jpeg_small
  and jpeg_large (about 20 MB of extracted JPEGs).

Sony's two JPEG sizes, from the dumps: bodies from the a7S III and a1
generation on (a7 IV, a7R V, a6700, a7C II, a9 III, a1 II, a7 V, FX2,
RX1R III) have a third IFD with a JPEG of 3504x2336 up to 8640x5760.
The a7 III, a7R IV, a9 II, a7C and ZV-E10 have only 1616x1080. Nikon's
Df and Z 30 have 1620x1080 next to the full-size one. Panasonic has
1920 wide only, the 2023 G9 II included. The 2005 Rebel XT has
1536x1024, under the threshold, and is shown as the largest.

## 9. Next, as it stood after the third commit

Items 1, 2, 3, 4, 5 and 6 are done, see sections 10 to 14. Item 7 is
open.

1. CR3. ISO BMFF boxes. From Laurent Clévy's notes, not checked on a
   file yet: a 160x120 JPEG in THMB, a 1620x1080 JPEG in PRVW, the
   full-size JPEG as the first track in mdat, and the EXIF as four
   separate TIFF blocks CMT1 to CMT4. Those four have their own
   offsets each, so the prefix trick of section 4 does not apply and
   CR3 needs a stitched block. 24 CR3 files are in raw_all.
2. RAF. 26 files, 7 of them turned.
3. The formats the walk may already cover, to check on the downloaded
   files before adding the extension: PEF, NRW, SRW, SR2, SRF, 3FR,
   FFF, IIQ, ERF, MEF, MOS, KDC, DCR, RWL, GPR.
4. A coverage number: how many of the 269 files show a JPEG.
5. The four turned ARW files of section 5.
6. README, Info.plist, .desktop MimeType.
7. Plan 011 sections 5 and 9 still have the Sony size and the
   orientation as unverified and the build question as open.

## 10. The split and the rule (27b9b42, 7118d5e)

src/raw.rs (267 lines) keeps what every format shares: `EXTENSIONS`
(30), `RawContents` (50), `read` (70), `contents` (74), `Found` (105),
`Source` (125), `pick_for_display` (159) and `lossy_jpeg_size` (181).
There is one finder per container, each `find(source) ->
io::Result<Option<Found>>` with `None` for "not my container":
src/raw/tiff.rs, src/raw/cr3.rs, src/raw/raf.rs. `contents` tries them
in that order. `Found` carries the EXIF block in place of the TIFF
header, so `contents` does not know which format it read. `Found` was
renamed `JpegsAndExif` on 2026-09-22 after review. The check
that a candidate lies inside the file moved from the TIFF walk into
`contents` and covers every finder.

The rule changed from "1600 on the long side" to `MIN_SHORT_SIDE`
(43) = 1000. Cameras store a JPEG of about 1080 lines for their own
screen. A Canon with a 4:3 sensor makes it 1440x1080 (PowerShot SX70
HS), which the long side rule passed over for the full-size JPEG. What
passes now: 1616x1080 (Sony), 1620x1080 (Nikon, Canon), 1440x1080,
1920 wide (Panasonic, Leica), 1536x1024 (Rebel XT, which was shown as
the largest before and still is).

## 11. CR3 (39551c0)

Checked on 24 files with a Python box dumper before writing the code.
23 have the same layout, which matches Laurent Clévy's notes:

- `ftyp` with brand "crx ".
- `moov` / `uuid` 85c0b687-820f-11e0-8111-f4ce462b6a48: `CMT1` to
  `CMT4`, each a TIFF block of its own ("II*\0", IFD at 8), and
  `THMB`, a 160x120 JPEG 16 bytes into the box body.
- `moov` / `trak` four or five times. The first has the full-size
  JPEG (5088x3392 to 6960x4640), the others the sensor data and a
  metadata track. All have codec `CRAW`, so the codec does not tell
  them apart and the frame header check does.
- `uuid` eaf42b5e-1c98-4b88-b9fb-b7dc406e4d16 after `moov`: 8 bytes,
  then `PRVW`, a 1620x1080 JPEG (1440x1080 on the SX70 HS) 16 bytes
  into the box body.

The 24th, Canon EOS R8, is the HDR PQ case Grok mentioned. `THMB` and
`PRVW` have version 1 and hold HEVC ("hvcC" 40 bytes in), and the
first track is HEVC too. The app shows "No embedded preview in this
RAW file" for it, and the panel still has its EXIF.

src/raw/cr3.rs:

- `boxes` (113) lists the boxes between two offsets: 32-bit size,
  64-bit size when the size field is 1, to the end of the parent when
  it is 0. It stops at the first box that does not fit and at
  `MAX_BOXES` (64).
- `find` (55) collects `THMB` and `PRVW` through `image_in` (154) and
  the first sample of every track through `first_sample` (162), which
  reads `stsz` and `co64` or `stco` under `mdia` / `minf` / `stbl`.
- The fields before the image differ between `THMB` and `PRVW` and
  between their versions, so `image_in` takes the image to run from
  byte 16 to the end of the box. The JPEG decoder stops at the end
  marker.
- The rule then shows `PRVW`. The headers of the track samples are
  never read, because candidates are tried from the fewest bytes up.

The EXIF does not work with the prefix trick of section 4: the four
blocks have their own headers and their offsets count from their own
start. `join_exif` (207) keeps `CMT1` as it is, appends `CMT2` and
`CMT4`, adds each block's new position to the offsets in its IFD
(`rebase_ifd`, 255, which also follows the interoperability IFD
pointer once), and writes a new IFD0 at the end: the old entries plus
pointers 0x8769 and 0x8825, sorted by tag. The header points to the
new IFD0 and the old one stays in the buffer unused. `CMT3`, the maker
note, is left out. The joined block is 3.4 to 4.3 KB in the real
files. The orientation is tag 0x0112 in `CMT1` (`orientation_of`,
192).

Orientation on a real file: the scan of section 8 found no turned
CR3, but the downloaded "Canon - EOS R10 - 3_2.CR3" has tag 8. Its
`PRVW` is stored sideways (a man in a cap, lying on his side) and is
upright after -rotate 270. `load_image` applies Rotate270 and the
record has camera, lens (EF50mm f/1.8 STM), shutter, ISO and the date
with its UTC offset, 53 tags.

## 12. RAF (39551c0)

All 26 files: magic "FUJIFILMCCD-RAW ", and at bytes 84 and 88 the
big-endian offset (148 in every file) and length of one JPEG. It is
1920x1280 or 2048x1536 from older bodies and 4416x2944 or 4000x3000
from newer ones, 0.7 to 5.6 MB. There is no smaller one apart from the
thumbnail inside the JPEG's own EXIF, so a RAF navigates like a camera
JPEG of that size.

That JPEG is a complete camera JPEG with its own EXIF block. So
`Found` and `RawContents` have `exif_in_jpeg`, and `decode_raw_into`
(src/file_io.rs 262) takes the EXIF and the orientation from the JPEG
decoder for it. The seven turned RAF files have tag 6 or 8 in the
JPEG's EXIF and a landscape JPEG. The X100VI one gives Rotate270 and
69 tags through `load_image`. Not looked at as a picture.

## 13. The TIFF walk covers more, and ORF (664a248, 457ba0a)

`raw::read` was run over all 268 RAW files in raw_all whatever their
extension (ignored test `real_raw_files_one_line_each` in src/raw.rs).
The walk already found a JPEG in every PEF (8), SRW (6), RWL (5), NRW
(4), KDC (2), SR2 (1) and SRF (1) file, so those extensions were
added. The SR2 and the SRF only have a 640 pixel wide JPEG.

ORF needed code. Checked on 13 ORF and 2 ORI files from 14 bodies:

- The header has "RO" or "RS" where 42 would be (`ORF_MAGICS`).
- The JPEGs are in the maker note: IFD0 -> Exif IFD -> tag 0x927C.
  The note is a header and an IFD. Tag 0x0100 is a 160x120 JPEG. Tag
  0x2020 (type IFD) points to the camera settings IFD, whose tags
  0x0101 and 0x0102 are the offset and length of a 1600x1200 (E-450)
  or 3200x2400 JPEG.
- Header "OLYMPUS\0" plus 4 bytes, or "OM SYSTEM\0\0\0" plus 4 bytes on
  the OM-5, TG-7, OM-1 II, OM-3 and OM-5 II: offsets count from the
  start of the maker note. Header "OLYMP\0" plus 2 bytes on the
  C7070WZ (2005) and SP550UZ (2007): offsets count from the start of
  the file, and only the small JPEG exists.

`olympus_jpegs` (src/raw/tiff.rs 360) does this, called from `find`
(186) when the magic is an ORF one. `exif_block` writes 42 over any
magic that is not 42, which covers RW2 and ORF. The turned one, XZ-10
with tag 6: a hot-air balloon, sideways as stored and upright after
-rotate 90. Records: XZ-10 52 tags, OM-3 62 tags with the lens.

Four more turned ARW files were looked at (A58, 6400A, A99, HX95, all
tag 8): sideways as stored, upright after the turn. With the A450 of
section 5 that settles Sony.

## 14. Coverage and the format lists (d44f0a4)

Of the 268 RAW files in raw_all, 229 have one of the 16 extensions the
app opens: arw, cr2, cr3, dng, kdc, nef, nrw, orf, ori, pef, raf, rw2,
rwl, sr2, srf, srw. 210 of them show a JPEG. The 19 that do not:

- 14 DNG: three phones (Samsung SM-G950U, OnePlus One, LG H850), two
  Sigma fp, two Leica M9, five from CHDK firmware, two Blackmagic.
- 4 CR2 that are CHDK sensor dumps with no header.
- 1 CR3 with HDR PQ.

Five DNGs show a JPEG under 1000 pixels on the short side: DJI FC4382
960x720, DJI FC9287 960x540, Pixel 7 Pro 1280x964, Ricoh GXR 640x480,
an Adobe-converted 7D Mark II 1024x683. These 24 files are the case
for PR 2.

The other 39 files have 12 extensions the app does not open, because
the walk finds nothing in them: crw 6, gpr 7, x3f 4, 3fr 3, fff 3, raw
3, mrw 3, iiq 3, erf 2, dcr 2, mos 2, tif 1. Old or rare bodies. Not
looked into.

Lists outside the code: resources/macos/Info.plist has a "Camera RAW
Image" document type with the extensions and the
public.camera-raw-image content type (parses with plistlib, not tested
on a Mac). assets/viewskater-egui.desktop and cargo-appimage.desktop
have the MIME types that /usr/share/mime/globs has for these
extensions, which is all but rwl and srw (both validate with
desktop-file-validate). cargo-appimage.desktop still lacks image/jxl,
as noted in an earlier handoff. README.md has one new line: "Opens
camera RAW files (arw, cr2, cr3, dng, nef, raf, rw2 and more) by
showing the JPEG the camera stored inside them". The owner has not
seen that wording.

Open:

1. The owner has not tested CR3, RAF, ORF or the panel for RAW in the
   app.
2. Plan 011 sections 5 and 9 still have the Sony size and the
   orientation as unverified and the build question as open.
3. Loading the full-size JPEG after the user stops on an image
   (section 7).
4. The PR draft.
5. A RAW and its JPEG shot together are two entries in the list.
   Pairing them was mentioned as a possible official build feature and
   not decided.

