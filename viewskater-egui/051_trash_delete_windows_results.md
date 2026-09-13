# Trash delete: Windows results and the network path bug in the trash crate

Date: 2026-09-13
Context: Windows testing of branch feat/trash-delete, following the
handoff tmp/handoffs/2026-09-13_windows_trash_testing.md. Owner tested
by hand in the app; the assistant set up the share, the mapped drive
and the USB copy, and verified on disk. Devlogs 049 (Linux) and 050
(macOS) are not edited.
Not verified claims are marked as such.

## Setup

- Machine: Windows 10 Home 10.0.19045, user `gota`, not elevated.
- Build: `cargo build --release` at c7c5263. The handoff said to test
  from 3df82e0; this clone already had the MacBook commit c7c5263 on
  top (pane strip click fix, no trash code), so it was tested as is.
- Test data: copied, not regenerated. ImageMagick is not installed here
  (the `convert` on PATH is the Windows filesystem tool). The set from
  `D:\ggando\culling_test_data` (generated on gota-home, same bytes the
  Linux and macOS runs used) was copied to:
  - `C:\Users\gotag\Pictures\culling_test\{generated_mixed,generated_other,readonly_dir}`
    for the local steps (C: is NTFS, `DRIVE_FIXED`).
  - `C:\Users\gotag\Pictures\culling_test\share\generated_mixed`, shared
    as `vs_share` (`net share vs_share=... /GRANT:gota,FULL` in an
    elevated prompt) and mapped with `net use V: \\localhost\vs_share`.
  - `E:\culling_test_mixed` on a FAT32 USB stick (`DRIVE_REMOVABLE`).
- D: is a local NTFS disk, not a share, so the originals there are
  not a network test location.

## Test results, Windows, 2026-09-13

Numbering follows tmp/test_plans/2026-09-12_move_to_trash_short.md
(Windows steps 11 to 13). Release build.

| Step | What | Result |
|---|---|---|
| 11 | Local NTFS drive, first Delete after launch | Pass (owner). File went to the Recycle Bin, no modal, no error toast. An `RPC_E_CHANGED_MODE` failure on the first `CoInitializeEx` would have surfaced as a "Could not move" toast, so the COM apartment question from 049 is answered. No log file to check: the app keeps the log in memory until exported from the menu. |
| 12 | Mapped `V:\generated_mixed`, Delete | Modal: Pass. "No Recycle Bin here / This location has no Recycle Bin, so Windows will delete the file permanently instead of moving it to the trash." with Cancel and Delete permanently. Cancel keeps the file. **Delete permanently: Fail.** Toast "Could not move 02_portrait.jpg to Trash: windows error: The system cannot find the file specified. (0x80070002)". File untouched on `V:` and in the backing folder, Recycle Bin empty. Cause below. |
| 12 | UNC path `\\localhost\vs_share\generated_mixed` opened directly | Not run in the app. The standalone repro (section below) gives the identical trace: canonicalizes to the same `\\?\UNC\` form, same 0x80070002 from the shell. |
| 13 | USB stick `E:\culling_test_mixed` (FAT32, `DRIVE_REMOVABLE`) | Pass (owner, fork build). Modal shown with the same text as step 12; three files confirmed (04_square.png, 09_landscape.tif, 13_landscape.jpg): all gone from the stick, none in the Recycle Bin, copied back afterwards. |
| 11-13 | Shell's own "permanently delete?" warning (`FOF_WANTNUKEWARNING`) | Still unknown. The shell never received a valid path, so whether it shows a dialog on a network location is not observed. |

Every-platform steps 1 to 11 on the local copy: owner reports them
as already tested; not recorded individually here.

## Bug: the trash crate cannot delete on any network path on Windows

trash 5.2.8 (latest on crates.io as of 2026-09-13), `src/windows.rs`,
`delete_specified_canonicalized`. Verified against the crate source in
`~/.cargo/registry` and by reproducing the canonicalization on this
machine.

1. `trash::delete` canonicalizes the parent folder first
   (`canonicalize_paths` in lib.rs, `Path::canonicalize`, which on
   Windows is `GetFinalPathNameByHandleW`). For a mapped drive that
   resolves through the redirector to the verbatim UNC form.
   Measured here with a ctypes call to the same API:

       V:\generated_mixed                   -> \\?\UNC\localhost\vs_share\generated_mixed
       E:\culling_test_mixed                -> \\?\E:\culling_test_mixed
       C:\Users\gotag\Pictures\...\generated_mixed -> \\?\C:\Users\gotag\Pictures\...\generated_mixed

2. Before `SHCreateItemFromParsingName` the crate strips exactly the
   four characters `\\?\`:

       let path_prefix = ['\\' as u16, '\\' as u16, '?' as u16, '\\' as u16];
       let wide_path_slice = if wide_path_container.starts_with(&path_prefix) {
           &wide_path_container[path_prefix.len()..]
       } else { ... };

   Local drives come out as `C:\...` and work. Network paths come out
   as `UNC\localhost\vs_share\generated_mixed\02_portrait.jpg`, which
   the shell does not recognize, hence `ERROR_FILE_NOT_FOUND`
   (0x80070002). The file is never touched.

3. A UNC path opened directly canonicalizes to the same
   `\\?\UNC\...` form, so it fails the same way (expected, not run).

Upstream: Byron/trash-rs issue #55 "Unable to delete file on Windows
shared folder", opened 2022-08-07, open, label "help wanted". The
reporter hit the exact same error on a Samba mount mapped to `Z:` with
trash 2.1.5. Byron's reply on 2022-08-09: "I have a hunch this is
related to a path-processing step happening when using trash, maybe
canonicalization." Nobody followed up with the diagnosis and there is
no PR. The stripping code is unchanged on master as of 2026-09-13. So
the symptom is reported upstream; the cause (the `\\?\UNC\` prefix
surviving the strip as `UNC\`) is not written down there.

The fix in the crate is small: after stripping `\\?\`, if the
remainder starts with `UNC\`, replace that with `\\` so the shell gets
`\\localhost\vs_share\...`. With that the `IFileOperation` call runs
with `FOF_ALLOWUNDO | FOF_NO_UI | FOF_WANTNUKEWARNING` on a network
path, which is the situation 049 expected to test (permanent delete,
possibly with the shell's own warning). Not verified until a patched
build runs.

Options for the app, owner's decision:

- Fork the crate with the three-line fix, pull it in through
  `[patch.crates-io]`, and send the fix upstream against #55. The
  fundamental route; the app keeps "the trash crate is the only
  deletion path".
- Until the fix exists, the modal's "Delete permanently" button does
  nothing useful on Windows network locations, only the error toast.
  Shipping like that would be misleading: the modal promises a
  permanent delete that then fails.

## Why the modal never appeared on macOS SMB (devlog 050)

By design, not a bug. `lacks_recycle_bin` in `src/trash_bin.rs` is
`cfg(target_os = "windows")` only and returns `false` everywhere else,
so `trash_or_confirm` in `src/app/culling.rs` goes straight to the
crate on macOS and Linux. On those platforms the crate returns an error
on a volume without a trash and the file stays, which is what 050
recorded ("the volume vs_share doesn't have one"). Windows is the one
platform where the shell would delete permanently instead of erroring,
so it is the only one with the confirmation. Whether macOS should get a
permanent-delete option on SMB is listed as an open decision in the
handoff.

## When the modal was written

Commit ca802cc "Add Move to Trash while navigating", 2026-09-11, the
first commit of the feature. The commit message already says "On
Windows, a path on a network share or removable drive has no Recycle
Bin, so a confirmation is shown first." The modal text, the
`lacks_recycle_bin` check and `pending_permanent_delete` all date from
that commit; later commits on the branch (footer button 6b1f817,
read-only precheck bd4eeb9) did not change it. Today was the first
time it was executed anywhere.

## State left on this machine

- Share `vs_share` exists and `V:` is mapped (non-persistent, gone
  after reboot). Remove with `net use V: /delete` and, elevated,
  `net share vs_share /delete`.
- `E:\culling_test_mixed` is on the USB stick, 16 files, untouched.
- Local copies under `Pictures\culling_test` are as the owner left them
  after the every-platform steps.

## The three path spellings, and why a four-character cut breaks one of them

Written down because it took a round of questions to get clear.

Windows has three ways to write a file address, each added on top of
the previous one for backward compatibility:

| Spelling | Example | Since |
|---|---|---|
| Drive letter | `C:\photos\a.jpg` | DOS, 1981 |
| UNC (Universal Naming Convention) | `\\hostname\share\a.jpg` | LAN Manager, late 80s |
| Verbatim | `\\?\C:\photos\a.jpg`, `\\?\UNC\hostname\share\a.jpg` | Windows NT, 1993 |

UNC: a drive letter never says which machine, the machine is implied.
A network path has to name it, so `\\` (a spelling no local path can
start with) means "host name next". Then the share, because a host
does not expose its disk, only named folders; `net share
vs_share=C:\...\share` publishes that folder under the name `vs_share`
and the client never sees the real location. Then a normal path inside
the share. A mapped drive (`V:`) is a nickname for `\\host\share` so
that drive-letter-only programs can reach it.

Verbatim: `\\?\` in front of a path tells the kernel to take the rest
literally: no `..` resolution, no trimming of trailing dots and
spaces, no 260-character limit, no `CON`/`NUL` device names. Rust's
`Path::canonicalize` on Windows calls `GetFinalPathNameByHandleW`,
which always answers in this spelling.

The snag: a UNC path already starts with `\\`, so `\\?\` + `\\host`
would be `\\?\\\host`, an empty segment between separators, which the
parser rejects. Microsoft's convention is to drop the network path's
own `\\` and write the token `UNC` in its place:

    \\?\  +  C:\a.jpg                =  \\?\C:\a.jpg
    \\?\  +  UNC\host\share\a.jpg    =  \\?\UNC\host\share\a.jpg

The word does nothing by itself; it is a reserved keyword in that one
position so the first segment after `\\?\` is always a real token the
parser can switch on (`C:` means a drive, `UNC` means "the next two
segments are host and share"). `UNC\` there stands for `\\`.
std::path knows all of these as distinct `Prefix` kinds: `Disk`,
`VerbatimDisk`, `UNC`, `VerbatimUNC`.

The bug, in those terms. The shell does not accept verbatim paths
(`SHCreateItemFromParsingName` on `\\?\...` returns 0x80070057, "the
parameter is incorrect", measured below), so the crate has to convert
back. It does so by dropping the first four `u16`s and keeping the
rest, `&wide[4..]`, without looking at what follows:

    index   0 1 2 3 4 5 6 7
            \ \ ? \ C : \ ...      -> C:\...                 valid
            \ \ ? \ U N C \ host   -> UNC\host\share\...     invalid

The second result starts with `UNC\`, which outside the verbatim
prefix means nothing. The shell reads it the normal way: not a drive
letter, not `\\`, not `\`, so a relative path to a folder named `UNC`
under the current directory. No such folder, `ERROR_FILE_NOT_FOUND`
(0x80070002). The file is never touched.

Two better ways were available. `Path::components()` already yields
`VerbatimUNC(host, share)`, so the crate could rebuild
`\\host\share\rest` from typed parts instead of slicing a wide string;
the app's own `lacks_recycle_bin` matches on those prefixes. And the
canonicalization is not needed at all: the shell only wants an
absolute path, and `std::path::absolute` gives one without producing
the verbatim form. Byron guessed "maybe canonicalization" on issue #55
in 2022; nobody checked.

## Standalone reproduction

`tmp/trash_unc_repro/` (local only, not in git), a cargo project with
trash 5.2.8 and windows 0.62. It repeats the crate's steps one at a
time and prints each intermediate string with its std prefix kind and
the shell's verdict, without deleting anything unless `--delete` is
passed:

    cd D:\ggando\viewskater-egui\tmp\trash_unc_repro
    cargo run -- V:\generated_mixed\02_portrait.jpg
    cargo run -- \\localhost\vs_share\generated_mixed\02_portrait.jpg
    cargo run -- E:\culling_test_mixed\02_portrait.jpg
    cargo run -- V:\generated_mixed\02_portrait.jpg --delete

Output for the mapped drive, 2026-09-13:

    input                    V:\generated_mixed\02_portrait.jpg
       std prefix            Prefix::Disk(V)
    1. after canonicalize    \\?\UNC\localhost\vs_share\generated_mixed\02_portrait.jpg
       std prefix            Prefix::VerbatimUNC("localhost", "vs_share")
    2. crate cuts 4 chars    UNC\localhost\vs_share\generated_mixed\02_portrait.jpg
       std prefix            no prefix, first component Normal("UNC") (relative path)
    3. what the shell says
       verbatim              -> ERROR 0x80070057 The parameter is incorrect.
       crate's string        -> ERROR 0x80070002 The system cannot find the file specified.
    4. fixed (UNC\ -> \\)    \\localhost\vs_share\generated_mixed\02_portrait.jpg
       std prefix            Prefix::UNC("localhost", "vs_share")
       fixed string          -> OK, resolves to \\localhost\vs_share\generated_mixed\02_portrait.jpg

The UNC path opened directly gives the identical trace from step 1 on,
so the UNC row in the results table is confirmed at the crate level
without running the app on it. The USB stick and the local copy
canonicalize to `\\?\E:\...` and `\\?\C:\...`, the cut yields `E:\...`
and `C:\...`, and the shell accepts them, so those locations are
unaffected.

With `--delete` on the mapped drive the crate itself reports the same
error the app's toast showed, and the file is still there afterwards:

    5. trash::delete(V:\generated_mixed\02_portrait.jpg)
       Err: Error during a `trash` operation: Os { code: -2147024894, description: "windows error: The system cannot find the file specified. (0x80070002)" }
       exists afterwards     true

The fixed string parses, but whether `IFileOperation` with
`FOF_ALLOWUNDO | FOF_NO_UI | FOF_WANTNUKEWARNING` then deletes
silently or shows the shell's warning on a network path is still not
observed. Only a patched crate build answers that.

## Fix in a fork, 2026-09-13

Fork: github.com/ggand0/trash-rs, branch `fix-windows-unc-parsing-name`,
three commits on top of v5.2.8. Local clone at
`C:\Users\gotag\projects\trash-rs`. No PR opened yet; draft in
tmp/trash-rs_pr_draft.md, issue comment drafts next to it.

The four-character cut is replaced by `to_shell_parsing_name`, which
rebuilds the path from `std::path::Prefix`: `VerbatimDisk` becomes
`C:\...`, `VerbatimUNC(host, share_name)` becomes `\\host\share_name\...`,
anything else is copied unchanged. The root's backslash is written from
the `RootDir` component and separators only go between name parts.

Commits:

1. c8bd24b Rebuild verbatim paths before handing them to the shell on
   Windows. The fix plus three tests.
2. 6f66b74 Rename the UNC share component to share_name.
3. fb9ee8a Keep the root backslash when rebuilding shell parsing names.
   The first version dropped the `RootDir` component, so `\\?\C:\`
   came out as `C:`, which the shell reads as the current directory on
   C:. Found while adding edge cases at the owner's request. Also a
   relative path like `dir\file.txt` lost its backslash. Eleven tests
   now: the two verbatim forms, non-verbatim and relative unchanged,
   empty path, drive root, share root with and without a trailing
   backslash, drive-relative `C:dir`, other verbatim and device
   prefixes, spaces and non-ASCII names, doubled and trailing
   separators, a path over 260 characters. One std quirk: `\\.\COM1`
   parses as a device prefix plus a root, so it gains a trailing
   backslash; the test uses `\\.\pipe\name` instead.

CI checks run locally on Windows, all clean: `cargo test`,
`cargo test --no-default-features --features coinit_apartmentthreaded`,
`cargo fmt --all -- --check`, `cargo clippy -- -D warnings`. The
warnings the build prints are in the crate's own `src/tests.rs` and
`examples/list.rs`, untouched by the branch. The full suite leaves its
own `trash-test-*`, `delete_me*`, `remove-me*` and
`test_file_to_delete_*` items in the Recycle Bin.

viewskater: `[patch.crates-io]` in Cargo.toml points `trash` at the
fork branch, Cargo.lock pins fb9ee8a. Both uncommitted on
feat/trash-delete. Re-resolving also moved iana-time-zone's
windows-core from 0.58.0 to 0.62.2, a compatible bump.

## Verification of the fix, 2026-09-13

| Run | Crate | Path | Result |
|---|---|---|---|
| App, owner pressed Delete then "Delete permanently" | fork c8bd24b | `V:\generated_mixed\06_tiny.png` | Gone from `V:` and from the backing folder, not in the Recycle Bin. Log: "Moved to trash: V:\generated_mixed\06_tiny.png" |
| Repro `--delete` | crates.io 5.2.8 | `V:\generated_mixed\02_portrait.jpg` | Err 0x80070002, file still there |
| Repro `--delete`, owner ran it | fork fb9ee8a | `V:\generated_mixed\02_portrait.jpg` | Ok, `exists afterwards false`, gone from `V:` and the backing folder, not in the Recycle Bin |

Both deleted files were copied back to the share from the untouched
set under `Pictures\culling_test\generated_mixed`. The repro's
Cargo.toml now carries the same `[patch.crates-io]` line; comment it
out to see the 5.2.8 failure again.

Step 12 of the Windows plan therefore passes with the fork: modal,
Cancel, and Delete permanently. Two items stay open:

- The app's log line and toast say "Moved to trash" after a permanent
  delete. Both strings are in `trash_paths` in `src/app/culling.rs`
  (lines 89 and 95); the modal confirm at line 229 calls the same
  function. Should say "Deleted permanently: name" when
  `lacks_recycle_bin` is true. Not fixed yet.
- Whether Windows shows its own "permanently delete?" dialog
  (`FOF_WANTNUKEWARNING`) on a network path: the owner has not said
  either way for the app run; the repro run printed no dialog note.
  Still unknown.

## Second repro, fork only, 2026-09-13

The first repro (`tmp/trash_unc_repro`) rebuilds the fixed string by
hand in its step 4 and only reaches the fork's real code in step 5,
which was confusing. `tmp/trash_fork_check/` replaces it for the proof:
one dependency, `trash = { git = "https://github.com/ggand0/trash-rs.git",
branch = "fix-windows-unc-parsing-name" }`, and a `main` that prints
the canonical path, calls `trash::delete`, and prints whether the file
still exists.

    cd /d/ggando/viewskater-egui/tmp/trash_fork_check && cargo run -- 'V:\generated_mixed\03_landscape.png'

Run twice, once by the assistant and once by the owner, same output:

    path          V:\generated_mixed\03_landscape.png
    canonical     \\?\UNC\localhost\vs_share\generated_mixed\03_landscape.png
    exists before true
    trash::delete Ok
    exists after  false

Both times the file was gone from `V:` and from the backing folder and
absent from the Recycle Bin, then copied back. This is the cleanest
evidence: the fork's `trash::delete` alone, no app, no hand-built
string. `trash_unc_repro` keeps its value as the 5.2.8 failure demo
(comment out the `[patch.crates-io]` block at the bottom of its
Cargo.toml).

While restoring, `08_landscape.bmp` and `11_animated.gif` were also
missing from the share, presumably from the owner's own app runs; both
restored from the untouched set. The Recycle Bin holds dozens of
`trash-test-*`, `delete_me*`, `remove-me*` and `test_file_to_delete_*`
items left by the crate's own test suite (three full runs); the
os_limited tests purge theirs, `tests/trash.rs` does not. Safe to empty.

## PR opened upstream, 2026-09-13

https://github.com/Byron/trash-rs/pull/150, "Fix deleting files on
network paths on Windows", opened by the owner at 06:06Z. Base
`Byron/trash-rs:master`, head `ggand0:fix-windows-unc-parsing-name`,
the three commits c8bd24b, 6f66b74, fb9ee8a. Body is the draft in
tmp/trash-rs_pr_draft.md after the owner's edits: symptom, cause with
the 5.2.8 code, the two before/after tables, the fix as three
`Prefix` bullets, the note that the shell deletes permanently on a
network path, the tests line, the `net share` / `net use` commands,
and a link to the app branch. "Fixes #55" closes the issue on merge.

CI started on its own, no first-contributor approval gate. All four
jobs green within about twenty minutes: build (linux), (macos),
(windows), (netbsd). Only the windows job compiles `windows.rs`; the
other three could not be affected by the change.

The two issue-comment drafts (tmp/trash-rs_issue55_comment_draft.md,
tmp/trash-rs_issue55_comment_short.md) were not posted; the PR body
carries the same diagnosis and GitHub links it on the issue.

What the repo expects, checked before opening: no CONTRIBUTING file,
nothing about PRs in the README. `.github/workflows/rust.yml` runs
`cargo test`, `cargo test --no-default-features --features
coinit_apartmentthreaded`, `cargo fmt --all -- --check` and `cargo
clippy -- -D warnings` on Linux, macOS, Windows, plus a NetBSD build
and a Linux container test for the freedesktop backend. All four ran
clean here on Windows (owner's run too). `rustfmt.toml` sets a
120-column width. CHANGELOG.md is generated by cargo-smart-release from
commit messages and must not be edited by hand; some contributors use
`fix:` prefixes so the tool files them under bug fixes, Byron's own
commits do not. Ours are plain imperative titles; not changed.

Wording rounds on the PR body, for the record: "verbatim" is Rust's
word (`Prefix::VerbatimDisk`, `VerbatimUNC`), not Microsoft's, so the
body says "the `\\?\` form"; the sentence about the root backslash and
drive-relative paths was cut because both cases are unreachable from
`trash::delete`; the tests line was shortened from a list of eleven
cases to the two conversions plus "paths without `\\?\` that should be
returned unchanged, plus a few edge cases"; "(elevated prompt)" became
"# run as administrator" above the `net share` line.

Things asked and answered along the way, kept because they will come
up again:

- `full_path` in `delete_specified_canonicalized` is the path after
  canonicalization, always `\\?\...` on Windows.
- `components()` is the typed cousin of `pathlib.PurePath.parts`: the
  first piece is a `Prefix` saying which spelling the path uses, then
  `RootDir`, then one `Normal` per name.
- `share_name` in `VerbatimUNC(host, share_name)` is the name the host
  publishes the folder under (`vs_share`), not the folder's real name.
- "The shell" is the Explorer layer and its COM API. `trash::delete`
  does not delete anything itself: it creates an `IFileOperation`
  (`windows::Win32::UI::Shell`), sets `FOF_ALLOWUNDO`, queues the item
  with `DeleteItem`, and `PerformOperations()` (windows.rs line 91) is
  where the file moves to the Recycle Bin or, on a network path, is
  destroyed. That is why the PR body says the shell deletes
  permanently, not the crate.
- Trailing spaces and dots in a name are trimmed by Win32 before
  `canonicalize` opens the file, so they never reach the function
  unless the input was already `\\?\`. Spaces elsewhere are ordinary.

## viewskater state after the PR

- feat/trash-delete has one new local commit, 416af37 "Patch the trash
  crate with the fork that fixes network paths on Windows" (Cargo.toml
  `[patch.crates-io]` entry plus Cargo.lock pinned to fb9ee8a). Not
  pushed; origin's feat/trash-delete is still at c7c5263, so the app
  link in the PR body shows a Cargo.toml without the patch until it is
  pushed.
- `docs/testing/move_to_trash.md` has an uncommitted edit to step 16
  adding the `net share` / `net use` commands, the UNC-path check and
  the cleanup. It was committed by mistake and uncommitted at the
  owner's request; left in the working tree, not discarded.
- Still open: the log line and toast after a permanent delete say
  "Moved to trash" (`trash_paths`, src/app/culling.rs lines 89 and 95);
  and whether Windows showed its own dialog during any of the network
  deletes was never stated by the owner, so `FOF_WANTNUKEWARNING` on a
  network path stays unobserved. Step 13 (USB stick) passed later the
  same day, see the results table.
