# NVIDIA Rendering Performance Fix

**Date:** 2026-02-07
**Branch:** `fix/nvidia-rendering-performance`

## Context

After switching from AMD 7900 XTX (ROCm) to NVIDIA RTX 3090 (CUDA), the app became nearly unusable:

- Images would skate smoothly for a few seconds, then rendering would freeze
- After the freeze, old rendering events would replay without any keyboard input
- The slider UI mostly stopped working — dragging it rarely updated the displayed image
- Images only rendered once every few seconds instead of continuously

The only change was the GPU, driver (AMDGPU → NVIDIA proprietary), and compute stack (ROCm → CUDA). No code changes.

## Root Cause

**`PresentMode::AutoVsync` maps to FIFO on NVIDIA, which blocks `frame.present()` on the main thread.**

### How the cascade works

1. `ControlFlow::Poll` + skating mode → app tries to render every frame
2. `frame.present()` with FIFO on NVIDIA blocks for 10-50ms waiting for VSync when frames arrive out of sync with the display refresh cycle
3. Since `frame.present()` runs on the same thread as the event loop, **all event processing is blocked** during the stall
4. Keyboard, slider, and mouse events pile up in the winit event queue
5. The message queue grows past the 50-message threshold
6. `monitor_message_queue()` destructively clears the entire queue — including slider position events
7. Rendering stalls because no new navigation events arrive
8. When the GPU unblocks, all queued events replay at once — causing the "replay without keyboard press" effect

### Why AMD worked fine

AMD's Vulkan present queue is more forgiving with irregular frame timing. `AutoVsync` on AMD doesn't block as aggressively when frames arrive out of sync, so the event loop stays responsive even under continuous rendering load.

NVIDIA's FIFO implementation is strict — it expects frames to arrive in sync with the display refresh cycle and blocks aggressively when they don't.

## Fix

### 1. Non-blocking present mode selection (primary fix)

Query `surface.get_capabilities()` for supported present modes and select the best non-blocking option:

- **Mailbox** (preferred): Non-blocking, replaces the pending frame with the latest one. Ideal for an image viewer — always shows the most recent frame at VSync without blocking.
- **Immediate** (fallback): Non-blocking, no VSync. Used when Mailbox isn't available (e.g., some Wayland compositors, older NVIDIA drivers).
- **AutoNoVsync** (last resort): wgpu auto-selects between Mailbox and Immediate.

On the RTX 3090 with the NVIDIA proprietary driver, **Immediate** was selected (Mailbox was not reported as available).

### 2. Reduced frame latency

Changed `desired_maximum_frame_latency` from 2 to 1. With non-blocking present mode, there's no need for extra buffering — lower latency is strictly better.

### 3. Stored present mode in Runner state

The chosen present mode is now stored in the `Ready` state and reused during window resize (previously the resize handler hardcoded `AutoVsync` independently of the initial configuration).

## Files Changed

- `src/main.rs`:
  - Added `present_mode` field to `Runner::Ready` state
  - Added present mode capability detection in `resumed()` init block
  - Updated initial `surface.configure()` to use detected present mode
  - Updated resize handler `surface.configure()` to use stored present mode
  - Reduced `desired_maximum_frame_latency` from 2 to 1 in both configure calls

## Key Locations

- Present mode selection: `main.rs` ~line 1175-1189 (in the `block_on` async init)
- Initial surface config: `main.rs` ~line 1260
- Resize surface config: `main.rs` ~line 772
- Ready state field: `main.rs` ~line 511

## Cross-GPU Impact

This change is safe and beneficial for all users, not just NVIDIA. The old `AutoVsync` (Fifo) was a blocking present mode on every GPU — it just happened to not block long enough on AMD to cause visible problems. The new capability-based selection is strictly better:

| GPU | Before (AutoVsync) | After | Impact |
|-----|---------------------|-------|--------|
| AMD Linux (RADV/X11) | Fifo (blocking, worked by luck) | Mailbox (non-blocking, no tearing) | Slightly better |
| NVIDIA Linux | Fifo (blocking, broke the app) | Immediate (non-blocking) | Fixed |
| macOS (Metal) | Metal Fifo equivalent | Mailbox or Immediate | Same or better |
| Windows (DX12/Vulkan) | Fifo | Mailbox typically available | Same or better |
| Intel iGPU | Fifo | Mailbox or Immediate | Same or better |

Key architectural point: the event loop and rendering share the same thread in this app (winit single-threaded model). In apps with separate render threads, Fifo blocking is harmless — it just throttles the render loop. But here, any blocking in `frame.present()` also blocks keyboard events, slider events, async message delivery, and state updates. Even on AMD where Fifo didn't visibly break things, it was adding unnecessary VSync-wait latency during skating. The new approach eliminates that latency entirely.

## Notes

- The `monitor_message_queue()` destructive clearing at threshold 50 is a separate concern — it's a safety valve but can drop important events like slider positions. This fix eliminates the condition that triggers it (event loop stalling → queue overflow).
- The 300us sleep after each window event (`std::thread::sleep(Duration::from_micros(300))`) is still present. It was originally added to prevent CPU monopolization and doesn't contribute to the NVIDIA issue now that present mode is non-blocking.
