# 002 — Dual Camera Setup

**Date**: 2026-02-14

## Problem

The wrist cam (`/dev/video0`, InnoMaker RGB) can see the cube from the reset position but not the bowl on the right side. Moving the bowl left would be unrealistic — real pick-and-place tasks involve some distance between source and target.

## Solution

Added a second overhead webcam (`/dev/video2`) mounted at a high angle that captures the full scene: arm, cube, and bowl.

## Camera config

| Camera | Device | Key in dataset | Role |
|--------|--------|---------------|------|
| InnoMaker RGB (wrist) | `/dev/video0` | `observation.images.wrist` | Close-up manipulation view |
| Overhead webcam | `/dev/video2` | `observation.images.overhead` | Full scene context |

Both cameras: 640x480 @ 30fps.

## SmolVLA multi-camera support

SmolVLA handles multiple cameras out of the box:

- Automatically discovers all `observation.images.*` features in the dataset
- Each camera image is independently embedded through the SigLIP vision encoder
- Embeddings are concatenated along the sequence dimension with optional start/end tokens
- Cross-attention lets the model learn relationships between camera views
- No code changes needed — just add cameras to the robot config

The `empty_cameras` config option also exists for padding when some cameras are missing (used by aloha_sim with wrist cameras).

## Changes

- `scripts/record.py`: renamed `FRONT_CAM` → `WRIST_CAM`, added `OVERHEAD_CAM`, camera keys changed from `front` to `wrist` + `overhead`

## Note

Dataset `observation.images.front` key from earlier test recordings is now `observation.images.wrist`. Any old test datasets are incompatible — delete and re-record.
