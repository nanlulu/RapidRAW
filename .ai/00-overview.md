# 00 · Overview

## What RapidRAW is

A cross-platform (Windows, macOS, Linux, Android) **RAW image editor**. It loads camera RAW
and standard image formats, lets the user apply non-destructive adjustments (exposure, color,
curves, masks, AI masks, lens correction, crop/transform, effects), and exports the result.
Performance is the headline goal: the processing pipeline runs on the GPU via **wgpu/WGSL**.

- Author: Timon Käch. License: **AGPL-3.0**. Site: getrapidraw.com.
- Desktop shell: **Tauri 2** (Rust core + system webview, not Electron). Bundle is < 20MB.

## Tech stack

| Layer | Tech | Where |
|---|---|---|
| UI | React 19, TypeScript, Tailwind v4, framer-motion, Konva (canvas overlays) | `src/` |
| State | **Zustand** (5 stores) | `src/store/` |
| i18n | i18next (10 locales) | `src/i18n/` |
| Shell / IPC | Tauri 2 (`@tauri-apps/api`) | `src-tauri/`, `src/` via `invoke`/`listen` |
| Core logic | Rust (edition 2024, rustc ≥ 1.95) | `src-tauri/src/` |
| GPU | **wgpu 29** + WGSL compute shaders | `src-tauri/src/gpu_processing.rs`, `src-tauri/src/shaders/` |
| RAW decode | `rawler` (custom fork: `RapidRAW-DngLab`) | `src-tauri/src/raw_processing.rs`, `image_loader.rs` |
| AI / ML | `ort` (ONNX Runtime) — SAM, U²-Net, Depth-Anything-v2, NIND denoise, LaMa, CLIP | `src-tauri/src/ai_*.rs`, `mask_generation.rs`, `tagging*.rs` |
| Parallelism | `rayon`, `tokio` | backend throughout |

Frontend build = **Vite**. The two are launched together by the Tauri CLI (`npm start`).

## The one mental model that matters

Everything an agent does touches one of these four flows. Internalize them:

```
                        ┌───────────────────────── FRONTEND (src/, React + Zustand) ─────────────────────────┐
  user moves a slider →  ControlsPanel → setAdjustments() → useEditorStore.adjustments (SOURCE OF TRUTH)
                              │                                        │
                              │ (Adjustments object = all edits)       │ debounced → pushHistory() (undo/redo)
                              ▼                                        ▼
                        useImageProcessing hook  ───────────── debouncedSave → .rrdata sidecar (persist)
                              │ invoke("apply_adjustments", {jsAdjustments, isInteractive, roi, ...})
  ════════════════════════════╪══════════════ Tauri IPC boundary ══════════════════════════════════════════
                              ▼
                        ┌───────────────────────── BACKEND (src-tauri/src/, Rust) ──────────────────────────┐
                        apply_adjustments (lib.rs) → enqueue PreviewJob → single preview-worker thread
                              │                                          │
                              ▼                                          ▼
                        load/cache decoded image      build AllAdjustments uniform (bytemuck)
                        apply geometry/crop (CPU)             │
                              │                               ▼
                              └──────────────► GpuProcessor → shader.wgsl (compute) → output texture
                                                                      │
                          return JPEG bytes (invoke result)  ◄────────┤
                          emit "preview-update-uncropped",            │
                               "histogram-update", "waveform-update" ─┘ (async, via app_handle.emit)
                              ▲
  ════════════════════════════╪══════════════ Tauri IPC boundary ══════════════════════════════════════════
                              │ listen(event) in useTauriListeners
                        canvas re-renders ← finalPreviewUrl / WGPU direct render
```

**The four flows, named:**

1. **Edit → preview** (above): the hot loop. Slider/tool → `Adjustments` → `apply_adjustments`
   → GPU → preview back to canvas. Interactive (dragging) requests are smaller/faster; settled
   requests render full resolution. Stale requests are superseded by job IDs.
2. **Persist** — `Adjustments` is debounced-saved to a per-image `.rrdata` JSON sidecar
   (see [glossary](glossary.md#sidecar)); reloaded and merged with `INITIAL_ADJUSTMENTS` on open.
3. **Library** — folder tree, thumbnails (worker pool), ratings/labels/tags, albums, filtering,
   virtual copies. Mostly `file_management.rs` ↔ `useLibraryStore`.
4. **Heavy ops** — export, batch, HDR merge, panorama stitch, denoise, AI mask generation,
   generative replace. Long-running; report progress via emitted events and modals.

## Where to start for a given task

| You're asked to… | Start in | Recipe |
|---|---|---|
| Add/modify an adjustment slider | `src/utils/adjustments.ts` + `shader.wgsl` | [04 §1](04-recipes.md) |
| Add a backend command | `src-tauri/src/lib.rs` invoke_handler + `AppProperties.tsx` Invokes | [04 §2](04-recipes.md) |
| Touch the GPU math | `src-tauri/src/shaders/shader.wgsl` + `gpu_processing.rs` | [04 §3](04-recipes.md) |
| Add a mask type | `src/.../Masks.tsx` + `mask_generation.rs` | [04 §4](04-recipes.md) |
| Library / files / thumbnails | `file_management.rs` + `useLibraryStore` | [02](02-codemap.md) |
| AI / ONNX models | `ai_processing.rs`, `ai_commands.rs`, `build.rs` | [02](02-codemap.md) |
| UI text / translations | `src/i18n/locales/*.json` | [03](03-conventions.md) |

## Verification

- Stack/deps: `package.json`, `src-tauri/Cargo.toml`.
- Flow diagram: traced through `src/hooks/useImageProcessing.ts`, `src/hooks/useTauriListeners.ts`,
  `src-tauri/src/lib.rs` (`apply_adjustments`, preview-worker, `emit` calls at lib.rs:629/641/843).
