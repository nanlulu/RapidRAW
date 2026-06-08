# 01 · Architecture

Two halves joined by Tauri IPC. This file explains each half and, in detail, the editing
pipeline that connects them — the part agents most often need to reason about.

---

## A. Frontend (`src/`)

React 19 + TypeScript. **No React Router**: the app is one window that switches between two
views based on whether an image is selected.

### Composition

- **`src/App.tsx`** (~770 lines) — the orchestrator. It is mostly *wiring*, not logic: it reads
  slices from the 5 stores, calls ~10 feature hooks, and renders either `LibraryView` or
  `EditorView` plus the resizable panels, modals, titlebar, and two headless "manager"
  components. Keep logic *out* of here; put it in a hook.
- **Views** (`src/components/views/`): `LibraryView` (folder tree + grid/list + community page)
  vs `EditorView` (canvas + right panel). Selected when `selectedImage` is null vs set.
- **Headless managers** (`src/components/managers/`): `ImageProcessingManager` and
  `ImageLoaderManager` render nothing — they are thin components wrapping `useImageProcessing`
  / `useImageLoader` so those effect-heavy hooks live at a stable point in the tree.
- **Feature hooks** (`src/hooks/`): each owns one concern — `useAppInitialization`,
  `useAppNavigation`, `useEditorActions`, `useFileOperations`, `useLibraryActions`,
  `useProductivityActions` (HDR/pano/denoise/collage), `useImageProcessing` (the preview loop),
  `useImageLoader`, `useThumbnails`, `useKeyboardShortcuts`, `useTauriListeners`, `usePresets`,
  `useAiMasking`. **When adding behavior, add/extend a hook — don't bloat App.tsx.**

### State: the 5 Zustand stores (`src/store/`)

| Store | Owns | Notes |
|---|---|---|
| **`useEditorStore`** | `adjustments` (current edits), `selectedImage`, undo/redo `history`+`historyIndex`, preview URLs, analytics (histogram/waveform), interaction state (zoom/sizes/dragging), active mask/AI-patch ids, clipboard | **Source of truth for the open image.** History methods: `pushHistory` (caps at 50, slices forward branch), `undo`, `redo`, `resetHistory`, `goToHistoryIndex`. |
| **`useLibraryStore`** | root paths, current folder, folder/album trees, `imageList`, ratings, multi-select, sort/filter/search criteria, scroll/column UI | |
| **`useProcessStore`** | export/import state, thumbnail+indexing progress, `thumbnails` (blob URLs), transient copied/pasted flags | |
| **`useSettingsStore`** | `appSettings`, `theme`, `supportedTypes`, `osPlatform`; `handleSettingsChange` persists via `save_settings` | |
| **`useUIStore`** | panel layout/widths, `activeRightPanel`, collapsible-section state, modal open/close flags | |

Data flow is unidirectional: components read store slices (via `useShallow` for multi-field
reads) and call store setters; effects/hooks react to changes.

### The adjustments data model — `src/utils/adjustments.ts`

The **`Adjustments`** interface is the central type (~60+ fields). Groups:

- **Basic tone**: exposure, contrast, highlights, shadows, whites, blacks, brightness.
- **Color**: temperature, tint, vibrance, saturation, HSL (`hue`/`saturation`/`luminance` per band),
  `colorGrading` (shadows/midtones/highlights wheels + balance/blending), `colorCalibration`.
- **Curves**: `curves`/`pointCurves` (point mode) + `parametricCurve`, switched by `curveMode`.
- **Details**: clarity, dehaze, structure, sharpness(+threshold), luma/color noise reduction,
  chromatic aberration.
- **Effects**: grain (amount/size/roughness), vignette, glow, halation, flare, **LUT**
  (`lutData`/`lutIntensity`/`lutName`/`lutPath`/`lutSize`).
- **Geometry**: `crop`, `aspectRatio`, `rotation`, flips, `orientationSteps`, and `transform*`
  (perspective/distortion) keys; **lens correction** (`lens*`) keys.
- **Masks**: `masks: MaskContainer[]` — each container has its own `MaskAdjustments` (a subset of
  the global adjustments) and a list of `SubMask` (Brush/Radial/Linear/Color/Luminance/AI*),
  combined by mode (Additive/Subtractive/Intersect).
- **AI patches**: `aiPatches: AiPatch[]` — generative-replace regions, each with its own subMasks
  + `patchData`.

`INITIAL_ADJUSTMENTS` is the zeroed baseline used on load and reset. `normalizeLoadedAdjustments()`
merges a loaded sidecar into the current schema (back-fills new fields, regenerates UUIDs) — the
**migration seam**: when you add a field, it gets a default here automatically via the spread.

### Right-panel editing UI

`RightPanelSwitcher` toggles one of: Metadata, Adjustments (`ControlsPanel`), Crop, Masks, AI,
Presets, Export. `ControlsPanel` hosts the collapsible sections backed by
`src/components/adjustments/{Basic,Color,Curves,Details,Effects}.tsx`. A slider's `onChange`
→ `setAdjustments(partial)` → store update (instant UI) + debounced history + the
`useImageProcessing` effect fires a new preview request.

---

## B. Backend (`src-tauri/src/`)

Rust, edition 2024. `main.rs` is a 5-line shim calling `rapidraw_lib::run()`; **`lib.rs`** is the
real entry: it builds the Tauri app, registers all `#[tauri::command]`s in one
`generate_handler![…]` block (~97 commands), spawns the worker threads, and manages global state.

### Modules (one responsibility each)

| Module | Responsibility |
|---|---|
| `lib.rs` | App bootstrap, command registry, **preview/analytics worker threads**, top-level preview commands (`apply_adjustments`, `generate_*_preview`). |
| `app_state.rs` | `AppState` — the shared `tauri::State`: the loaded image, GPU context/processor, AI state, and **every cache** (see below). |
| `image_processing.rs` | CPU↔GPU bridge: parse JSON adjustments → `AllAdjustments` uniform struct; `GpuContext`; orchestrate a render. |
| `gpu_processing.rs` | wgpu plumbing: `GpuProcessor`, pipelines (main compute, blur, flare), bind groups, uniform buffers. |
| `shaders/*.wgsl` | The actual image math. `shader.wgsl` (≈66KB) is the main compute pipeline. |
| `raw_processing.rs`, `image_loader.rs`, `formats.rs` | Decode RAW (`rawler`) and standard formats; format detection. |
| `file_management.rs` (≈3600 lines) | Folder tree, listing, **thumbnails**, sidecars (`.rrdata`), albums, virtual copies, ratings/labels, presets, import/move/copy/delete. |
| `exif_processing.rs` | EXIF read/write, the `.rrdata`/`.rrexif` sidecar read/write helpers, metadata caching. |
| `export_processing.rs` | Single + batch export, size estimation, cancellation. |
| `denoising.rs`, `panorama_stitching.rs`(+`panorama_utils/`), `negative_conversion.rs`, `lut_processing.rs`, `lens_correction.rs` | Heavy/standalone ops. |
| `ai_processing.rs`, `ai_commands.rs`, `ai_connector.rs`, `mask_generation.rs` | AI/ONNX inference + mask rasterization (see §D). |
| `tagging.rs`, `tagging_utils/` | CLIP-based auto-tagging + background indexing. |
| `android_integration.rs`, `window_customizer.rs` | Platform glue. |
| `app_settings.rs`, `cache_utils.rs`, `culling.rs`, `adjustment_utils.rs`, `preset_converter.rs` | Settings persistence, hashing, culling, helpers. |

### Concurrency model

- A **single preview-worker thread** (started in `lib.rs`) consumes `PreviewJob`s from a channel.
  `apply_adjustments` just enqueues; the worker coalesces/drops stale jobs so the canvas never
  falls behind the user. This is why previews come back partly as the invoke return value and
  partly as emitted events.
- A separate **analytics worker** computes histogram/waveform off the hot path and emits
  `histogram-update` / `waveform-update`.
- A **thumbnail worker pool** (`ThumbnailManager` queue + condvar) generates library thumbnails;
  `rayon` parallelizes CPU-bound per-pixel and batch work.

---

## C. The IPC contract (how the halves talk)

**Frontend → Backend**: `invoke("snake_case_command", { camelCaseArgs })`. Command names are
centralized in the **`Invokes` enum** at `src/components/ui/AppProperties.tsx`. Every entry must
match a function registered in the `generate_handler![…]` block in `lib.rs`. Tauri converts JS
camelCase arg keys to Rust snake_case parameter names automatically.

**Backend → Frontend**: `app_handle.emit("event-name", payload)`. The frontend subscribes in
**`src/hooks/useTauriListeners.ts`**. Known events the backend emits include:

| Event | Payload | Consumed for |
|---|---|---|
| `preview-update-uncropped` | data URL | the uncropped/transform preview overlay |
| `histogram-update` | histogram data | analytics panel |
| `waveform-update` | waveform data | waveform display |
| `thumbnail-generated` | path + thumb | library grid |
| `*-progress` (e.g. `hdr-progress`, export progress) | status string/number | modal progress |

> Grep both sides when wiring IPC: `grep -rn "\.emit(" src-tauri/src` and the `listen(` calls in
> `src/hooks/useTauriListeners.ts`. The set above is representative, not exhaustive.

---

## D. The editing pipeline, end to end (the part to really understand)

1. **Trigger.** User changes `Adjustments` (slider, mask edit, crop). `useImageProcessing`
   debounces and calls `invoke("apply_adjustments", { jsAdjustments, isInteractive, targetResolution, roi, computeWaveform, activeWaveformChannel })`.
2. **Enqueue.** `apply_adjustments` (lib.rs) wraps it in a `PreviewJob` and sends it to the
   preview-worker channel, returning a `oneshot` receiver. Older queued jobs are dropped.
3. **Hydrate + geometry (CPU).** The worker loads the decoded original from cache (decoding RAW
   on first touch), then applies **crop / rotation / flip / perspective / lens** transforms on the
   CPU, producing a transformed image. Results are cached by **`transform_hash`** so pure tonal
   tweaks skip re-warping.
4. **Build uniforms.** JSON adjustments → `AllAdjustments` (a `bytemuck`-`Pod` struct: global
   tonal/color params, per-band HSL, curve tables, color-grading, per-mask `MaskAdjustments`,
   tone-mapper selection, LUT intensity, …). Masks are rasterized to grayscale bitmaps
   (`mask_generation.rs`, cached) and uploaded as a texture array.
5. **GPU compute.** `GpuProcessor` binds input texture + uniforms + mask textures + optional 3D
   LUT texture and dispatches `shader.wgsl`. The shader applies tone → curves → color/HSL →
   color-grading → tone-mapping → local (masked) adjustments → blur-based effects
   (clarity/structure/dehaze) → vignette/grain → LUT, writing the output texture.
6. **Return.** For settled (non-interactive) requests the worker JPEG-encodes (mozjpeg) and
   returns bytes via the `oneshot`; the frontend turns them into `finalPreviewUrl`. Some paths
   instead drive a **direct wgpu window render** (the `"WGPU_RENDER"` signal +
   `update_wgpu_transform`) for zero-copy display. Analytics + uncropped overlays come back as
   emitted events.
7. **Persist.** Independently, `Adjustments` is debounced-saved to the `.rrdata` sidecar.

**Caches** (all in `AppState`, keyed/invalidated as noted — see [glossary](glossary.md)):
`decoded_image_cache` (by path, LRU), `full_warped_cache`/`full_transformed_cache`/`cached_preview`
(by geometry/transform hash), `gpu_image_cache`, `mask_cache`, `geometry_cache`,
`thumbnail_geometry_cache`, `lut_cache`, `patch_cache`. The hashing lives in `cache_utils.rs`:
`calculate_geometry_hash` (geometry-only keys), `calculate_transform_hash` (geometry + crop +
rotation/flip + patches), `calculate_visual_hash` (path + tonal/color adjustments). **Getting an
edit to invalidate the right cache is the most common subtle backend bug** — if a new field
changes pixels but the relevant hash ignores it, you'll see stale previews.

## Verification

- Frontend: `src/App.tsx`, `src/store/*.ts`, `src/utils/adjustments.ts`, `src/hooks/useImageProcessing.ts`,
  `src/hooks/useTauriListeners.ts`, `src/components/ui/AppProperties.tsx`.
- Backend: `src-tauri/src/lib.rs` (invoke_handler block ~line 2222; preview worker; `emit` calls),
  `app_state.rs` (`AppState` fields, lines 109–140), `cache_utils.rs`, `image_processing.rs`,
  `gpu_processing.rs`, `mask_generation.rs`.
