# 02 · Code Map — "where does X live?"

File:line anchors verified on 2026-06-08 (`main`). Line numbers drift; if one is off by a few,
grep the named symbol. Anchors point at the **entry point**, not the only relevant line.

## Top-level layout

```
RapidRAW/
├── src/                     # React/TS frontend (UI + edit state)
├── src-tauri/               # Rust backend (Tauri shell + processing)
│   ├── src/                 # Rust source
│   │   ├── shaders/*.wgsl   # GPU image math
│   │   ├── panorama_utils/  # panorama submodule
│   │   └── tagging_utils/   # auto-tagging submodule
│   ├── lensfun_db/          # lens-correction database
│   ├── resources/           # bundled resources + license texts
│   ├── capabilities/        # Tauri permission manifests
│   ├── Cargo.toml           # Rust deps + release profile (lto, codegen-units=1)
│   ├── build.rs             # downloads & verifies the ONNX Runtime lib at build time
│   └── tauri.conf.json      # Tauri app config (window, bundle, plugins)
├── public/                  # static frontend assets
├── data/                    # bundled data
├── packaging/               # platform packaging (Flatpak, etc.)
├── package.json             # frontend scripts + deps
├── vite.config.js           # Vite config
└── .github/workflows/       # CI (lint, ci, pr-ci, build, release)
```

## Frontend (`src/`)

| Concern | File | Anchor |
|---|---|---|
| App orchestrator / wiring | `App.tsx` | `function App()` ~`:83` |
| **Adjustments data model** | `utils/adjustments.ts` | `interface Adjustments` `:149`; `INITIAL_ADJUSTMENTS` `:475`; `normalizeLoadedAdjustments` `:~591` |
| **Command name registry (Invokes)** | `components/ui/AppProperties.tsx` | `enum Invokes` `:32` |
| Editor store (edits + history) | `store/useEditorStore.ts` | `create<EditorState>` `:83`; history methods `:133`–`:172` |
| Library store | `store/useLibraryStore.ts` | |
| Process store (export/thumbs) | `store/useProcessStore.ts` | |
| Settings store | `store/useSettingsStore.ts` | `handleSettingsChange` `:~41` |
| UI store (panels/modals) | `store/useUIStore.ts` | `setRightPanel` `:~199` |
| **Preview/render loop** | `hooks/useImageProcessing.ts` | `invoke(ApplyAdjustments)` `:174` |
| Image load lifecycle | `hooks/useImageLoader.ts` | |
| **Backend event listeners** | `hooks/useTauriListeners.ts` | `listen(...)` block `:~73` |
| Edit actions (set/copy/paste/reset, debounced save) | `hooks/useEditorActions.ts` | `setAdjustments` `:~35`; `debouncedSave` |
| Navigation (library↔editor, selection) | `hooks/useAppNavigation.ts` | |
| File ops (CRUD, import, delete) | `hooks/useFileOperations.ts` | |
| Library actions (rating/label/album) | `hooks/useLibraryActions.ts` | |
| Heavy workflows (HDR/pano/denoise/collage) | `hooks/useProductivityActions.ts` | |
| AI masking | `hooks/useAiMasking.ts` | |
| Keyboard shortcuts | `hooks/useKeyboardShortcuts.ts` | |
| Thumbnails | `hooks/useThumbnails.ts` | |
| Presets | `hooks/usePresets.ts` | |
| Canvas (zoom/pan/overlays, Konva) | `components/panel/editor/ImageCanvas.tsx` | largest FE file (~2600) |
| Right-panel switcher | `components/panel/right/RightPanelSwitcher.tsx` | |
| Adjustments panel host | `components/panel/right/ControlsPanel.tsx` | |
| Adjustment sections | `components/adjustments/{Basic,Color,Curves,Details,Effects}.tsx` | |
| Masks UI + types | `components/panel/right/MasksPanel.tsx`, `Masks.tsx` | `enum Mask`, `SubMask` in `Masks.tsx` |
| AI patches UI | `components/panel/right/AIPanel.tsx` | |
| Crop UI | `components/panel/right/CropPanel.tsx` | `OverlayMode` |
| Export UI | `components/panel/right/ExportPanel.tsx` | |
| Metadata/EXIF UI | `components/panel/right/MetadataPanel.tsx` | |
| Presets UI | `components/panel/right/PresetsPanel.tsx` | |
| Library grid/list | `components/panel/library/*`, `components/panel/MainLibrary.tsx` | |
| Folder tree | `components/panel/FolderTree.tsx` | |
| Filmstrip | `components/panel/Filmstrip.tsx` | |
| Settings screen | `components/panel/SettingsPanel.tsx` | ~2400 lines |
| Modals (HDR/pano/denoise/collage/lens/…) | `components/modals/*.tsx`, aggregated by `AppModals.tsx` | |
| Reusable UI | `components/ui/*` (Slider, Dropdown, Switch, ColorWheel, …) | |
| Image LRU cache (FE) | `utils/ImageLRUCache.ts` | |
| Crop/mask/keyboard/theme utils | `utils/{cropUtils,maskUtils,keyboardUtils,themes}.ts` | |
| i18n setup + locales | `i18n/index.ts`, `i18n/locales/*.json` | en is the source-of-truth locale |

## Backend (`src-tauri/src/`)

| Concern | File | Anchor |
|---|---|---|
| **App bootstrap + command registry** | `lib.rs` | `generate_handler![` `:~2222`; `run()` |
| `apply_adjustments` command | `lib.rs` | `:~686` |
| Preview & analytics worker threads | `lib.rs` | worker spawns + `emit` `:629`/`:641`/`:843` |
| Other preview commands | `lib.rs` | `generate_uncropped_preview`, `generate_original_transformed_preview`, `preview_geometry_transform`, `generate_preset_preview` |
| **Shared state + caches** | `app_state.rs` | `struct AppState` `:109` |
| JSON→uniform bridge, `GpuContext` | `image_processing.rs` | `AllAdjustments` `:~1396`; auto-adjust `calculate_auto_adjustments` |
| wgpu pipelines / `GpuProcessor` | `gpu_processing.rs` | `struct GpuProcessor`; pipeline init |
| **Main image shader** | `shaders/shader.wgsl` | the pixel math (≈66KB) |
| Blur / flare / display shaders | `shaders/{blur,flare,display}.wgsl` | |
| Cache hashing | `cache_utils.rs` | `calculate_transform_hash`, `calculate_geometry_hash`, `calculate_visual_hash` |
| RAW decode | `raw_processing.rs`, `image_loader.rs` | `load_image` command in `image_loader.rs` |
| Format detection | `formats.rs` | `is_raw_file` |
| **Files/thumbs/sidecars/albums** | `file_management.rs` | `parse_virtual_path` `:165`; `.rrdata` writers `:~176`/`:185`; thumbnail workers; many commands |
| EXIF + sidecar read/write | `exif_processing.rs` | `get_primary_sidecar_path` (`.rrdata`) `:~1074`; `get_rrexif_path` (legacy `.rrexif`) `:~1081` |
| Export (single/batch/estimate/cancel) | `export_processing.rs` | `export_images`, `cancel_export`, `estimate_export_sizes` |
| AI model load/inference | `ai_processing.rs` | `get_or_init_*` model loaders; `AiState` |
| AI Tauri commands | `ai_commands.rs` | `generate_ai_{subject,foreground,sky,depth}_mask`, `invoke_generative_replace_with_mask_def` |
| Cloud AI connector | `ai_connector.rs` | `check_ai_connector_status` |
| Mask rasterization | `mask_generation.rs` | `generate_mask_bitmap`, `generate_mask_overlay` |
| Auto-tagging + indexing | `tagging.rs`, `tagging_utils/` | `start_background_indexing` |
| Denoise | `denoising.rs` | `apply_denoising`, `batch_denoise_images` |
| Panorama | `panorama_stitching.rs`, `panorama_utils/` | `stitch_panorama` |
| HDR merge | `lib.rs` | `merge_hdr`, `save_hdr` |
| Negative conversion | `negative_conversion.rs` | `convert_negatives` |
| LUT load/parse | `lut_processing.rs` | `load_and_parse_lut`; `struct Lut` |
| Lens correction | `lens_correction.rs` | `autodetect_lens`, `get_lens_distortion_params` |
| Culling (duplicate/blur detect) | `culling.rs` | `cull_images` |
| Settings persistence | `app_settings.rs` | `load_settings`, `save_settings` |
| Android glue | `android_integration.rs` | `resolve_android_content_uri_name` |
| Window customizing | `window_customizer.rs` | `PinchZoomDisablePlugin` |

## The two indexes you'll grep most

- **Add/trace a command**: name appears in `lib.rs` `generate_handler![…]` **and** in
  `AppProperties.tsx` `enum Invokes`. Both must agree (snake_case ↔ enum string value).
- **Add/trace a backend→frontend event**: `app_handle.emit("name", …)` in Rust ↔ `listen("name", …)`
  in `useTauriListeners.ts`.

## Verification

Counts/anchors from `find`, `wc -l`, and targeted `grep` over `src/` and `src-tauri/src/` on
2026-06-08; the registry/Invokes pairing confirmed against `lib.rs:2222–2320` and
`AppProperties.tsx:32+`.
