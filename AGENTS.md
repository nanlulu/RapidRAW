# AGENTS.md

Guidance for AI coding agents working in **RapidRAW** — a GPU-accelerated, non-destructive RAW
photo editor built with **Tauri 2 + React 19 + Rust + wgpu**.

## Start here

A full agent onboarding pack lives in **[`.ai/`](.ai/)**. Read it in order before non-trivial work:

1. [`.ai/00-overview.md`](.ai/00-overview.md) — what the app is + the one data-flow diagram.
2. [`.ai/01-architecture.md`](.ai/01-architecture.md) — frontend/backend halves, IPC, the edit→preview pipeline.
3. [`.ai/02-codemap.md`](.ai/02-codemap.md) — "where does X live" with file:line anchors.
4. [`.ai/03-conventions.md`](.ai/03-conventions.md) — build/run/lint, CI gates, gotchas.
5. [`.ai/04-recipes.md`](.ai/04-recipes.md) — step-by-step playbooks for common changes.
6. [`.ai/glossary.md`](.ai/glossary.md) — domain terms (sidecar, transform_hash, AiPatch, …).

## 30-second model

- **Frontend** (`src/`, React/TS) owns UI + edit **state** (5 Zustand stores). The current image's
  edits are one object, `Adjustments` (`src/utils/adjustments.ts`) — the source of truth and the
  unit of undo/redo, copy/paste, and presets.
- **Backend** (`src-tauri/src/`, Rust) owns RAW decode, the **wgpu/WGSL** pipeline
  (`shaders/shader.wgsl`), AI/ONNX models, files, thumbnails, export.
- They talk over Tauri: frontend `invoke(command, args)` ↔ Rust `#[tauri::command]` (registered in
  `src-tauri/src/lib.rs`, named in the `Invokes` enum at `src/components/ui/AppProperties.tsx`);
  Rust pushes async results back via `app_handle.emit(event, …)` ↔ `src/hooks/useTauriListeners.ts`.
- Editing is **non-destructive**: edits serialize to a per-image `.rrdata` JSON sidecar, never into
  the image file.

## Commands

```bash
npm install          # install frontend deps
npm start            # = `tauri dev` — THE dev command (Rust core + Vite together)
                     # `npm run dev` is Vite-only; invoke()/listen() won't work there.
```

Before declaring work done, run the checks CI enforces:

```bash
npm run typecheck                                            # tsc --noEmit (strict)
npm run lint && npm run format:check                         # eslint + prettier
npm run i18n:check                                           # translations must stay in sync
cargo fmt -p RapidRAW -- --check                             # (run inside src-tauri/)
cargo clippy --all-targets --all-features -- -D warnings     # warnings are errors
```

## Conventions that matter

- **Adjustment fields span three layers** that must agree: TS `Adjustments` (+`INITIAL_ADJUSTMENTS`),
  the Rust `AllAdjustments` uniform (`image_processing.rs`), and the WGSL struct/logic
  (`shader.wgsl`). A layout mismatch produces garbled output, not an error.
- **Cache invalidation is manual.** If an edit changes pixels, ensure the right hash in
  `cache_utils.rs` covers it (geometry → geometry/transform hash; tonal → visual hash). This is the
  most common subtle backend bug.
- **Feature logic goes in `src/hooks/`**, not in components and especially not in `App.tsx`
  (`App.tsx` is wiring only).
- **No hardcoded user-facing strings** — use i18next; English (`src/i18n/locales/en.json`) is the
  source locale.
- **IPC names** go through the `Invokes` enum; don't hardcode command strings.

## Verify, don't assume

Actually run the app via `npm start` and observe the change working — the pipeline supersedes stale
preview jobs, so "it compiled" is not "it works." When this file or anything in `.ai/` disagrees
with the code, **the code wins** — and please update the docs in the same change.
