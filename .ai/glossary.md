# Glossary

Domain and codebase terms an agent needs. App-specific meanings, not generic definitions.

### Adjustments
The single object (`Adjustments` interface, `src/utils/adjustments.ts`) holding **all** edits for
the open image: tone, color, curves, details, effects, crop/geometry, lens, masks, AI patches.
It is the source of truth (`useEditorStore.adjustments`), the unit of undo/redo, copy/paste,
presets, and what gets serialized to the sidecar. `INITIAL_ADJUSTMENTS` is the zeroed baseline.

### `AllAdjustments`
The Rust `#[repr(C)] + bytemuck::Pod` mirror of (a subset of) `Adjustments`, built in
`image_processing.rs` and uploaded to the GPU as a uniform/storage buffer. Its field layout must
match the WGSL struct in `shader.wgsl` exactly.

### Sidecar (`.rrdata` / `.rrexif`)
Non-destructive edits are not written into the image file. They live in a **sidecar** next to it.
- **`.rrdata`** — primary sidecar: JSON `ImageMetadata` (`version`, `rating`, `adjustments`,
  `tags`, `exif`). This is what persists the user's edits. (`get_primary_sidecar_path`.)
- **`.rrexif`** — legacy EXIF sidecar, still read for back-compat. (`get_rrexif_path`.)
- Virtual copies use `{filename}.{6-hex-id}.rrdata`.

### Virtual copy
A second set of edits for the same source image, stored as its own `.rrdata` sidecar with a hex
id suffix and addressed by a "virtual path" (`path?vc=id`). Parsed by `parse_virtual_path`
(`file_management.rs`). Has its own thumbnail and metadata.

### Preview worker / `PreviewJob`
A single background thread (started in `lib.rs`) that serializes all preview renders.
`apply_adjustments` enqueues a `PreviewJob` and returns a `oneshot` receiver; the worker drops
stale jobs so the canvas keeps up with the user. The reason previews arrive both as the `invoke`
return value (settled, full-res JPEG) and as emitted events (uncropped overlay, analytics).

### Analytics worker
Separate thread computing histogram/waveform off the hot path, emitting `histogram-update` /
`waveform-update`.

### `transform_hash` / `geometry_hash` / `visual_hash`
Cache keys computed in `cache_utils.rs`:
- **geometry_hash** — only geometry/perspective/lens keys; keys the fully-warped image cache.
- **transform_hash** — geometry + crop + rotation/flip + patches; keys the transformed-preview
  caches.
- **visual_hash** — path + tonal/color adjustments; keys post-geometry visual caches/thumbnails.
Getting an edit to invalidate the *right* hash is the most common subtle backend correctness bug.

### Interactive vs settled preview
While the user drags (`isInteractive`/`isSliderDragging` true), the backend renders a smaller/ROI
preview for low latency; when interaction settles it renders full resolution. Controlled from
`useImageProcessing.ts`.

### `WGPU_RENDER` / direct render
Some preview paths skip JPEG round-tripping and drive a **direct wgpu render to the window**
(zero-copy). The backend signals this instead of returning bytes; `update_wgpu_transform` keeps
the on-screen transform in sync.

### MaskContainer / SubMask / mask modes
A `MaskContainer` is one local-adjustment layer: it has its own `MaskAdjustments` (a subset of the
global adjustments) plus a list of `SubMask`s combined into the final region. A `SubMask` has a
`type` (Brush, Radial, Linear, Color, Luminance, AiSubject, AiSky, AiForeground, AiDepth, …) and a
`mode` (Additive / Subtractive / Intersect). Rasterized to a grayscale bitmap in
`mask_generation.rs`, uploaded to the shader as a texture array.

### AiPatch / generative replace
An `AiPatch` is a region (its own subMasks) processed by the **LaMa** inpainting model to remove
or replace content (`invoke_generative_replace_with_mask_def`). `patchData` holds the result.

### AI models (ONNX via `ort`)
Downloaded on first use into the app data dir and SHA-verified: **SAM** (subject mask via box
prompt — encoder+decoder), **U²-Net** (foreground), **SkySeg** (sky), **Depth-Anything-v2**
(depth mask), **NIND** (denoise), **LaMa** (inpainting), **CLIP** (auto-tagging). Inference lives
in `ai_processing.rs`; commands in `ai_commands.rs`. The ONNX **Runtime** native lib itself is
fetched at build time by `build.rs`.

### AI connector (cloud)
Optional hosted service for AI features (no local GPU/models needed). `ai_connector.rs` +
`check_ai_connector_status`; the README "Cloud" section describes the convenience offering.

### LUT
3D color look-up table (loaded/parsed in `lut_processing.rs` → `Lut`, cached in `lut_cache`),
applied as the final shader stage; controlled by `lutData`/`lutIntensity`/`lutPath`/`lutSize` in
`Adjustments`. RapidRAW can also export LUTs.

### Tone mapper (`agx` / `basic`)
Selectable highlight roll-off / tone-mapping curve applied in the shader; field `toneMapper` in
`Adjustments`.

### `Invokes`
The TypeScript enum (`src/components/ui/AppProperties.tsx`) of every backend command name. Frontend
calls `invoke(Invokes.X, args)`. Each value must match a function registered in `lib.rs`'s
`generate_handler![…]`.

### Manager components
`ImageProcessingManager` / `ImageLoaderManager` — render nothing; they just host effect-heavy
hooks (`useImageProcessing` / `useImageLoader`) at a stable place in the tree.

### Library / album / culling
**Library** = the folder-tree-driven browser (grid/list) of images. **Albums** = user-defined
collections (JSON), independent of folders. **Culling** (`culling.rs`) = automated duplicate/blur
detection to thin a shoot.

### `rawler` / RapidRAW-DngLab
The RAW-decoding crate — a project-specific fork (`CyberTimon/RapidRAW-DngLab`). RAW decode entry
is `image_loader.rs` / `raw_processing.rs`.
