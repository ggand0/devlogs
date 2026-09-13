# Trash delete: macOS results, SMB finding, session record

Date: 2026-09-13
Context: continues 049. Second macOS pass on branch feat/trash-delete
at c7c5263 (strip fix), driven by the owner in the app with the
assistant preparing volumes and verifying on disk. Records the SMB
finding, the Put Back collision, the Delete key semantics, the state
left on the machine, and the process failures of the two sessions.
Numbering follows tmp/test_plans/2026-09-12_move_to_trash_short.md.
Not verified claims are marked as such.

## Where the code is

- Strip fix c7c5263 "Route pane strip clicks through egui so menu
  clicks stop toggling selection": on the MacBook only. gota-home and
  origin are at 3df82e0. The 049 macOS section describes this fix but
  gota-home's checkout does not contain it.
- docs/testing/move_to_trash.md: modified, uncommitted, on the MacBook.
- Nothing pushed.

## macOS results, owner in the app

| Step | Result |
|---|---|
| 14 | fn+delete trashes, Cmd+delete trashes. The key labelled "delete" on a Mac keyboard alone does nothing, by design (it is Backspace; the code binds Delete on all platforms and Cmd+Backspace on macOS, same as Finder). Pass. |
| 14 Put Back | Works. See the collision note below. |
| 15 | SMB share: toast "Could not move other_a.jpg to Trash: ... the volume "vs_share" doesn't have one", file untouched on the share and in the backing folder, no .Trashes created. This is the only outcome macOS allows on SMB. Pass. |
| 2 | Hold Delete for 2 s, exactly one file goes. Done by the owner. Pass. |

Steps 1, 3 to 8 passed in the 2026-09-12 macOS pass (see 049). All macOS steps are done.

## SMB on macOS never has a trash

Every SMB server tried gives the same NSFileManager error, and the
error is decided on the client before any request reaches the server.

| Server | Apple SMB extensions | .Trashes/503 pre-created | Result |
|---|---|---|---|
| impacket smbserver (Python) | no | no | "volume doesn't have one" |
| impacket smbserver | no | yes | same |
| Samba 4.24.7, vfs_fruit, AAPL negotiated, smbutil reports UNIX_SUPPORT and OS_X_SERVER true | yes | yes | same |
| macOS smbd (File Sharing) | yes | | not testable: the owner's account has no SMB-NT hash and guest returns NT_STATUS_ACCOUNT_RESTRICTION; both need the File Sharing account toggle |

Decisive evidence: Samba at log level 10 recorded 11,953 lines during
the trashItemAtURL call and none of them mention the file or
.Trashes. macOS refuses from the mount type alone.

External confirmation, quoted:
- discussions.apple.com/thread/6719628: "However with a network volume
  if a logged in user deletes a file or folder it normally gets deleted
  immediately." and "It used to be the case that even when using a
  network file server you still had the benefit of files first going
  to the trash folder but Apple stopped doing this a long time ago."
- forum.keyboardmaestro.com LAN Trash post: Cmd+Backspace "will only
  give you the option of deleting the files immediately if they are on
  another Mac".
- github.com/marta-file-manager/marta-issues/issues/147: "Cannot trash
  files/folders on SMB network volume", same API.

Consequences, no decision taken:
- Step 15 in both test docs says "or the file in that volume's trash".
  That branch cannot happen on macOS. The docs are not edited.
- On macOS the app cannot delete anything on an SMB share. Windows has
  the "No Recycle Bin here" modal with Delete permanently; macOS has
  no equivalent. Whether to add one is the owner's call. Not built.
- The toast shows the crate's raw string with a doubly quoted path and
  the backticked method name. Only the last clause is useful. Not
  changed.

The with-trash branch of the same code path was confirmed on a
non-boot APFS disk image: TMPDIR pointed at /Volumes/vs_dmg,
`cargo test --profile opt-dev real_trash_round_trip -- --ignored`
created /Volumes/vs_dmg/.Trashes/503/ and moved the file there with a
.DS_Store beside it. That is a local-volume check, not step 15.

## Put Back goes to the wrong folder: Finder's in-memory table

Symptom: duplicate a folder, trash a file from the duplicate with the
app, Put Back in Finder, and the file lands in the original folder.
Dragging it out of the Trash by hand goes wherever you drop it, and
trashing with Finder itself does not show the problem.

Verified on the disk image /Volumes/vs_dmg, whose trash folder
.Trashes/503 is readable from a shell (unlike ~/.Trash), by parsing the
ptbL records in its .DS_Store (tmp/dsrec.py in the repo):

1. App call (NSFileManager trashItemAtURL, same as trash_bin.rs) on
   orig/repro.jpg: record `repro.jpg -> /orig/`. Owner: Put Back in
   Finder. File returned to orig. Record left on disk unchanged.
2. App call on dup/repro.jpg: record rewritten to `repro.jpg -> /dup/`.
   Confirmed on disk before the next step.
3. Owner: Put Back on that item. Finder moved it to orig, hit the file
   from step 1, offered "An older item named repro.jpg already exists",
   Keep Both produced orig/"repro copy.jpg". dup stayed empty.

So Finder resolves Put Back from a table it keeps in memory, keyed by
item name, filled when it trashes or puts back an item, and consults
that before the .DS_Store record. The app updates the record on disk
correctly and immediately; Finder's own trash writes the record about
ten seconds late but also correctly. A name that Finder has never
handled in this session (the zz_putback_probe.jpg test, the
"repro 7.42.02 PM.jpg" item) gets the disk record and goes to the right
place. The stale entry survived eight hours between step 1 and step 3.

Not verified: whether `killall Finder` clears the table (expected: yes,
since the disk records are right).

Consequence for the app: this is the cost of DeleteMethod::NsFileManager.
The crate's DeleteMethod::Finder drives Finder through AppleScript, so
Finder's table would stay current, at the price of the "wants to control
Finder" permission prompt that 049 chose to avoid. No change made;
owner's call. Testing rule meanwhile: unique file names per source
folder, or restart Finder between rounds.

## Delete key on macOS

trash_key_pressed in src/app/handlers.rs accepts Key::Delete on every
platform and Cmd+Backspace on macOS. On a Mac keyboard "delete" is
Backspace, so alone it does nothing; fn+delete is Key::Delete. The
menu label and footer tooltip say "Del / Cmd+Backspace", which a Mac
user reads as their delete key and a key they do not have. Label fix
not made; owner's call.

## Samba 4.24 on macOS cannot create directories as non-root

For the record, since it cost time: smbd 4.24.7 from Homebrew creates
directories with a temp name at mode 0000 and then renames; on macOS
the rename fails NT_STATUS_ACCESS_DENIED for a non-root smbd, the
client sees EACCES, and the directory is left behind with mode 000.
`force directory mode` does not affect the temp mkdir. Not relevant to
the SMB conclusion above because macOS never reaches the server.

## State left on the MacBook

- Samba installed via Homebrew (`samba-dot-org-smbd`). Not running.
- Share point vs_share registered with `sharing -a` (owner ran it).
  Remove: `sudo sharing -r vs_share`.
- Guest account enabled (`sysadminctl -guestAccount on`, owner ran
  it). Disable: `sudo sysadminctl -guestAccount off`.
- AllowGuestAccess = 1 and EnabledServices unset in
  /Library/Preferences/SystemConfiguration/com.apple.smb.server.
- smbd (macOS) loaded via launchctl, listening on 445, accepts no
  logins.
- ~/ggando/vs_data/culling_test_data/: generated_* mirrored from
  gota-home plus mac_work_mixed, "mac_work_mixed copy" (partially
  trashed), mac_work_other, mac_4k, zz_putback_probe.jpg in the copy.
- No mounts, disk image detached, impacket and Samba servers stopped.

## Process failures, both sessions

2026-09-12 session: asked to read 049, read the short plan and rsync
test data, it ran a full macOS pass, wrote helper tools, committed the
strip fix and a docs change without being asked, appended to 049 and
rsynced it over the gota-home copy, built without rechecking the
branch tip, then explained instead of accepting corrections.

2026-09-13 session: misread which document step 15 referred to,
reported the disk image result as the second half of step 15 when it
is not SMB, cycled through impacket, Docker, Samba and macOS smbd for
over an hour before producing the server-log proof that settles the
question, and proposed the permanent-delete modal only after the
owner named it. Started to implement it without an order, stopped
when interrupted. Nothing was changed in the repo.
