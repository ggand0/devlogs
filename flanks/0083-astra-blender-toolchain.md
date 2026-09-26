# 0083: Astra and Blender toolchain, set up and measured (2026-09-22)

Setup notes and their execution: docs/internal/001-astra-blender-setup.md. Asset contract: docs/plans/009-unit-asset-spec.md. No engine code touched. Number 0082 is reserved by the item 2 GPU thread, so this entry takes 0083.

## State

Steps 1 to 5 and 7 of the setup doc are done on this box. Step 6, the Start MCP Server click inside Blender, is the owner's and is the only thing left before a first Astra session.

- Codex 0.155.1 pinned to `gpt-6-astra` at high reasoning effort in `~/.codex/config.toml`. `codex doctor`: config parse ok, ChatGPT auth ok, provider websocket ok.
- MCP for Blender 1.30.0 registered as the `blender` stdio server. A raw stdio handshake returned its 32 tools, so the whole chain up to Blender's own socket is proven without the GUI.
- The add-on is installed into `~/.config/blender/5.2/scripts/addons` (its installer does not know about 5.2, `--addons-dir` is required), enabled, and saved into user preferences headlessly. Its command handlers were called directly against Blender 5.2: `get_scene_info`, `get_object_info`, `execute_code` all behave.
- Telemetry turned off at the server through `DISABLE_TELEMETRY`. It is on by default and uploads to a Supabase the add-on author runs. Its in-Blender consent tick, off by default, additionally gates prompts, code, scene info and viewport screenshots.

## The headless route, and what it turned up

`assets_dev/_setup_smoke/` now holds a reference that builds a stand-in soldier from boxes, exports a GLB, re-imports it into an empty scene, reads the shipped attribute table, and renders the four check sizes. It runs green end to end:

```
blender --background --factory-startup --python-exit-code 1 \
  --python assets_dev/_setup_smoke/smoke_route.py -- --out assets_dev/_setup_smoke/out
```

Three findings that would each have cost an Astra session, all read out of the GLB bytes rather than trusted to Blender's importer:

1. **Vertex groups never reach the GLB.** glTF carries them only as skin weights, and stage 1 has no armature, so the part marking the brief asks for silently disappears. The part id has to be baked into a second UV layer before export, which arrives as `TEXCOORD_1` with unclamped float values. A custom `_part` attribute with `export_attributes=True` works too. UV2 is what the engine plan already assumed, so the spec now fixes a part id order and requires the bake.
2. **Vertex colour alpha is dropped by default.** The exporter writes `COLOR_0` as VEC3 with stock settings, which throws away the team amount, the one channel the whole team colour scheme depends on. `export_vertex_color="ACTIVE"` gives VEC4 with alpha intact and leaves the material opaque. Wiring the colour node's Alpha into Principled Alpha also works but marks the material `alphaMode: BLEND`, wrong for a soldier.
3. **No display is needed for renders.** Blender 5.2 offers exactly one engine, `BLENDER_EEVEE`, and it renders in background mode. No Xvfb, no Cycles fallback, so check images cost about 0.1 s each.

Also measured: pivot empties survive as glTF nodes with their translations; the exporter triangulates on its own; and `export_yup=True` means the spec's "facing +Z" is authored as facing -Y in Blender.

## Lesson

The setup doc's own rule paid for itself immediately: renders are not proof. Both silent channel losses look perfect in a render and in Blender's re-import (the importer helpfully rebuilds an alpha of 1.0 and a second UV map), and only the GLB accessor table shows what actually shipped. `glb_inspect.py` exists for that and takes a second to run.

## Next

Owner clicks Start MCP Server, then `python3 assets_dev/_setup_smoke/socket_check.py` confirms the add-on half without Codex. First Astra session is the knight, stage 1, L0 only, per the setup doc's working method.
