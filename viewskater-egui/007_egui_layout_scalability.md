# 007: egui Layout Scalability — Reflections on N-Pane Layout

**Date:** 2026-03-08

## Context

After implementing dual pane mode with a draggable divider (~60 lines total vs ~400 for the iced `Split` fork), reflecting on whether egui's layout approach scales to more complex layouts and what the fundamental tradeoffs are.

## How egui layout works

egui is immediate mode: every frame, your `update()` function runs and describes the entire UI from scratch. There is no persistent widget tree or retained layout state managed by the framework.

For standard layouts, egui provides helpers — `TopBottomPanel`, `SidePanel`, `CentralPanel`, `ui.horizontal()`, `ui.columns()` — that internally compute rects for you and allocate space within the current `Ui` region.

For custom layouts, you drop down to `allocate_new_ui(UiBuilder::new().max_rect(rect), |ui| ...)`, which says: "here is a rect in window coordinates — give me a fresh `Ui` scoped to this region and I'll draw whatever I want inside it." You compute the rects yourself with arithmetic. This is what we used for the dual pane split: compute `left_rect` and `right_rect` from `available_rect_before_wrap()`, fraction, and divider width, then hand each rect to a pane's `show_content()`.

Every widget call (buttons, labels, images, even the divider's invisible grab area) goes through `allocate_rect()` or similar, which reserves a portion of the current `Ui`'s available space. The painter draws into absolute screen coordinates. There is no layout pass separate from rendering — allocation and drawing happen in the same traversal.

## What scales well to N panes

- **Rect arithmetic is trivial to generalize**: divide available width by N, loop over N rects. The draggable divider is just `allocate_rect` + `drag_delta()` — loop over N-1 of them.
- **`allocate_new_ui` creates isolated sub-regions**: each pane gets its own `Ui` with independent input routing. Zoom, pan, drag all work per-pane without interference.
- **Borrow checker**: `split_at_mut` works for 2 panes. For N, you'd iterate with indices or use scoped borrows. No fundamental complexity increase.
- **Synced navigation**: the `all()` gate + `fold()` advance pattern already works for any N.

## Potential challenges at scale

1. **N-1 draggable dividers**: with 2 panes it's one fraction. With N, dragging divider K needs to decide: resize panes K and K+1 only? Or push all subsequent dividers? This is an interaction design problem, not an egui problem, but you'd implement the logic yourself.

2. **Nested splits (tiling WM-style)**: horizontal + vertical splits would require a layout tree data structure. egui won't build or manage this for you — you'd implement a tree of `SplitNode::Horizontal(fraction, Box<Node>, Box<Node>)` and recurse, computing rects at each level. Doable, but it's your code.

3. **Input routing / z-ordering**: `allocate_new_ui` sub-regions handle input cleanly. But floating overlays, popups, or context menus inside panes would need `egui::Area` or `egui::Window` for proper z-ordering. egui's layering is limited compared to retained-mode frameworks.

4. **Cache memory**: each pane has an 11-slot texture cache. At N=4 with 4K images, that's ~44 decoded RGBA textures in GPU memory. This is a resource problem, not a layout problem, but it's the first practical limit at scale.

5. **Synced navigation throughput**: the `all()` gate waits for the slowest pane's cache. With more panes, the probability all caches are simultaneously ready drops, reducing navigation FPS.

## The spectrum: egui vs iced

egui and iced sit at opposite extremes of a spectrum, and the ideal state for a continuous-rendering app with complex layout is somewhere in the middle.

**iced's position**: strict widget tree + message passing + event-driven architecture. The framework manages layout, diffing, and rendering. This gives you structured layouts for free, but continuous rendering (image viewer, video playback) fights the architecture — you write custom event loops, manage `LoadOperation` queues, coordinate per-pane `LoadingStatus` flags, and route messages through a `panes_to_load` filter. The 400-line `Split` widget fork with custom `Renderer` trait impls exists because iced's widget tree is the only way to define layout.

**egui's position**: immediate mode, no widget tree, render everything every frame. You get total freedom for continuous rendering — display an image, handle input, update state, all in one function. But for layout, you're on your own. The framework gives you `allocate_new_ui` as an escape hatch and some panel helpers, but anything beyond simple linear layouts means computing rects manually. Nested tiling splits would require building your own layout tree.

**The tradeoff**: in iced, the framework gives you layout structure but you fight it for rendering freedom. In egui, the framework gives you rendering freedom but you'd need to build layout structure yourself. Both approaches converge toward the same middle ground — you end up writing the part the framework doesn't provide.

For viewskater's use case (continuous image rendering with simple split layouts), egui's side of the spectrum is clearly more favorable. The layout is simple enough that manual rect arithmetic is trivial, and the rendering freedom eliminates entire categories of complexity (message passing, loading queues, widget tree diffing). The calculus could shift for an app with deeply nested, resizable, dockable panel layouts — at that point you'd be rebuilding what iced (or a docking library) gives you for free.

The key insight: the complexity isn't eliminated, it's relocated. iced makes you pay for rendering in a layout-first world. egui makes you pay for layout in a rendering-first world. Pick whichever tax is cheaper for your specific app.
