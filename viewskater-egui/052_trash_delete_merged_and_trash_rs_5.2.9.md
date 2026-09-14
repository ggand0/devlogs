# Trash delete merged, and the fix that went upstream into trash-rs 5.2.9

Date: 2026-09-14
Context: closes the trash delete thread that runs through 049 (design
and Linux results), 050 (macOS results, SMB finding) and 051 (Windows
results, the crate bug). Records the upstream release, the merge of PR
#43, what the feature ended up being, and what is left.
Not verified claims are marked as such.

## The fix went upstream

Byron/trash-rs#150 "Fix deleting files on network paths on Windows",
opened by the owner on 2026-09-13 at 06:06Z, was merged the same day at
15:29Z (merge commit 29ad086, one comment from Byron: "Thanks a lot").
Byron rebased the three fork commits into two plus a "review" commit
of his own that only rewords the doc comment on
`to_shell_parsing_name`. trash v5.2.9 was tagged and published to
crates.io on 2026-09-13, so the fix was on crates.io about nine hours
after the PR was opened. Issue #55, open since 2022-08, closed with it.

Checked on 2026-09-14 by diffing the fork tip (fb9ee8a) against the
v5.2.9 tag: `src/windows.rs` is identical apart from that doc comment.
So the Windows runs in 051 (mapped drive, USB stick, local NTFS) were
done against the same code that is now in the release.

5.2.9 carries one more change, from someone else's PR (#151, "fall
back to the home trash when per-volume trash cannot be created"). On
Linux, when a mount's root is read-only so `.Trash-$uid` cannot be
created, the crate used to return PermissionDenied and leave the file.
Now it moves the file into the home trash instead, which for another
device means copy then delete inside the crate. The sshfs step 10 in
049 recorded the old behavior ("refused with the file untouched"); with
5.2.9 the first half of that step would trash the file into
~/.local/share/Trash instead. Not re-run. The app's own read-only
precheck (parent folder not writable) is unaffected.

The app moved from the fork to the release in commit 7e568d4 "Use
trash 5.2.9 from crates.io instead of the fork": `trash = "5.2.9"`,
the `[patch.crates-io]` entry removed. Cargo re-resolved one
transitive windows-core for iana-time-zone to 0.62.2. Tests and clippy
clean. The fork branch ggand0/trash-rs:fix-windows-unc-parsing-name is
no longer referenced by anything.

## PR #43 merged

"Add Move to Trash with the Delete key, Edit menu and footer button",
14 commits, merged into main on 2026-09-14 (local time; 21:46Z on the
13th) as c691191. Resolves #36, closed by the merge. PR body is
tmp/drafts/pr_trash_delete_v2.md.

Before the PR opened, the branch history was rewritten three times,
each with a backup ref kept locally:

- Dropped the three commits that added docs/testing/move_to_trash.md
  and scripts/gen_culling_test_data.sh. The owner did not want an
  internal test doc in the repo, and the generator script only served
  that doc. Both files are gone from history, not just from the tree.
- Reworded "Say why decode results carry paths and when each removal
  case happens" to "Describe in comments why decode results carry
  paths and when each removal happens".
- Reworded "renumbered" and "renumber" to "reindexed" and "reindex" in
  the body of the first commit, to match the code.

Two commits were added on top after the final review of the diff
against main: 5dd5598 (a clippy `get().is_none()` in a test) and
7e568d4 (the crate switch above).

## What the feature is, in one place

Delete, and Cmd+Backspace on macOS, moves the current image to the
platform trash and shows the next one, or the previous one at the end
of the folder. Key repeats are ignored so holding the key moves one
file. No confirmation on local disks. Edit > Move to Trash does the
same. A wastebasket glyph in each pane's footer, left of the counter,
accent color on hover, does the same for that pane regardless of pane
selection; Preferences > Display > Footer Buttons hides it. A toast at
the bottom shows the outcome, red on error.

Platform behavior:

- Windows: local drives go to the Recycle Bin. Network shares, mapped
  drives and removable media have none and the shell deletes
  permanently, so `lacks_recycle_bin` (UNC prefix, or `GetDriveTypeW`
  reporting remote or removable) triggers a "No Recycle Bin here"
  modal with Cancel and Delete permanently. Enter confirms, Escape
  cancels, keyboard navigation is blocked while it is open.
- macOS: `NSFileManager` `trashItemAtURL`, no Finder Automation
  prompt, Finder still offers Put Back. SMB volumes have no trash on
  macOS at all (050); the app shows the crate's error and the file
  stays.
- Linux: same mount as home goes to ~/.local/share/Trash, other mounts
  to `<mount>/.Trash-$uid`, the same place Nautilus uses. Read-only
  parent folder is refused before the crate runs, because 5.2.8 left a
  zero-byte placeholder in the trash on a failed move.

The app never unlinks. Every path goes through `trash_bin::move_to_trash`,
and a unit test scans src/ for `remove_file`, `remove_dir` and
`remove_dir_all`.

Index bookkeeping, the bulk of the work: all three caches are keyed by
file index. `remove_index` on `SlidingWindowCache` drops the slot,
reindexes `running_decodes`, `pending_decodes` and `pending_uploads`,
and loads one file to refill the window (or shifts the window when the
removal is outside it, which only happens in dual pane). `DecodeLruCache`
and `ThumbnailCache` shift their keys and keep byte totals and LRU
order. `DecodeResult` carries the path instead of the index, and
`running_decodes` maps path to current index, so a decode that
finishes after its file was trashed is dropped, and one whose file
shifted lands in the right slot. `Pane::remove_current` takes the
trash function as a parameter; nothing changes unless it returns Ok.
`App::trash_paths` moves each file through the pane that shows it and
drops it from every other pane that lists it.

Also fixed, pre-existing: the pane selection strips were not egui
widgets and toggled on any click over their rectangle, including
clicks on a menu row drawn on top. They are `ui.interact` widgets now.

Numbers: 15 files, +1792 / -204 against main before the two last
commits. 38 unit tests (36 run, 2 ignored). Testing across 049 to 051:
Linux 10 steps, macOS 15 steps plus the 121 x 10 MB PNG run, Windows
steps 11 to 13 plus the every-platform steps.

## Left open, owner's decisions

- Undo (Ctrl+Z restores the last trashed file). Own branch, 0.4.0.
- macOS SMB: error toast only. A permanent-delete confirmation like the
  Windows one was discussed, not decided.
- "Del / Cmd+Backspace" in the menu and tooltip reads wrong on a Mac
  keyboard (delete there is Backspace; Key::Delete is fn+delete).
- After Delete permanently on Windows the toast and log say "Moved to
  Trash" (`trash_paths`, src/app/culling.rs). Owner: nitpick, fix
  before the 0.4.0 release without booting Windows again if possible
  (it is a string change).
- Test machines still hold the shares, mounts and test copies from
  050 and 051; cleanup commands are in
  tmp/handoffs/2026-09-14_trash_merged_exif_next.md.

## Next

EXIF and metadata (#6, #7, #40, Andrew Law's email), the second item of
the 0.4.0 order agreed in 049. Handoff with the request details, code
pointers and the open design question (where the EXIF read runs) is
tmp/handoffs/2026-09-14_trash_merged_exif_next.md.

## Process notes for the record

Two sessions of PR drafting cost far more rounds than the code review.
The rules that came out of it are in the memory files
(feedback_pr_draft_versions and friends): read every file in
tmp/drafts/ before drafting, copy the accent presets draft layout, no
Summary header, cut every sentence a reader could not want the
opposite of, a shorter version is the same text cut into one _v2 file,
never overwrite the original, commit messages name no PR or issue
links. The diff review at the end found one clippy warning and nothing
else.
