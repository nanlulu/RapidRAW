# `.ai/` — Agent Onboarding Pack

AI-agent-native documentation for **RapidRAW**, a GPU-accelerated, non-destructive RAW
photo editor (think a lightweight Lightroom) built with **Tauri 2 + React 19 + Rust + wgpu**.

These files exist to let a coding agent ramp up to productive work in one session
**without re-reading the whole tree**. They are hand-derived from the source and verified
against it; treat them as a map, not as ground truth. When a doc and the code disagree,
**the code wins** — and please fix the doc.

## Read order

1. **[`00-overview.md`](00-overview.md)** — what the app is, the tech stack, the top-level
   mental model, and the one diagram that explains the whole data flow.
2. **[`01-architecture.md`](01-architecture.md)** — the two halves (React frontend, Rust
   backend), how they talk (Tauri `invoke` + events), and the editing pipeline end to end.
3. **[`02-codemap.md`](02-codemap.md)** — directory-by-directory "where does X live" index
   with file:line anchors for every important entry point.
4. **[`03-conventions.md`](03-conventions.md)** — build/run/lint commands, code style, the
   guardrails CI enforces, and gotchas that will bite you.
5. **[`04-recipes.md`](04-recipes.md)** — step-by-step playbooks for the changes agents are
   most often asked to make (add an adjustment, add a Tauri command, add a mask type, …).
6. **[`glossary.md`](glossary.md)** — domain terms (sidecar, transform_hash, AiPatch, …).

### Feature specs

- **[`specs/fangse-tone-match.md`](specs/fangse-tone-match.md)** — 仿色 / Tone Match: AI-assisted,
  explainable tone emulation (reference photos → derived `Adjustments` + teaching breakdown).
  Draft spec, not yet implemented.

## The 30-second model

- **Frontend** (`src/`, TypeScript/React) owns all UI and the editing **state** (5 Zustand
  stores). The current image's edits live in one object: `Adjustments`. It is the single
  source of truth and the unit of undo/redo, copy/paste, and presets.
- **Backend** (`src-tauri/src/`, Rust) owns all heavy lifting: RAW decode, the **wgpu
  GPU pipeline** (`shader.wgsl`), AI/ONNX models, file I/O, thumbnails, export.
- They talk over Tauri: frontend `invoke(command, args)` → Rust `#[tauri::command]`; Rust
  pushes async results back via `app_handle.emit(event, payload)` → frontend listeners.
- Editing is **non-destructive**: edits are never baked into the file. They are serialized
  to a `.rrdata` JSON **sidecar** next to each image and re-applied on load.

## Maintenance note for agents

If you make a structural change (new store, new command, moved subsystem, renamed sidecar),
update the relevant file here in the same change. These docs were written on **2026-06-08**
against branch `main`; the `## Verification` section at the bottom of each file lists how its
claims were checked so you can re-verify cheaply.
