# 0058 Open-source prep: history scrub, repositioning, repo split

Date: 2026-07-26

Prep for the public release (X demo post + repo link).

## Repositioning

The project no longer presents as an M2TW clone. Title-screen tagline
is now "Massed medieval battles, every soldier simulated" (hardware-
agnostic on purpose: no soldier-count claims that depend on the dev
GPU). README intro reframed the same way: M2TW is the measured
starting point for the battle model, the point is the scale and going
beyond the classics. README also gained Assets (code-built art;
ElevenLabs SFX + one freesound loop) and License (MIT) sections, and
Cargo.toml gained description/license metadata.

## Dev-note scrub

"Owner" phrasing (AI-session vernacular) removed from every tracked
surface:

- 14 code comments across src/ reworded to neutral phrasing (combat,
  audio, formation, regiments, render_units, movement, the WGSL
  shader). cargo check + clippy clean; behavior untouched.
- 11 commit bodies rewritten via `git filter-branch --msg-filter` with
  per-commit anchored perl replacements (work/scripts/reword-owner-commits.pl).
  All subjects were already clean. Verified after rewrite: 0 owner /
  claude / co-authored mentions in all 138 messages, tree hash
  byte-identical (96731c8), commit count unchanged.
- PR records: comments/reviews all clean; PR #6 body has one "owner"
  mention, moot because PRs stay on the private repo (below).

## Repo split (chat history preserved)

GitHub PR pages keep old commits reachable (refs/pull/*) even after a
force-push, so the cleaned history went to a fresh repo instead:

- ggand0/flanks renamed to ggand0/flanks-dev (private; keeps all 10
  PRs and the pre-rewrite records).
- Fresh private ggand0/flanks created; rewritten main pushed. Flip
  public at release time.
- Local working dir unchanged (Claude Code chats are keyed by the
  directory path, so all sessions survive). Remotes: origin = flanks,
  dev = flanks-dev.

## Round 2: merge commits and identity

Follow-up pass (same day): the 10 GitHub merge commits still read
"Merge pull request #N from ggand0/..." with committer
GitHub <noreply@github.com>, betraying the PR workflow. One
filter-branch pass rewrote them to vanilla "Merge branch 'feat/x'"
subjects (PR-title bodies dropped) and unified every author/committer
to Gota Gando <gota@gando.dev> (the web-UI "Initial commit" was
github@ggando.me). Verified: all 276 identity slots unified, date
checksum identical pre/post (contribution graph unaffected in timing),
tree identical, 0 "pull request" mentions. Force-pushed to
origin/flanks. Extra backup: refs/backup/main-pre-mergefix-20260726 +
work/backups/flanks-pre-mergefix-20260726.bundle.

## Backups

- /data/backups/flanks-full-20260726/flanks/ — entire working dir
  (42G, includes target/).
- /data/backups/flanks-full-20260726/claude-chats/ — the flanks,
  cascade, and cascade.bak Claude project dirs (315M).
- refs/backup/main-pre-reword-20260726 + verified bundle
  work/backups/flanks-pre-reword-20260726.bundle — pre-rewrite git state.

## Round 3: license, README rewrite, final checks

- Dual-licensed MIT OR Apache-2.0 (LICENSE-MIT + LICENSE-APACHE,
  Cargo.toml license field, standard Rust blurb in the README).
- Second slop pass over all commit messages: dumped the full log
  (work/backups/all-commit-messages.txt), grep batteries for workflow tells /
  LLM stock phrasing / pronouns, plus a full manual read of all 1614
  lines. Clean; the he/his/him hits all refer to simulated soldiers.
- README rewritten for a general audience on the viewskater template
  (github.com/ggand0/viewskater): short plain intro, one-line feature
  bullets, controls table (verified against KeyCode/MouseButton
  usage), minimal build section; engine internals, the map-art item,
  and the verification-process paragraph cut, FL_* flags get one
  sentence. Style rule from the owner: no "masturbatory" internal
  process text, no device-specific numbers without the tested
  hardware (scale claim is now "a few hundred thousand, 200k tested
  on an RTX 3090").
- Marching loop attribution corrected: it was labeled freesound but
  is Pixabay sfx 32908 (people-marching-loop); file renamed to
  bed_march_loop_14.5s.mp3, README links the source and notes audio
  keeps its own license. Pixabay Content License allows commercial
  use without attribution. All other SFX are ElevenLabs-generated.
- Irreversibility sweep before flipping public: full-history secret
  scan (git log -p grep for keys/tokens/paths) — clean; only
  identity in history is gota@gando.dev, already public via
  viewskater commits. Name check: no "Flanks"/"Flank" title on
  Steam, no exact "Flanks" on itch.io (only small FLANK jam games),
  "flanks" crate name free on crates.io.
- Hero screenshot added: 20k-unit battle with the perf overlay
  (205 fps) at assets/screenshot.jpg (PNG converted to q85 JPG,
  1.5 MB -> 208 KB).

## PUBLIC

ggand0/flanks flipped public 2026-07-26 (owner did the flip after
the screenshot landed). Consequences from here on:

- NO force-pushes or history rewrites on origin/main, ever. The
  private-era force-with-lease permission is lapsed.
- Commit messages on main are public-facing prose now.
- CLAUDE.md stays untracked and never committed; also never add it
  to the public .gitignore (the ignore line itself advertises AI
  tooling). Insurance belongs in the global ignore
  (~/.config/git/ignore) if wanted.

## Still open (non-blocking)

- Feature branches were NOT rewritten (main only); don't push them to
  the public repo without a scrub pass.
- .gitignore is still the Python template (works, target/ covered) —
  cosmetic.
- GitHub topics + profile pin for discoverability.
- Demo video link in the README once the X post exists.
