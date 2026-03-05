# LeRobot Dataset Curation Tool — Planning Notes

## Problem

During IL data recording with the SO-101, failed or low-quality episodes must be identified manually — either by taking notes during recording or watching each episode back one by one afterward. There's no fast, keyboard-driven workflow for reviewing episodes and marking/deleting bad ones.

## Existing Tools & Why They Don't Fit

| Tool | Why it doesn't fit |
|---|---|
| **lero** (masato-ka/lero) | Python (slow), 10 stars, unprofessional commits, no tight recording-loop integration |
| **lerobot-dataset-visualizer** (HF) | Next.js web app, requires Docker or upload, not local-first |
| **Rerun viewer** | General-purpose multi-modal visualizer for teams/production; wrong abstraction level; adding curation features would be upstream scope creep |
| **lerobot-edit-dataset CLI** | Can delete episodes by index, but no visual review — you still have to watch episodes manually to know which indices to delete |

**Gap:** No fast, local, keyboard-driven GUI tool for the individual researcher's record → review → mark bad → delete loop.

## Proposed Tool

A small, standalone Rust app for LeRobot dataset curation. Scope: **LeRobot format only, v1 MVP**.

**Core workflow:**
1. Open a LeRobot dataset folder
2. Browse episode list, play video per episode
3. Mark episodes bad/good with keystrokes during review
4. Commit: delete flagged episodes (via file ops + metadata rewrite, or shell out to `lerobot-edit-dataset`)

**Positioning vs Rerun:** Rerun is a production-grade multi-modal visualizer. This tool is a focused curation utility for individual researchers — same relationship as a text editor vs an IDE.

---

## GUI Framework Decision

### viewskater (current architecture — iced)

- Uses **iced** with a **custom event loop** to work around iced's event-driven model
- Achieves fast image rendering via **GPU-side wgpu texture caching** — textures live in GPU memory, allowing instant frame display
- Currently achieves ~8–10 FPS on 10MB 4K images navigating a directory
- Pain points: iced's Elm-style architecture (Message enum + update/view) gets complex and scattered for apps that need continuous per-frame rendering; the custom event loop was necessary but adds boilerplate

### egui — considerations

**Pros:**
- Immediate mode natively fits continuous video playback / scrubbing (no fighting the framework)
- Async/background work done via channels — background thread decodes video, sends frames to main thread, egui polls each frame and uploads as texture. Cleaner than iced's subscription model for this use case
- Mature enough (used by Rerun viewer); Emil Ernerfeldt maintains it as part of Rerun's core needs so it gets serious attention
- Less code for the rendering loop pattern this tool needs

**Cons:**
- Layout system is flow-based; complex multi-panel layouts (sidebar + main + timeline) can get awkward
- No existing egui image viewer matches viewskater's GPU caching performance — would need to reimplement the fast rendering core from scratch
- "Maintained by one guy" (though that guy's main project depends on it)

**Verdict (current thinking):** Starting with iced + lifting the existing custom event loop + wgpu texture pattern from viewskater is the faster path to an MVP. The GUI framework question can be revisited after the core curation workflow works. egui is worth a prototype to evaluate layout feasibility before fully committing.

---

## Rust Stack (planned)

| Concern | Crate |
|---|---|
| GUI | iced (carry over from viewskater) or egui (to evaluate) |
| Parquet reading | `polars` or `arrow2` |
| JSON/JSONL parsing | `serde_json` |
| Video decoding | `ffmpeg-next` bindings, or shell out to ffmpeg/mpv |
| GPU texture upload | `wgpu` (already used in viewskater) |
| Background loading | `std::sync::mpsc` channels |

**Note on video:** Decoding MP4 in pure Rust is still painful. `ffmpeg-next` is the pragmatic choice. Alternatively, decode frames in a background thread and pass raw RGBA over a channel to the render thread — same pattern viewskater uses for images.

---

## Repo Strategy

**New repo** (not a viewskater PR or fork). Reasons:
- viewskater's core abstraction is image sequences; this tool's core abstraction is a LeRobot dataset (Parquet + MP4 + JSONL metadata)
- Different enough that mixing them pollutes both codebases
- Could share crates later if overlapping rendering code emerges

**Future format support:** RLDS support (for SpatialVLA workflow) is on the horizon but deliberately out of scope for v1. Design the episode abstraction loosely enough that adding a second format doesn't require a full rewrite.

---

## LeRobot Dataset Format Reference (v2.1)

```
dataset_root/
├── data/
│   └── chunk-000/
│       ├── episode_000000.parquet
│       └── ...
├── videos/
│   └── chunk-000/
│       ├── observation.images.main/
│       │   ├── episode_000000.mp4
│       │   └── ...
│       └── observation.images.wrist/
│           └── ...
└── meta/
    ├── info.json          # dataset-level metadata
    ├── episodes.jsonl     # per-episode metadata
    ├── tasks.jsonl        # task descriptions
    └── episodes_stats.jsonl  # per-episode stats (v2.1, enables easy deletion)
```

v2.1 is the target — per-episode stats make deletion much cleaner than v2.0's global `stats.json`.

Rerun's `re_data_loader::lerobot` module (Rust) is a useful reference for parsing this format.

---

## Open Questions

- egui layout feasibility for sidebar + video + timeline — worth a 2–3 hour prototype before deciding
- Whether to implement episode deletion directly in Rust or shell out to `lerobot-edit-dataset`
- MVP feature scope: video playback + mark/delete is the core; joint angle plots are nice-to-have
