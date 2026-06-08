# 04 · Recipes — common changes, step by step

Each recipe lists the files to touch in order. Verify against the current code before editing
(line numbers drift). After any recipe, run the matching checks in [03](03-conventions.md).

---

## §1 · Add a new global adjustment (e.g. a new tone slider)

The data must agree across **three layers**: TS model, Rust uniform, WGSL math.

1. **Frontend model** — `src/utils/adjustments.ts`:
   - Add the field to the `Adjustments` interface.
   - Add its default to `INITIAL_ADJUSTMENTS`. (Loading old sidecars auto-back-fills it through
     `normalizeLoadedAdjustments`'s spread — no migration code needed.)
   - If it belongs to a group enum (e.g. `BasicAdjustment`/`ColorAdjustment`), add it there too.
2. **Frontend UI** — add a `Slider` (or control) in the right `src/components/adjustments/*.tsx`
   section (Basic/Color/Curves/Details/Effects). Wire `onChange → setAdjustments({ field })`.
   Use a **translated label** (i18next) — never a raw string.
3. **Rust uniform** — `src-tauri/src/image_processing.rs`: add the field to the `AllAdjustments`
   (or the relevant nested) `#[repr(C)] + bytemuck::Pod` struct, and populate it where JSON
   adjustments are parsed into the uniform. **Match field order/padding with the WGSL struct.**
4. **WGSL** — `src-tauri/src/shaders/shader.wgsl`: add the matching field to the corresponding
   struct and apply it in the pixel function at the correct pipeline stage (tone → curves →
   color → grading → tonemap → local → effects → LUT).
5. **Cache** — if the field changes pixels, confirm it's covered by a hash in `cache_utils.rs`.
   Tonal/color → it must feed `calculate_visual_hash`; geometry → geometry/transform hash. If you
   added it to the JSON the preview-worker already hashes wholesale, you may be covered — verify.
6. **Verify**: `npm run typecheck && npm run lint`, `cargo clippy … -D warnings`, then run
   `npm start` and confirm the slider moves pixels and the value persists across reload.

> Pitfall: a struct-layout mismatch between Rust and WGSL produces *garbled* output, not an
> error. Keep the two struct definitions visually side-by-side while editing.

---

## §2 · Add a new Tauri command (backend capability the UI calls)

1. **Implement** the function in the most relevant `src-tauri/src/*.rs` module, annotated
   `#[tauri::command]`. Take `tauri::State<'_, AppState>` and/or `AppHandle` as needed; return
   `Result<T, String>` (the frontend gets the `Err` string).
2. **Register** it in `src-tauri/src/lib.rs` inside `generate_handler![…]` (use `module::fn` form
   if it's not defined in `lib.rs`).
3. **Expose the name** in `src/components/ui/AppProperties.tsx` → add a variant to `enum Invokes`
   whose string value is the **exact snake_case command name**.
4. **Call** it from a frontend hook: `await invoke(Invokes.YourCommand, { camelCaseArgs })`.
   Tauri maps camelCase JS keys → snake_case Rust params automatically.
5. If it's long-running, **emit progress** from Rust (`app_handle.emit("your-progress", payload)`)
   and `listen` for it in `src/hooks/useTauriListeners.ts`; reflect state in the right store
   (usually `useProcessStore`).
6. **Tauri permissions**: check `src-tauri/capabilities/` if the command touches a capability that
   needs allow-listing (fs/dialog/shell/etc.).
7. **Verify**: `cargo clippy`, `npm run typecheck`, exercise it via `npm start`.

---

## §3 · Change the GPU image math

1. Edit `src-tauri/src/shaders/shader.wgsl` (main pipeline) or `blur.wgsl`/`flare.wgsl`. Respect
   the existing stage ordering and the `GlobalAdjustments`/per-mask struct layouts.
2. If you add/remove a binding or buffer, update the bind-group layout, buffer creation, and
   dispatch in `src-tauri/src/gpu_processing.rs`, and the `AllAdjustments`/param struct in
   `image_processing.rs`.
3. WGSL has no debugger here — debug by isolating the stage (e.g. temporarily output the
   intermediate value) and rebuild. Watch for `f16`/`f32` and color-space (linear vs sRGB)
   assumptions already in the file.
4. **Verify** visually with `npm start` across a RAW and a JPEG; check the histogram/waveform
   panel reacts sanely. Test the GPU backend setting if the change is backend-sensitive (Metal
   on macOS, Vulkan/DX12 elsewhere — see README "Common Problems").

---

## §4 · Add a new mask type

1. **Frontend**: add the variant to the `Mask` enum and handle its parameters/UI in
   `src/components/panel/right/Masks.tsx` + `MasksPanel.tsx`. A `SubMask` carries `type`, `mode`
   (Additive/Subtractive/Intersect), `parameters`, opacity/invert/visible.
2. **Rasterization**: implement how the mask becomes a grayscale bitmap in
   `src-tauri/src/mask_generation.rs` (`generate_mask_bitmap` and friends). Ensure it's covered by
   `generate_mask_overlay` so the user sees the overlay.
3. **AI masks**: if it's ML-backed (like subject/sky/depth/foreground), add the inference path in
   `ai_processing.rs` + a command in `ai_commands.rs`, register in `lib.rs`, expose via `Invokes`,
   and drive it from `src/hooks/useAiMasking.ts` / `AIPanel.tsx`.
4. **Cache**: mask rasterization is cached in `AppState.mask_cache` by a hash of the mask def +
   geometry; make sure your new parameters participate in that key.
5. **Verify**: create the mask, confirm overlay + that masked adjustments only affect the region,
   reload to confirm it persists in `.rrdata`.

---

## §5 · Add a UI string / new language key

1. Use the i18next translation function in the component (no literal strings).
2. Add the English key to `src/i18n/locales/en.json` (source locale, nested key style).
3. Run `npm run i18n:extract`, then `npm run i18n:check` until clean. Other locales
   (`de, es, fr, it, ja, pl, pt, ru, zh-CN`) can carry the English fallback until translated, but
   the **key set must stay in sync** or CI fails.

---

## §6 · Add/extend a frontend feature behavior

Prefer a hook over component code. Create or extend a `src/hooks/use*.ts`, read/write the relevant
Zustand store, and call it from `App.tsx` (wiring only) or the owning view/panel. Keep `App.tsx`
free of business logic — it's a wiring board.

---

## §7 · Persisted edits / sidecar changes

The per-image edit state round-trips through the `.rrdata` sidecar (`ImageMetadata` JSON:
`version`, `rating`, `adjustments`, `tags`, `exif`). Read/write helpers live in
`exif_processing.rs` (`get_primary_sidecar_path`, `load_sidecar`, save helpers) and are driven by
`file_management.rs` commands (`load_metadata`, `save_metadata_and_update_thumbnail`,
`apply_adjustments_to_paths`, …). If you change the schema, bump `version` and handle the old
shape on load; the frontend's `normalizeLoadedAdjustments` is the complementary FE-side migration.

---

## Quick checklist before "done"

- [ ] `npm run typecheck` clean
- [ ] `npm run lint` + `npm run format:check` clean
- [ ] `npm run i18n:check` clean (if you touched UI strings)
- [ ] `cargo fmt -p RapidRAW -- --check` + `cargo clippy --all-targets --all-features -- -D warnings` clean
- [ ] Ran the actual app via `npm start` and observed the change working (not just compiled)
- [ ] If pixels changed: confirmed the correct cache hash invalidates
- [ ] Updated the affected `.ai/*.md` if you changed structure

## Verification

Recipes synthesized from the architecture in [01](01-architecture.md) and the anchors in
[02](02-codemap.md); the three-layer adjustment path confirmed against `adjustments.ts`,
`image_processing.rs` (`AllAdjustments`), and `shader.wgsl`; command-registration path against
`lib.rs` `generate_handler!` and `AppProperties.tsx` `Invokes`.
