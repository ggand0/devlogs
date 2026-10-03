# Stars dropped from 0.4.0: demand, storage, and the review of feat/stars

Date: 2026-10-03
Context: branch feat/stars at f927577, five commits on top of main
7a3c4e5, pushed to origin, no PR. A review of the finished branch
before release: is there demand, and is the storage right. Design in
docs/plans/012_star_bookmarks.md, earlier research in devlog 057, test
plan in tmp/test_plans/2026-09-25_stars.md. Checked on 2026-10-03
against the code, the GitHub API and the pages linked at the end. A
line says so where a claim could not be checked.

## 1. Decisions

Made by the owner on 2026-10-03.

1. Stars are not in 0.4.0. They may ship in a later release.
   feat/stars stays as it is, unmerged.
2. The stars move out of the image folders. One yaml in the user
   folder (the `dirs` crate) replaces the hidden `.viewskater.yaml` in
   every folder with a star. Not built.
3. .xmp files stay a setting that is only built if someone asks.
4. It stays one star. No grades, no colour labels, no reject mark.

Why stars are not in 0.4.0:

- Nobody asked for them (section 3).
- The feature came out of the 0.4.0 planning with Opus 5.5. Plan 008
  has it as candidate 5 and calls it "The bet, not the request". The
  owner did not want it strongly and stayed away from the project for
  about two weeks because of it.
- A feature cannot be removed once people use it, and the storage it
  would ship with was changed on the same day (decision 2). Shipping
  it in 0.4.0 would need the storage rewrite and the Windows and macOS
  rows of the test plan first. None of those rows has been run.
- Nothing on main waits for it. Main has Move to Trash (#43), the
  metadata panel (#45), EXIF orientation (#46) and RAW through the
  embedded JPEG (#47) since 0.3.0.

## 2. Storage: one yaml in the user folder

The owner, on the Picasa complaint in section 5: "Actually I'd agree
with them. One yaml in every dir is pretty bad. We should keep a
single yaml in the user folder fetched by the dirs crate."

What decides it: a file in the config folder belongs to the app, so a
later version can convert it or drop it. Files left in people's image
folders have to be read by every later version.

The cost: a star is tied to the folder's path.

- Moving or renaming a folder outside the app loses its stars.
- The same NAS folder opened from two computers has separate stars.

The iced version's selection feature already works this way.
`SelectionManager` (data-viewer src/selection_manager.rs 127 to 138)
keeps one JSON per folder in
`dirs::data_local_dir()/viewskater/selections/`, named by a hash of
the folder path, with the marks keyed by file name. It is behind the
`selection` cargo feature: S selects, X excludes, U clears, Ctrl+E
exports the JSON (README 96, "for dataset curation").

Parts of feat/stars that exist only because the files are in the image
folders (src/stars.rs module doc, lines 15 to 18):

- `star_folders.yaml`, the list of folders that have a star file.
- The Danger Zone in Preferences > Stars that moves every star file to
  the trash.
- `create_hidden` with `FILE_ATTRIBUTE_HIDDEN` on Windows.
- The star file listing in `enumerate_images` (src/file_io.rs).

With one file in the user folder, stars also work in a read-only
folder (row G1 of the test plan, and the NTFS disk mounted read-only
on 2026-09-25).

## 3. Demand

The owner's repos: no request. `gh search issues` on
ggand0/viewskater-egui and ggand0/viewskater for rating, star, cull,
flag, favorite and bookmark, any state, run on 2026-10-03. The two
matches are not requests: #16 (the slider preview proposal) and iced
#91 (a build log with "bitflags").

Other viewers, read through the GitHub API on 2026-10-03:

| Issue | Opened | State | Thumbs-up | Comments |
|---|---|---|---|---|
| ImageGlass #141 "Add Star, or Flag, bookmark" | 2016-10-27 | open | 12 | 10 |
| ImageGlass #411 "XMP Ratings" | 2018-08-29 | closed | 0 | 1 |
| ImageGlass #1002 "Show and change Exif.Image.Rating" | 2021-03-18 | closed | 0 | 1 |
| ImageGlass #2080 "Ability to mark images and skip them on a Slideshow" | 2025-01-30 | open | 0 | 0 |
| ImageGlass #2313 "Add a bookmarking feature" | 2026-04-15 | closed as duplicate | 0 | 1 |
| nomacs #138 "Filter by star rating" | 2017-07-26 | open | 6 | 2 |
| qView #842 "Add a bookmarking feature" | 2026-04-15 | open | 0 | 0 |

ImageGlass has 14,531 GitHub stars and nomacs 3,204. Comments on
ImageGlass #141:

- PwrSrg, 2020 (4 thumbs-up): "I LOVE Image Glass, and this is the
  ONLY missing feature in my opinion! (Favorites)"
- lukaspechar, 2022 (2): "adding the culling feature would save more
  time and would allow me to avoid installing another tool."
- thepwnshop, 2023 (5): "Only to find out it doesn't have a simple
  favorite or rating system. This will be the most depressing
  uninstall I've ever done"
- UltraKeelan, 2025: a hotkey for a quick rating, like L in the
  Windows image viewer.
- jacobtriffo, 2018, the first comment: an XMP 5-star system "instead
  of just a flag".
- drandarov-io, 2025: a setting for "the file itself or a separate
  xmp file".

The requester of #141 wanted to star photos, sort by rating in
Explorer and "delete all the photos that were not starred", and named
Picasa.

Read together:

- About one request a year across these viewers. Against the owner's
  own inbox (plan 008): EXIF 3 people, formats 3 people.
- Two kinds of request. A simple favorite key (PwrSrg, thepwnshop,
  UltraKeelan, qView #842), which is what the branch has. A standard
  rating that other apps read (#411, #1002, jacobtriffo,
  drandarov-io), which a yaml does not give.
- A request for a simple flag got a request for 5 stars as its first
  comment.
- qView and oculante have no rating. oculante's README (line 156) has
  "Bookmark directories, favorite and manage files", and
  `favourite_images` appears only in its src/settings.rs (devlog 057:
  declared, never read or written).

## 4. Where other apps keep the rating

| Where | Who | Cost |
|---|---|---|
| Inside the image file | nomacs, Photo Mechanic for JPEG | Rewrites the photo. nomacs #821 (open since 2021): "i now have 50ish raw files that i can only edit in darktable", rated ORF files no longer import into Lightroom or Capture One. nomacs #552: "the raw file is unreadable for lighrtoom". |
| One .xmp per photo | FastRawViewer, darktable, Photo Mechanic for RAW | Read by the RAW editors. Lightroom ignores an .xmp next to a JPEG (FastRawViewer manual, Adobe community thread). One visible file per marked photo. |
| App database | FastStone, XnView MP, Lightroom, digiKam, the iced version's selection feature | The mark is tied to the path. |
| File system attributes | Gwenview | Lost on a disk or share that does not keep them (Arch forum thread, not checked further). |
| One hidden file per folder | Picasa `.picasa.ini` (`star=yes`), gThumb `.comments/`, Bridge `.BridgeLabelsAndRatings` for folder labels | Other apps do not read it. Files stay behind in every folder. |

Every option has a cost. The serious tools keep their own record
and offer `xmp:Rating` as the exchange format (devlog 057 section 4).
Plan 012 section 8 has the same shape.

On .xmp: plan 012 section 8 writes an .xmp only for a starred photo
and only with the setting on, never for every photo. The owner is an
ML engineer and not a photographer, does not like how .xmp files work,
and keeps them as a toggle if someone asks. A yaml is also what a
Python script reads (`yaml.safe_load`). If the feature is ever
announced to photographers as culling (plan 007), the .xmp setting for
RAW files is the part they will miss, with or without a request.

## 5. The Picasa complaint

One thread, AnandTech forums, 2004-07-19, "Picasa.ini ARG!", two
people complaining:

- edro: "it puts a Picasa.ini file in EVERY directory that has images
  in it", "Stupid files... I hate extra files", "Is there a way to
  disable them?"
- Modeps: "I think Picassa could have put these files in their main
  directory instead and just point to different places on your hard
  drive."
- PlatinumGold: "you like WHAT it does, but not HOW it does it".

Modeps's suggestion is decision 2. No report of Picasa losing stars
was found. PhotoPrism #2211 (2022) asks to import `star=yes` from
`.picasa.ini`, so the files were still in people's folders six years
after Picasa ended.

On 2026-10-03 I wrote that feat/stars "handles the things Picasa was
disliked for" above five bullets. Only two of them answer this thread
(the file appears with the first star, and the Stars tab finds every
file). The other three were file-safety checks, listed in section 6.

## 6. Checks on src/stars.rs at f927577

- A save writes `.viewskater.yaml.tmp`, syncs it and renames it over
  the file (`write_through_temp`, 557).
- A save reads the file again and changes one entry, so a change from
  the other pane or another window stays (`save_change`, 537).
- A file that does not parse, or has a higher `version`, is never
  written over (`read_file`, 527, and `locked`).
- An entry whose file is gone or has another size shows no star and
  stays in the file (`read_folder`, 503).
- The star files leave only through the trash (`move_star_files`,
  311).
- `serde_yaml` 0.9 is archived by its author (prior knowledge, not
  re-checked). settings.yaml has the same dependency. The file is
  plain YAML.
- Two app windows starring in the same folder at the same moment use
  the same temp file name and no lock, so one change can be lost.
  Inside one window the writer thread saves in order.

What a person who never presses S sees on the branch:

- Main view: nothing. The footer keeps the star slot only when the
  pane's list has a star (src/menu.rs 493), the star is painted only
  on a starred image (660), and the slider marks need a star.
- Metadata panel: one gray outline star while the panel is open
  (src/menu.rs 683, src/metadata_panel.rs 221).
- Menus: Edit > Star, View > Starred Only, and the Stars tab in
  Preferences.

What cannot be taken back after a release: the file format, the four
bare keys S, F, Q and E, and the follow-up requests (grades, colour
labels, reject, .xmp). The icons can change in any release.

FastStone ships its ratings turned off until Rating > Enable File
Rating (devlog 057). The branch needs no such setting, because the
main view does not change without a star.

## 7. Grok

Prompt: tmp/grok/2026-10-03_star_bookmarks.md. Response:
tmp/grok/star_feature_response.md.

Its answer: "Release it." Demand "too small to justify it as a growth
bet". Add an export of the starred names first. Leave .xmp off:
"photographers who need Lightroom will not adopt this viewer for
culling either way" (no source given).

| Claim | Check |
|---|---|
| ImageGlass #141 has "no comment thread visible" | Wrong: 10 comments, 12 thumbs-up. |
| The iced version has S/X/U marks and a JSON export | True, README 96. |
| oculante's README lists favorites | True for the README, see section 3. |
| nomacs #909 "Ratings are not saved", 2022-12-07 | True. |
| nomacs #1320, #1399, #1482 | Titles match the issue search. |
| IrfanView forum threads 12889 and 3855 | Not checked, the forum returned 502. |
| Malwarebytes forum, 496 `.picasa.ini` files | Not found by a search. |
| nsxiv-extra #71, multi-marks patch | No pull request with that number. |

From its sources, if they hold:

- After marking, people copy or move the keepers to a folder
  (IrfanView F7 and F8, the 2016 ImageGlass request). Nobody in its
  sources opens the starred set in an editor.
- An IrfanView user wanted the X mark remembered after closing,
  "Not necessarily to alter the file ... but maybe to put the 'x' in
  some kind of central file."
- IrfanView's answer to a 1 to 5 request: "No. It is just a viewer, no
  a catalogue program."
- No viewer was called bloated for a mark like this, and none removed
  one after shipping it.
- nsxiv has marks, next and previous marked, and `-o` to print the
  marked names.

Not taken from it: the export before release. Nobody asked for it, and
a yaml in the user folder is a list a script can read.

## 8. If the branch is picked up again

- Storage rewrite to one yaml in the user folder (section 2), and the
  parts listed there removed.
- What the key is. The branch keys an entry by file name inside one
  folder's file, with the size as a check. One file needs the folder
  path too.
- Test plan rows not run: G1 (read-only folder), J1 to J3 (Windows),
  K1 to K3 (macOS). L1 (slider mark colour) is open. Several rows
  change or go with the storage.
- Plan 012 stays as written.

## Links

- ImageGlass: https://github.com/d2phap/ImageGlass/issues/141,
  https://github.com/d2phap/ImageGlass/issues/411,
  https://github.com/d2phap/ImageGlass/issues/1002,
  https://github.com/d2phap/ImageGlass/issues/2080,
  https://github.com/d2phap/ImageGlass/issues/2313
- nomacs: https://github.com/nomacs/nomacs/issues/138,
  https://github.com/nomacs/nomacs/issues/552,
  https://github.com/nomacs/nomacs/issues/821,
  https://github.com/nomacs/nomacs/issues/909
- qView: https://github.com/jurplel/qView/issues/842
- Picasa: https://forums.anandtech.com/threads/picasa-ini-arg.1364150/,
  https://gist.github.com/fbuchinger/1073823,
  https://github.com/photoprism/photoprism/issues/2211
- .xmp next to a JPEG:
  https://www.fastrawviewer.com/usermanual17/xmp-metadata,
  https://community.adobe.com/t5/lightroom-classic-discussions/importing-a-jpg-with-metadata-in-xmp-sidecar-from-fastrawviewer-lr-seems-to-ignore-the-metadata/m-p/14566910/highlight/true
- Bridge:
  https://community.adobe.com/questions-558/where-is-stored-the-folder-label-registry-location-172018
- gThumb and Gwenview: https://bbs.archlinux.org/viewtopic.php?id=238755
- IrfanView (from Grok, not opened):
  https://irfanview-forum.de/forum/program/feature-requests/12889-,
  https://irfanview-forum.de/forum/program/support/3855-
