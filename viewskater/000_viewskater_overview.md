# ViewSkater Overview

**Type:** Project Overview Document

## Project Summary
ViewSkater is a fast, native image viewer app written in Rust using the Iced GUI library and integrated with wgpu for GPU acceleration. The app is designed for smooth browsing of large image directories, with a strong emphasis on rendering performance, caching efficiency, and multi-pane workflows. Its primary audience includes developers, researchers, and professionals working with large image datasets, technical drawings, and multi-format media.

The project started as a personal tool to handle large volumes of images quickly and has grown into an open-source project with community traction (100+ GitHub stars, user issues, PRs). A proprietary enterprise version is planned for future monetization. The app is being prepared for distribution on the Mac App Store, Microsoft Store, and possibly Gumroad or other platforms.

Key Features

GPU-accelerated rendering using wgpu

Continuous rendering at 50–60 fps with keyboard and slider navigation

Sliding window cache to preload next/previous images for instant navigation

Support for large images up to 8192×8192 px (auto-resizes oversized textures)

Keyboard navigation with arrow keys

Global slider for fast navigation across directories

Dual-pane mode with synced zooming, panning, and navigation (shared slider)

Drag-and-drop support for files and folders

File association and “Open With” support on macOS, Linux, and Windows

Supports common formats: JPG, PNG, BMP, TIFF, WebP, GIF, QOI, TGA, etc.

Experimental RAW support via libraw-rs (feature branch, not yet merged)

Multi-page document support (PDF/TIFF converted to PNGs)

Caching and Loading

Global image cache with async loading queue

Preloading of neighboring images for smooth slider movement

Batched loading operations (LoadPos, LoadNext, LoadPrevious, etc.) with multi-pane awareness

Optional experimental GPU atlas and tile-based rendering for large 4K images

Architecture and Implementation

Built with Iced 0.13.1 using the new theming and .style() APIs

Custom shader widget and canvas APIs for GPU-side image rendering

Event loop built on winit 0.30.x with a custom fix for macOS continuous rendering (selective VecDeque event batching)

Proxy integration with iced_runtime for async tasks

Pane struct for managing individual viewports with a dynamic number of panes

Global cache and loading queue for managing images across panes

Async image loading via Task::perform

Distribution and Packaging

macOS: pkg distribution for App Store, code-signed, notarized, sandboxed. Handles sandbox restrictions with security bookmarks and NSOpenPanel.

Linux: .desktop integration for “Open With”, CLI args for direct file launching.

Windows: registry-based file associations, installer tooling (NSIS or Inno Setup) under consideration.

Roadmap

Short-term: finalize winit 0.30.x patch, migrate to Iced 0.14 when stable, improve caching with GPU atlas or tile-based rendering, add COCO dataset visualization.

Medium-term: release on Mac App Store and Microsoft Store, add support for video, text, and 3D panes, improve RAW support behind feature flags.

Long-term: proprietary enterprise version with annotation and dataset management, possible fork into a full annotation tool for ML workflows.

Community and Reception

Shared on Reddit, gaining traction among Rust and image processing communities

Over 100 stars on GitHub with community issues and PR proposals (e.g. ZIP file browsing)

Active discussion on Iced Discord

Positioned as a lightweight alternative to heavy image viewers and editors, optimized for large datasets and technical workflows

### User's notes
I usually use the command `RUST_LOG=viewskater=debug cargo run --profile opt-dev`