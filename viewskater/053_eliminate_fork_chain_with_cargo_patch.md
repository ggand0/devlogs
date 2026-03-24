# Eliminate Fork Chain with Cargo [patch]

**Date:** 2026-03-24

## Problem

The current dependency structure requires **4 forked repos** chained together:

```
winit (ggand0/winit)
  └─ iced (ggand0/iced) Cargo.toml points to winit fork
       └─ iced_fonts (ggand0/iced_fonts) Cargo.toml points to iced fork's iced_core
       └─ iced_aw (ggand0/iced_aw) Cargo.toml points to iced fork + iced_fonts fork
            └─ data-viewer
```

Every change to winit or iced requires a cascading commit+push across all intermediate repos in dependency order. Switching between local path deps and git deps for development requires editing Cargo.toml in **all four** repos.

### Audit of actual code changes per fork

| Fork | Code changes | Cargo.toml-only changes |
|------|-------------|------------------------|
| **winit** | Yes — X11 DnD position fix, DnD cursor support | — |
| **iced** | Yes — custom event loop, widget customizations | — |
| **iced_fonts** | **Zero** | 100% — just repoints iced_core dep |
| **iced_aw** | **Zero** | 100% — just repoints iced + iced_fonts deps |

**iced_fonts and iced_aw forks exist solely to redirect dependency pointers.** They have no code changes whatsoever.

## Solution: Cargo [patch]

Cargo's `[patch]` section overrides any dependency anywhere in the transitive dependency tree, from a single location (data-viewer's Cargo.toml). This eliminates the need for iced_fonts and iced_aw forks entirely.

### How upstream deps reference iced

- **iced_aw upstream** (`iced-rs/iced_aw` v0.11.0): depends on `iced` via `git = "https://github.com/iced-rs/iced.git"` rev `b474a2b7a763dcde6a377cb409001a7b5285ee8d`, and `iced_fonts = "0.1.1"` from crates.io
- **iced_fonts upstream** (`Redhawk18/iced_fonts` v0.1.1): depends on `iced_core` via workspace dep from crates.io (`iced_core = "0.13"`)

### Proposed data-viewer Cargo.toml

```toml
# Direct deps — use upstream iced_aw, custom iced
iced_custom = { package = "iced", git = "https://github.com/ggand0/iced.git", branch = "custom-0.13", features = [...] }
iced_winit = { git = "https://github.com/ggand0/iced.git", branch = "custom-0.13" }
iced_wgpu = { git = "https://github.com/ggand0/iced.git", branch = "custom-0.13" }
iced_widget = { git = "https://github.com/ggand0/iced.git", branch = "custom-0.13", features = ["wgpu"] }
iced_core = { git = "https://github.com/ggand0/iced.git", branch = "custom-0.13" }
iced_runtime = { git = "https://github.com/ggand0/iced.git", branch = "custom-0.13" }
iced_futures = { git = "https://github.com/ggand0/iced.git", branch = "custom-0.13", features = ["tokio"] }
iced_graphics = { git = "https://github.com/ggand0/iced.git", branch = "custom-0.13" }
iced_aw = { git = "https://github.com/iced-rs/iced_aw.git", tag = "v0.11.0", features = ["menu", "quad", "tabs"] }

# Override iced deps coming from iced_aw -> upstream iced-rs/iced.git
[patch."https://github.com/iced-rs/iced.git"]
iced = { git = "https://github.com/ggand0/iced.git", branch = "custom-0.13" }
iced_core = { git = "https://github.com/ggand0/iced.git", branch = "custom-0.13" }
iced_widget = { git = "https://github.com/ggand0/iced.git", branch = "custom-0.13" }
iced_runtime = { git = "https://github.com/ggand0/iced.git", branch = "custom-0.13" }
iced_winit = { git = "https://github.com/ggand0/iced.git", branch = "custom-0.13" }

# Override iced_core coming from iced_fonts (crates.io) -> iced_core (crates.io)
[patch.crates-io]
iced_core = { git = "https://github.com/ggand0/iced.git", branch = "custom-0.13" }
iced_fonts = { git = "https://github.com/Redhawk18/iced_fonts.git", tag = "v0.1.1" }
```

For local iced development, only `[patch]` entries change:

```toml
[patch."https://github.com/iced-rs/iced.git"]
iced = { path = "../iced" }
iced_core = { path = "../iced/core" }
# ...

[patch.crates-io]
iced_core = { path = "../iced/core" }
```

### What this eliminates

- **iced_fonts fork** — upstream iced_fonts from crates.io, iced_core patched
- **iced_aw fork** — upstream iced_aw from iced-rs, iced patched
- **Cascading Cargo.toml edits** across 4 repos when switching local/remote
- **Cascading commit+push** for every iced or winit change

### What remains

- **winit fork** — has real code (DnD support, X11 position fix)
- **iced fork** — has real code (custom event loop, widget customizations)
- **data-viewer** — one `[patch]` section to rule them all

## Risks / Things to Verify

1. **Version compatibility**: `[patch]` requires the replacement crate to have a semver-compatible version with the original. Our iced fork is 0.13.5. Upstream iced_aw pins to a specific rev of iced 0.13.x. Should be compatible, but must verify Cargo resolves it.
2. **Duplicate iced_core**: If `[patch.crates-io]` and `[patch."https://github.com/iced-rs/iced.git"]` both override `iced_core`, Cargo must unify them to the same source. Both point to the same git+branch so this should work, but needs testing.
3. **iced_fonts workspace dep**: Upstream iced_fonts uses `iced_core.workspace = true` which resolves to a crates.io version. The `[patch.crates-io]` for iced_core should override this. Needs verification.
4. **Feature flags**: Upstream iced_aw may enable different default features than the fork. The fork disables defaults and enables `["advanced", "wgpu"]`. The data-viewer Cargo.toml must match.

## Relationship to DnD Fix

This is a **separate concern** from the dual-pane DnD fix. The DnD fix only touches:
- winit: already pushed (rev `7dea6466`)
- iced: update winit rev in Cargo.toml on `custom-0.13`
- data-viewer: `split.rs` and `synced_image_split.rs` event position handling

The DnD fix should be done first using the existing fork chain (push up iced, switch data-viewer back to git deps, commit on feature branch). The `[patch]` migration is a follow-up refactor that can be done once the DnD fix is merged and tested.

Doing both at once conflates a behavior fix with a dependency resolution restructure — if something breaks you won't know which change caused it.
