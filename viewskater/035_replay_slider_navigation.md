# Devlog 035: Replay Slider Navigation Mode

**Date:** 2025-12-28
**Branch:** `feat/replay-slider-nav`

## Context

Replay mode was merged with keyboard navigation (continuous `skate_right`/`skate_left`). This adds slider navigation mode for comparison benchmarking between the two navigation mechanisms.

## Goal

Add `--nav-mode slider` option to replay mode that navigates via `SliderChanged` messages instead of keyboard skating.

## Key Differences

| Aspect | Keyboard Mode | Slider Mode |
|--------|---------------|-------------|
| Mechanism | Sets `skate_right`/`skate_left` flags | Sends `SliderChanged` messages |
| Movement | Continuous (every frame) | Stepped (one position per interval) |
| Image loading | Incremental cache-aware | Direct jump, async preview |
| Typical Image FPS | ~200+ | ~20 |

## Implementation

### Files Modified

| File | Changes |
|------|---------|
| `src/replay.rs` | Added `NavigationMode` enum, slider tracking, `SliderNavigate` action |
| `src/main.rs` | Added `--nav-mode` and `--slider-step` CLI args |
| `src/app/replay_handlers.rs` | Handle `SliderNavigate` action, slider mode logic |
| `src/app/message_handlers.rs` | Call `set_image_count()` after LoadPos completes |

### 1. NavigationMode Enum (`src/replay.rs`)

```rust
#[derive(Debug, Clone, PartialEq, Default)]
pub enum NavigationMode {
    #[default]
    Keyboard,
    Slider,
}
```

### 2. ReplayConfig Updates (`src/replay.rs`)

```rust
pub struct ReplayConfig {
    // ... existing fields ...
    pub navigation_mode: NavigationMode,
    pub slider_step: u16,  // Step size for slider mode (default: 1)
}
```

### 3. ReplayController Slider Tracking (`src/replay.rs`)

```rust
pub struct ReplayController {
    // ... existing fields ...
    pub current_slider_position: u16,
    pub max_slider_position: u16,  // Set when directory loads
}
```

### 4. New ReplayAction Variants (`src/replay.rs`)

```rust
pub enum ReplayAction {
    // ... existing variants ...
    SliderNavigate { position: u16 },
    SliderStartNavigatingLeft,
}
```

### 5. Slider Navigation Logic (`src/replay.rs`)

```rust
fn navigate_slider_right(&mut self) -> Option<ReplayAction> {
    if self.current_slider_position >= self.max_slider_position {
        self.set_at_boundary(true);
        return None;
    }
    let new_pos = (self.current_slider_position + self.config.slider_step)
        .min(self.max_slider_position);
    self.current_slider_position = new_pos;
    self.on_navigation_performed();
    Some(ReplayAction::SliderNavigate { position: new_pos })
}

fn navigate_slider_left(&mut self) -> Option<ReplayAction> {
    if self.current_slider_position == 0 {
        self.set_at_boundary(true);
        return None;
    }
    let new_pos = self.current_slider_position.saturating_sub(self.config.slider_step);
    self.current_slider_position = new_pos;
    self.on_navigation_performed();
    Some(ReplayAction::SliderNavigate { position: new_pos })
}
```

### 6. CLI Arguments (`src/main.rs`)

```rust
#[arg(long, default_value = "keyboard")]
nav_mode: String,  // "keyboard" or "slider"

#[arg(long, default_value = "1")]
slider_step: u16,
```

### 7. Action Processing (`src/app/replay_handlers.rs`)

```rust
crate::replay::ReplayAction::SliderNavigate { position } => {
    debug!("Slider navigate to position {}", position);
    Some(Task::done(Message::SliderChanged(-1, position)))
}
```

## Speed Control

Navigation speed is controlled by two parameters:

### 1. `--nav-interval` (milliseconds between navigations)

Controls how frequently navigation actions are triggered.

- Default: `50` ms
- Lower = faster navigation (more actions per second)
- Higher = slower navigation (fewer actions per second)

**Formula:** `navigations_per_second = 1000 / nav_interval`

| nav_interval | Navigations/sec |
|--------------|-----------------|
| 50ms (default) | 20 |
| 100ms | 10 |
| 25ms | 40 |

### 2. `--slider-step` (images per navigation, slider mode only)

Controls how many images to skip per navigation action in slider mode.

- Default: `1` (single image per step)
- Higher = faster traversal (jumps multiple images)

**Formula:** `images_per_second = (1000 / nav_interval) * slider_step`

| nav_interval | slider_step | Images/sec |
|--------------|-------------|------------|
| 50ms | 1 | 20 |
| 50ms | 5 | 100 |
| 100ms | 10 | 100 |

### Examples

```bash
# Default speed (20 images/sec in slider mode)
cargo run --profile opt-dev -- --replay --test-dir /path --nav-mode slider

# Faster: 40 images/sec (shorter interval)
cargo run --profile opt-dev -- --replay --test-dir /path --nav-mode slider --nav-interval 25

# Faster: 100 images/sec (larger step)
cargo run --profile opt-dev -- --replay --test-dir /path --nav-mode slider --slider-step 5

# Slower: 10 images/sec
cargo run --profile opt-dev -- --replay --test-dir /path --nav-mode slider --nav-interval 100
```

### Keyboard Mode Speed

In keyboard mode, `--nav-interval` controls how often the skating flag is checked, but actual image loading is continuous (every frame). The `--slider-step` parameter has no effect in keyboard mode.

## Usage

```bash
# Keyboard mode (default)
cargo run --profile opt-dev -- --replay --test-dir /path --duration 10

# Slider mode
cargo run --profile opt-dev -- --replay --test-dir /path --duration 10 --nav-mode slider

# Slider with larger steps (faster traversal)
cargo run --profile opt-dev -- --replay --test-dir /path --duration 10 --nav-mode slider --slider-step 5

# Compare both modes
cargo run --profile opt-dev -- --replay --test-dir /path --nav-mode keyboard --output kb.json --output-format json --auto-exit
cargo run --profile opt-dev -- --replay --test-dir /path --nav-mode slider --output sl.json --output-format json --auto-exit
```

## Notes

- Slider mode does NOT trigger `SliderReleased` - it simulates continuous drag
- Boundary detection in slider mode uses position tracking (not pane index checking)
- `set_image_count()` is called after LoadPos completes to initialize `max_slider_position`
- In slider mode, `skate_right`/`skate_left` flags are not used

## Commit

```
6897cf3 Add slider navigation mode for replay benchmarking
```
