# Devlog 020 -- Fix trajectory loading for v3 multi-file datasets

## Problem

When a LeRobot v3 dataset spans multiple parquet data files (e.g. after a
recording session was interrupted and resumed), trajectories for episodes in
any file other than `file-000.parquet` silently failed to load.

Discovered while inspecting `lbracket-pick-place-practice-v3-10ep_gota`:
episodes 0-5 lived in `data/chunk-000/file-000.parquet`, episodes 6-8 in
`file-001.parquet`. Selecting episodes 6-8 in the grid showed no highlighted
trajectory because the loader returned zero rows for those episodes.

## Root causes

Two independent bugs, both triggered by resumed recording sessions:

### 1. Wrong data file path

`trajectory::episode_data_path()` hardcoded `file-000.parquet` for all v3
episodes. It checked whether `file-000.parquet` existed and returned it
immediately, never consulting the episode metadata to find the correct file.

The episode metadata parquet already contained `data/chunk_index` and
`data/file_index` columns with the correct per-episode file mapping, but
the app only read the `videos/*/chunk_index` and `videos/*/file_index`
columns for video playback -- not the data file columns.

### 2. Variable-length list encoding in observation.state

Even after fixing the file path, `lbracket-pick-place-50ep_gota` still
failed for episodes 3-4. Their data file (`file-001.parquet`) encoded
`observation.state` as `list<float>` (variable-length) instead of
`fixed_size_list<float>[24]`. The loader only handled `FixedSizeListArray`
and rejected the column outright.

This is not specific to any particular recording script. Both encodings
are valid parquet representations of the same data. Different pyarrow
versions, writer settings, or session restarts can produce either encoding.
A robust reader should accept both.

## Fixes

- `dataset.rs`: Added `data_chunk_index` and `data_file_index` fields to
  `EpisodeMeta`. Parse `data/chunk_index` and `data/file_index` from the
  v3 episode metadata parquet during dataset loading.
- `trajectory.rs`: `episode_data_path()` now takes the metadata-provided
  chunk and file indices and constructs the path directly instead of
  guessing. `load_episode_states()` tries `FixedSizeListArray` first,
  then falls back to `ListArray` using per-row offsets.
- `ui/panels.rs`: The trajectory cache loader looks up the episode's
  metadata to pass the correct indices.

## Files changed

- `src/dataset.rs`
- `src/trajectory.rs`
- `src/ui/panels.rs`
