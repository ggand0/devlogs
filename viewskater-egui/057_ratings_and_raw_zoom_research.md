# Ratings in other viewers, and zooming into RAW files

Date: 2026-09-23
Context: main at 7a3c4e5, after PR #47. Research for the 0.4.0 choice
between ratings, HEIC (#9) and configurable hotkeys (#8). Nothing
built. Checked on 2026-09-23 against official manuals, changelogs and
FAQs, and against the source code of the open-source viewers at their
upstream HEAD. Forum posts and a news article only where a line says
so. Links at the end.

## 1. Who asked for ratings

Nobody. No issue in ggand0/viewskater-egui or ggand0/viewskater asks
for ratings, flags or culling, open or closed (`gh issue list --state
all`, `gh search issues` for rating, star, cull and flag). Neither
email does: Andrew Law (2026-03-27) and Jan Vysloužil (2025-12-09).
Andrew wants to search his photos by lens, focal length, shutter
speed, GPS and date, browse the matches, and "organise the pictures".
That is a filter on EXIF fields, not a rating. The owner wants ratings
for their own use.

## 2. Ratings in twelve viewers

| App | Checked on | Stars | Other marks | Filter by rating | Keys for stars | Where the rating is saved |
|---|---|---|---|---|---|---|
| IrfanView | changelog up to 4.76 (2026-09-18) | no | none | | | |
| oculante | source 9bf06c3 | no | none | | | |
| qView | source c5eca1c | no | none | | | |
| ImageGlass | source e2db588 | shows a rating already in the file, cannot set one | none | sorts by it | | |
| nomacs | source d1c4f65 | yes | none | no, and no sort by rating | bare 0 to 5 | inside the image file |
| FastStone 8.5 | changelog, gHacks | yes since 7.6 (2022), off until Rating > Enable File Rating | tag (Q) | yes, Shift+1..5 | Alt+1..5, Alt+0 clears | its own database |
| XnView MP | FAQ, beginners guide, forum | yes | colour labels, categories | yes | Ctrl+0..5 | its catalog, .xmp export optional |
| FastRawViewer 2 | manual | yes | colour labels (Alt+6..9), reject (off by default) | yes | Alt+1..5, Alt+0 clears | .xmp sidecar |
| Photo Mechanic | Camera Bits docs | yes | tag (T), 8 colour classes (bare 0..8) | yes, View > Ratings | Ctrl+1..5 on macOS, Alt+1..5 on Windows, bare numbers as an option | .xmp sidecar for RAW, inside the file for JPEG |
| Lightroom Classic | Adobe help, Julieanne Kost's blog | yes | pick and reject (P, X, U), colour labels (6..9) | yes | bare 1 to 5 | its catalog, .xmp with a setting |
| darktable | manual | yes | reject (R), colour labels (F1..F5) | yes | bare 0 to 5 | .xmp sidecar, written at import |
| digiKam | manual | yes | pick labels (Alt+0..3), colour labels (Ctrl+Alt+0..9) | yes | Ctrl+0..5 | its database, plus the file or an .xmp sidecar by setting |

IrfanView: the only trace is a forum request for ratings from 2009
(irfanview-forum.de, thread 4332). oculante: settings.rs 160 declares
`favourite_images`, and nothing reads or writes it. The XnView MP keys
are from its forum: Ctrl+number confirmed by the developer in 2018
(thread 37472). FastStone's keys are from gHacks' article on 7.6.

## 3. What the culling tools have in common

FastRawViewer, Photo Mechanic, Lightroom Classic, darktable, digiKam
and XnView MP all have:

- 0 to 5 stars on the current image, one key press per value, no
  dialog.
- A filter that shows only the images with a given rating or more.
  FastStone has the same since 7.6.
- The rating saved in the XMP field xmp:Rating. The XnView MP FAQ:
  "star ratings (xmp:Rating), color labels and keywords are exchanged
  in both directions" with Lightroom and Bridge.
- Colour labels.

Most of them also have a yes or no mark next to the stars, and each
keeps it in a field of its own:

| App | Mark | Key | Saved as |
|---|---|---|---|
| Lightroom Classic | pick, reject | P, X, U clears | catalog, and XMP since 13.2 (February 2024) |
| Photo Mechanic | tag | T | XMP photomechanic:Tagged |
| FastStone | tag | Q | its database |
| digiKam | pick label: rejected, pending, accepted | Alt+1..3, Alt+0 clears | XMP, by setting |
| darktable | reject | R | not checked |
| FastRawViewer | reject, off by default | [R] button | xmp:Rating -1, Adobe Bridge's value for a reject |

How the rating is shown:

- FastRawViewer draws the rating and label above, below or over each
  thumbnail, set per view. A contrasting notice for about one second
  after a change is a setting the user turns on (Preferences > XMP >
  Ratings & Labels > Visual Feedback).
- darktable draws the stars over the thumbnails.
- Lightroom Classic moves to the next image after a rating, flag or
  label while Shift is held or Caps Lock is on.

## 4. Sidecar files

- Two naming schemes. IMG_0001.xmp is FastRawViewer's default,
  IMG_0001.CR3.xmp is darktable's. FastRawViewer has a setting for
  either one and for the search order.
- With the first scheme, a RAW and a JPEG shot together (IMG_0001.CR3
  and IMG_0001.JPG) share IMG_0001.xmp. FastRawViewer has a RAW+JPEG
  mode that also writes the XMP block into the JPEG.
- Lightroom and Bridge ignore .xmp sidecars of JPEG files
  (FastRawViewer manual, pages 54 and 56). They only see a JPEG's
  rating when the XMP is inside the JPEG. FastRawViewer can write it
  there. The option is off by default and warns that writing into
  image files can damage them, for example on a bad card reader. Photo
  Mechanic writes IPTC and XMP into JPEG, TIFF and PSD files.
- FastRawViewer keeps XMP for TIFF, PNG and HEIC off by default,
  because "The overwhelming majority of applications that work with
  graphic formats do not support XMP sidecar files for TIFF, PNG,
  HEIC/HEIF files."
- Photo Mechanic writes an .xmp sidecar for TIFF-based RAW files by
  default. "Allow RAW files to be modified" is off by default.
- Lightroom Classic writes .xmp only with Catalog Settings > Metadata
  > "Automatically write changes into XMP".
- darktable reads a sidecar when it imports the image. After that its
  database wins, and changes made by other programs are overwritten
  the next time it writes the file.
- XnView MP keeps ratings and labels in its catalog. Writing them to
  files is an option in Settings > Metadata (forum tutorial, 2022,
  thread 43651).
- nomacs writes the rating into the image file through exiv2: EXIF
  Rating and RatingPercent, xmp:Rating and MicrosoftPhoto:Rating
  (`DkMetaDataT::setRating`, DkMetaData.cpp 1276).
- FastStone keeps tags and ratings in its database. Its 7.6 changelog:
  "When copying or moving files, tags and ratings will be preserved in
  the database".

## 5. Zooming into RAW files

The app shows the smallest embedded JPEG with at least 1000 px on its
short side (`pick_for_display`, src/raw.rs 160). Zooming in magnifies
that JPEG. Nothing loads a larger image after the user stops on an
image or zooms.

The largest JPEG inside the file, per body. Files in raw_all were read
with tmp/raw_tools/probe_raw.py and probe_cr3.py. The dumps are the
exiv2 text files raw.pixls.us keeps next to each sample.

| Body | Largest embedded JPEG | Checked on |
|---|---|---|
| Sony a7 III | 1616x1080 | DSC00116.ARW |
| Sony a7R IV, a9 II | no IFD2, so no full-size JPEG | raw.pixls.us dumps |
| Sony a7 IV | 4608x3072 on an APS-C crop shot, 7008x4672 on a full-frame one | raw_all file, dump |
| Sony a7CR | 6240x4160 | raw_all file |
| Panasonic G9 II, GH7 | 1920x1440 | raw_all files |
| Panasonic S5 II | 1920x1280 | raw_all file |
| Canon CR3 | the size of the sensor data, as the first track's only sample | 23 of the 24 CR3 files in raw_all |

The 24th CR3 is the R8 shot with HDR PQ, which has HEVC where the
JPEGs would be. On every Sony body in the table the app shows the
1616x1080 JPEG.

On a body that stores a full-size JPEG, loading it after the user
stops on an image gives a sharp zoom without decoding the sensor data.
The rawler decode (PR 2 in plan 011) needs the same loading step to
put its result on screen. On top of that it covers the bodies without
a full-size JPEG, and the 19 files with no JPEG at all (devlog 056
section 14).

Andrew Law's camera files are CR3: "use the general GPS settings from
my Samsung phone pictures for 2026-04-11 for all JPG/CR3 images that
were taken the same day". Plan 011 section 3 has "He shoots Sony ARW",
which the email does not say. His question 4 asks about "Canon CR3/2
or Sony ARW".

## 6. Open

- What the feature looks like: one mark or 0 to 5 stars, the keys,
  where the mark is drawn, where it is saved, the filter.
- Plan 011 section 3, Andrew's camera.
- Loading the full-size JPEG after the user stops on an image, and
  whether rawler follows.

## Links

- IrfanView: https://www.irfanview.com/main_history.htm,
  https://www.irfanview.com/history_old.htm,
  https://irfanview-forum.de/forum/program/feature-requests/4332-
- XnView MP: https://www.xnview.com/en/faq/,
  https://www.xnview.com/download/XnViewMP%20for%20Beginners.pdf,
  https://newsgroup.xnview.com/viewtopic.php?t=37472,
  https://newsgroup.xnview.com/viewtopic.php?t=31175,
  https://newsgroup.xnview.com/viewtopic.php?t=43651
- FastStone: https://www.faststone.org/FSViewerDetail.htm,
  https://www.ghacks.net/2022/04/04/faststone-image-viewer-7-6-improved-performance-and-new-rating-system/
- FastRawViewer:
  https://updates.fastrawviewer.com/data/FastRawViewer2-Manual-ENG.pdf,
  https://www.fastrawviewer.com/usermanual17/xmp-metadata
- Photo Mechanic:
  https://docs.camerabits.com/support/solutions/articles/48001143067-star-ratings,
  https://docs.camerabits.com/support/solutions/articles/48001252564-color-class-ratings,
  https://camerabits.freshdesk.com/support/solutions/articles/48001252562-tagging-photos,
  https://docs.camerabits.com/support/solutions/articles/48001146198-iptc-xmp-preferences,
  https://exiftool.org/TagNames/PhotoMechanic.html
- Lightroom Classic:
  https://helpx.adobe.com/lightroom-classic/help/flag-label-rate-photos.html,
  https://jkost.com/blog/2024/06/applying-flags-stars-and-color-labels-in-lightroom-classic.html,
  https://community.adobe.com/t5/lightroom-classic-discussions/lrc-v13-2-and-flags-in-xmp/m-p/14442353
- darktable:
  https://docs.darktable.org/usermanual/development/en/lighttable/digital-asset-management/star-color/,
  https://docs.darktable.org/usermanual/development/en/overview/sidecar-files/sidecar/
- digiKam:
  https://docs.digikam.org/en/setup_application/shortcuts_settings.html,
  https://docs.digikam.org/en/setup_application/metadata_settings.html,
  https://docs.digikam.org/en/left_sidebar/labels_view.html
- Source: https://github.com/woelper/oculante,
  https://github.com/nomacs/nomacs, https://github.com/jurplel/qView,
  https://github.com/ImageGlass/ImageGlass
- raw.pixls.us dumps: a7 III
  https://raw.pixls.us/getfile.php/2414/exif/_DSC0009.ARW.exif.txt,
  a7R IV
  https://raw.pixls.us/getfile.php/3478/exif/DSC00395.ARW.exif.txt, a9
  II
  https://raw.pixls.us/getfile.php/3989/exif/SONY_A9II_(ILCE-9M2)_-_compressed_(3:2)_14bit.ARW.exif.txt,
  a7 IV
  https://raw.pixls.us/getfile.php/6928/exif/ILCE-7M4_DSC06673_FullFrame-Raw-Uncompressed.ARW.exif.txt
