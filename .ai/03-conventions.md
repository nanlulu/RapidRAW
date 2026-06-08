# 03 · Conventions, Build & Guardrails

## Build & run

Prerequisites: **Rust** (≥ 1.95, edition 2024) and **Node.js**. First `npm start` also triggers
`build.rs`, which **downloads and SHA-verifies the ONNX Runtime** native lib for your platform —
so the first build needs network access and is slow.

```bash
npm install        # install frontend deps (also vendors node_modules used by Vite)
npm start          # = `tauri dev` — launches Rust core + Vite together (THE dev command)

npm run dev        # Vite only (frontend in a browser; no Tauri APIs — invoke() will fail)
npm run build      # Vite production build of the frontend
npm run tauri      # raw Tauri CLI passthrough (e.g. `npm run tauri build` for installers)
```

Backend-only iteration happens inside `src-tauri/` with normal `cargo` (`cargo build`,
`cargo clippy`, `cargo fmt`). The Tauri dev server hot-reloads the frontend; Rust changes require
a rebuild (the `tauri dev` watcher handles it but it's a full recompile).

## Quality commands (run before you call work done)

```bash
npm run typecheck      # tsc --noEmit (strict mode is ON)
npm run lint           # eslint .         (CI runs this)
npm run lint:fix       # eslint --fix
npm run format         # prettier --write .
npm run format:check   # prettier --check .   (CI runs this)
npm run i18n:check     # i18next-cli extract --ci --dry-run  (CI runs this — translations must be in sync)
npm run i18n:extract   # regenerate locale keys after adding UI strings
npm run i18n:lint
```

Backend:
```bash
cargo fmt -p RapidRAW -- --check      # CI runs this
cargo clippy --all-targets --all-features -- -D warnings    # CI runs this — WARNINGS ARE ERRORS
```

## What CI enforces (`.github/workflows/lint.yml`)

CI **will fail** on any of: unformatted JS/TS (prettier), eslint errors, **out-of-sync
translations** (`i18n:check`), unformatted Rust (`cargo fmt`), or **any clippy warning**
(`-D warnings`). There are additional `ci.yml`, `pr-ci.yml`, `build.yml`, `release.yml` workflows
for cross-platform builds. Before declaring a change done, run the matching commands locally.

## Code style

**TypeScript / React**
- Prettier: single quotes, semicolons, trailing commas (`all`), **printWidth 120**.
- `tsconfig` is `strict: true`, `target es2025`, `jsx: react-jsx`, `moduleResolution nodenext`,
  `rootDir ./src`. No implicit any; keep types honest.
- React 19, function components + hooks only. **Feature logic lives in `src/hooks/`**, not in
  components and *especially* not in `App.tsx`.
- State via Zustand stores; read with `useShallow` when selecting multiple fields to avoid
  re-renders. Don't add React Context for app state — match the store pattern.
- IPC: always go through the `Invokes` enum in `AppProperties.tsx`; don't hardcode command strings.

**Rust**
- Edition 2024, `cargo fmt` defaults, clippy-clean (no `#[allow]` without reason).
- All user-callable functions are `#[tauri::command]` and registered in `lib.rs`'s
  `generate_handler![…]`. Shared state is accessed via `tauri::State<AppState>`.
- Heavy CPU work → `rayon`; async/IO → `tokio`; never block the preview-worker thread on I/O you
  can cache.
- GPU uniform structs must be `#[repr(C)]` + `bytemuck::Pod`/`Zeroable`; field order and padding
  must match the WGSL struct exactly.

**i18n**
- User-facing strings are **never** hardcoded — use the i18next translation function. English
  (`src/i18n/locales/en.json`) is the source locale; after adding keys run `npm run i18n:extract`
  and ensure `i18n:check` passes. i18n-ally config: nested key style, namespaces off.

## Gotchas (these will bite you)

1. **Cache invalidation is manual.** If your new adjustment field changes pixels, make sure the
   relevant hash in `cache_utils.rs` includes it (geometry-affecting → geometry/transform hash;
   tonal → visual hash). Miss this and previews/thumbnails go stale silently. See [01 §D](01-architecture.md).
2. **Two sidecars.** `.rrdata` is the **primary** per-image sidecar (JSON `ImageMetadata`:
   version, rating, adjustments, tags, exif). `.rrexif` is **legacy** EXIF, still read for
   back-compat. New adjustment persistence goes through `.rrdata` (`get_primary_sidecar_path`).
3. **New adjustment field → three places.** TS `Adjustments` + `INITIAL_ADJUSTMENTS`
   (and it auto-migrates via `normalizeLoadedAdjustments` spread), the Rust `AllAdjustments`
   uniform, and the WGSL struct/logic. Skip one and you get a type error, a silent no-op, or a
   GPU layout mismatch (garbage output).
4. **`npm run dev` ≠ the app.** Tauri APIs (`invoke`, `listen`) only exist under `npm start`.
   The browser-only Vite server is for pure UI work.
5. **Job superseding.** The preview pipeline drops stale jobs by id. When debugging "my preview
   didn't update," check whether a newer job superseded it before suspecting the GPU.
6. **First build downloads models/runtime.** ONNX Runtime via `build.rs`; AI **models** download
   on first use into the app data dir and are SHA-verified. Offline = those features no-op/fail.
7. **Sidecars are user data.** Don't delete/rewrite `.rrdata`/`.rrexif` casually; they hold the
   user's non-destructive edits.

## Git / contribution

- Default branch `main`. There's a PR template (`.github/pull_request_template.md`), issue
  templates, and `CODEOWNERS`. License is AGPL-3.0 — keep new files compatible.
- The task said **do not modify existing files/structure**; this `.ai/` pack is additive only.

## Verification

`package.json` scripts; `.prettierrc`; `tsconfig.json`; `.github/workflows/lint.yml`;
`src-tauri/build.rs`; `src-tauri/Cargo.toml`; `.vscode/settings.json` (i18n-ally).
