# Product Spec — 仿色 (Tone Match): AI-Assisted, Explainable Tone Emulation

> Status: **Draft for review** · Owner: Lu · Drafted 2026-06-08 (branch `main`)
> Type: feature spec, no code yet. Lives in `.ai/specs/`; additive only.

---

## 1. Summary

**仿色** ("fǎngsè" — *to emulate/replicate a color tone*) lets a user provide one or more
**reference photos** whose look they admire and have RapidRAW **reproduce that tone on their
current photo** using only the app's existing, human-readable adjustments.

The differentiator is **teaching**, not filtering. The feature does not hand back a black-box LUT.
It:

1. **measures** the reference's tonal signature (deterministic, in Rust),
2. asks an **LLM to reason** about *what makes that look distinctive* and to **propose concrete,
   schema-valid adjustments** (vision + numeric stats as input),
3. **validates and applies** those adjustments locally, optionally **refining** by comparing the
   edited result's signature back to the reference's,
4. **shows the sliders move** so the user watches the tone being built, and
5. **explains** the reference's pattern and the rationale for each move, so the user learns to do
   it themselves.

Outputs: **apply to the current photo** and **save as a reusable preset** (with reference
thumbnails + the tonal summary embedded as teaching metadata).

## 2. Goals / Non-goals

**Goals**
- Reproduce a reference's *color grade / tonal style* on an arbitrary target photo using existing
  `Adjustments` only (no new LUT-baking in v1).
- Teach: every result comes with (a) a concise signature summary and (b) an on-demand per-slider
  breakdown explaining *why*.
- Be honest about transferability: clearly distinguish "the grade" (transferable) from "the
  reference's lighting/scene" (not reproducible by tone alone).
- Reuse RapidRAW's existing systems: `Adjustments`, presets, the auto-analysis stats path, the
  AI-Connector, the panel/tab UX patterns.

**Non-goals (v1)**
- No automatic 3D LUT generation (roadmap — see §13).
- No batch/multi-image application in v1 (single open editor image; preset enables manual batch
  later via existing apply).
- No subject/region-aware transfer (global grade only).
- No "match the target's content to the reference's content" (we match *signature*, not pixels).

## 3. User stories

- *As a photographer*, I drop in an Instagram screenshot I love and click Run; my open photo takes
  on the same mood, and I can see which sliders moved and tweak from there.
- *As a learner*, I expand "Teach me" and read why the AI lifted shadows, cooled the whites, and
  pushed teal into the midtones — then I try to reproduce it on the next photo by hand.
- *As a power user*, I save the result as a preset named "Kodak-ish warm" that remembers the
  reference thumbnails and the tonal summary, and apply it to future shoots.

## 4. Key product decisions (resolved with stakeholder)

| # | Decision | Choice |
|---|---|---|
| D1 | Compute split | **Rust derives/executes adjustments deterministically; LLM is the reasoning engine** that analyzes references, summarizes the tone pattern, and proposes adjustments. |
| D2 | LLM input | **Vision + numeric stats** (downscaled images *and* Rust-extracted statistics). |
| D3 | LLM provider | **Reuse the existing RapidRAW-AI-Connector** as the routing layer; user brings their own LLM (local Ollama-style OpenAI-compatible endpoint, or Gemini/Claude/OpenAI keys) configured behind it. |
| D4 | Output fidelity | **Teachable sliders only** — solution constrained to the existing `Adjustments` subset; no LUT in v1. |
| D5 | Reference semantics | **Tone/color signature only** — transfer the grade (WB bias, tonal contrast shape, color grading, saturation/HSL tendencies), **not** scene-specific exposure/subject color. |
| D6 | LLM→engine contract | **Strict JSON schema + validate/clamp + re-render→refine loop** (bounded passes). |
| D7 | Reference count | **N = 1 to ~10**, single reference is first-class; a set is aggregated (median signature) with outlier warning. |
| D8 | Explanation depth | **Tiered**: always a short signature summary; deep per-slider tutorial generated **on demand** ("Teach me"). |
| D9 | Match metric | **Match the reference's signature, not the target** — the loop minimizes a content-independent signature distance (hard gate), with an optional LLM visual sanity check (roadmap). |
| D10 | Outputs | **Apply to current photo** + **Save as preset** (preset stores reference thumbnails + tonal summary as metadata). |
| D11 | UI placement | **Left-panel tab**, symmetric to the right-panel switcher: a Folders/Album tab + a 仿色 tab. Activating 仿色 **replaces** the folder/album content on the left so the **right adjustments panel stays visible** and the user watches sliders animate. |
| D12 | Apply animation | **Default: apply instantly** (all-at-once smooth transition). Settings let the user switch to **sequential-fast** or **sequential-narrated**. All three on the roadmap. |
| D13 | Honesty vs. push | **v1 = honest grade transfer (mode A), default.** A more aggressive "push to match" (mode B) is a fast-follow when A is insufficient. |
| D14 | Failure UX | **Apply best-effort AND explain the gap** when the signature-distance gate isn't met after N passes. |
| D15 | Reference source (v1) | **External files only** (drag-in / file picker), including others' JPEGs / social screenshots. "Add from library catalog" is roadmap. |
| D16 | Privacy | **Always downscale + strip EXIF/GPS** before sending to any LLM; **show a notice when a cloud provider is selected** (no notice for local). |
| D17 | v1 scope | **Single currently-open editor image.** |

## 5. What a "reference" is

- Any **image file** the user supplies (JPEG/PNG/screenshot/etc.), edited by anyone — we analyze
  **pixels**, never anyone's hidden edit settings. RAW references allowed (decoded via existing
  loader) but are decoded to pixels like any image.
- v1: external files via drag-drop or file picker. Library-catalog picking is roadmap.
- A reference set (1–10) is assumed tonally consistent; obvious outliers are flagged as a
  non-blocking warning. The set's signature is the **median** of per-image signatures.

## 6. The "tonal signature" (the heart of D5/D9)

A **content-independent** descriptor extracted in Rust (extends `perform_auto_analysis` in
`image_processing.rs`, which already computes luma histogram, mean saturation, and center/edge
luma). The signature is normalized so two photos of different scenes but the same grade score
similar. Proposed components (final list is an implementation detail of M1):

- **White-balance bias** — average chromaticity offset (cool/warm, green/magenta) in a
  luminance-normalized space, ideally measured on near-neutral regions to avoid subject color.
- **Tonal contrast shape** — the mapping curve implied by the luma histogram (black point, white
  point, shadow lift, highlight rolloff, overall S-curve strength) — *shape*, not absolute
  brightness (brightness is scene-dependent and deliberately excluded per D5).
- **Color grading tendencies** — hue/saturation pushes in shadows / midtones / highlights
  (split-tone signature), mappable to the existing `colorGrading` wheels.
- **Saturation / vibrance posture** — global saturation level and per-hue (HSL band) tendencies.
- **Optional**: faded-black / matte tendency, subtle channel mixing.

**Explicitly excluded** from transfer: absolute exposure/brightness, scene-specific subject hues,
anything that is a property of the reference's *light* rather than its *grade*. The teaching text
names these exclusions (D13/D14).

Define `signatureDistance(A, B)` over this descriptor; it is the loop's hard gate (D9).

## 7. End-to-end flow

```
[User] drops reference(s) + has target open
   │
   ▼
(1) Rust: decode → downscale → STRIP EXIF/GPS (D16) → extract signature per reference → aggregate (median, flag outliers)
   │     also extract target's current signature
   ▼
(2) Build LLM request: { reference downscaled images, reference signature(JSON), target downscaled image,
                          target signature, the Adjustments schema+ranges, mode A/B, instruction }
       └── routed through RapidRAW-AI-Connector to the user's chosen provider (D3)
   ▼
(3) LLM returns STRICT JSON: { proposedAdjustments(subset, in-range), shortSummary, perAdjustmentRationale[],
                               honestyNotes[] (what the target can't reproduce) }   (D2/D6/D8/D14)
   ▼
(4) Rust: VALIDATE against schema → CLAMP to ranges → drop unknown fields → apply to a copy of target Adjustments
   ▼
(5) Rust: render preview → measure edited-target signature → signatureDistance vs reference signature
       │
       ├── within tolerance OR pass-budget exhausted → DONE
       └── else → feed measured gap back to LLM for ONE refinement pass → goto (4)   (bounded, e.g. ≤ 2 passes)  (D9)
   ▼
(6) Frontend: write proposedAdjustments into useEditorStore (pushHistory) → sliders ANIMATE per setting (D12)
   ▼
(7) Frontend: show shortSummary always; "Teach me" expands → on-demand LLM call for deep per-slider tutorial (D8)
   ▼
(8) User: keep & tweak (it's just Adjustments) · Save as preset (with ref thumbs + summary) (D10) · Undo
```

If step (5) never converges: **apply best-effort and surface the honesty/gap explanation** (D14).

## 8. The LLM ↔ engine contract (D6)

- **Output schema** (versioned): a strict subset of `Adjustments` with documented numeric ranges,
  e.g. `{ temperature, tint, contrast, highlights, shadows, whites, blacks, vibrance, saturation,
  hsl{...}, colorGrading{shadows,midtones,highlights,balance,blending}, curves?, toneMapper? }`.
  Exposure/brightness are **excluded or tightly bounded** by default (D5).
- **Validation (Rust)**: reject unknown keys, clamp out-of-range, type-check, ensure the JSON
  parses to the same struct the GPU pipeline already consumes. Invalid → one re-ask with the
  validation error, else fail gracefully (D14).
- **Determinism boundary**: the LLM never touches pixels and never returns a LUT/curve it invented
  numerically beyond the schema; Rust is the sole executor, so results are reproducible and the
  existing cache/hash invariants ([01 §D](../01-architecture.md)) hold.
- **Schema is the teaching vocabulary**: because the LLM can only speak in real RapidRAW sliders,
  every explanation maps 1:1 to something the user can reproduce by hand.

## 9. UI / UX

**Placement (D11):** introduce a **left-panel tab switcher** mirroring `RightPanelSwitcher`:
- Tab A: **Folders / Album** (today's `FolderTree`/album content).
- Tab B: **仿色 / Tone Match**.
Switching to Tab B swaps the *left* panel content (the right Adjustments panel stays put), so the
user watches the right-side sliders animate while the left holds the Tone Match workspace.

**Tone Match workspace (left panel):**
- Reference **drop zone** + file picker; thumbnails of added references; per-reference remove;
  outlier warning chip.
- **Mode** selector: *Honest grade (A, default)* / *Push to match (B, roadmap)* (D13).
- **Run** button → progress (analyze → reason → apply → refine).
- **Result area**: short signature **summary** (always), **"Teach me"** expander (on-demand deep
  tutorial), **honesty notes**, and actions: **Apply** (already previewed), **Save as preset**,
  **Reset**.
- **Playback controls** (roadmap M4): replay the slider animation; step through.

**Apply animation (D12):** default instant; Settings → "Tone Match apply style" = Instant /
Sequential-fast / Sequential-narrated. Sequential plays moves in a teaching order
(WB → tonal contrast → color grade → saturation/HSL), with captions in narrated mode.

**Cloud notice (D16):** when the configured provider is cloud, show a one-line notice near Run
that downscaled, EXIF-stripped images will be sent.

## 10. Preset integration (D10)

Saving produces a normal `UserPreset` (`usePresets.ts` / `Preset` in `AppProperties.tsx`) whose
`Adjustments` are the derived values, **plus new optional teaching metadata**:
`{ toneMatch?: { referenceThumbnails: string[], signatureSummary: string, signatureVersion: number,
mode: 'A'|'B' } }`. This keeps presets backward-compatible (field is optional and ignored by older
code) and makes the preset itself re-teachable later. Applying the preset uses the existing apply
path — no special-casing in the pipeline.

## 11. Persistence, settings, security

- **Provider config** routed through the existing **RapidRAW-AI-Connector** (D3); LLM
  provider/base-URL/model/key live in app settings (extend `app_settings.rs` /
  `useSettingsStore`). **Keys never bundled**, stored locally (prefer OS keychain where available),
  never written to sidecars or presets.
- **Privacy (D16)**: a shared "prepare-for-LLM" step always downscales and **strips EXIF/GPS**
  before any send. Local LLM = no notice; cloud = explicit notice.
- **No new user-data file formats**; preset teaching metadata rides inside the existing preset JSON.

## 12. Touch points in the codebase (for implementers — verify before editing)

| Area | Likely file(s) | Nature |
|---|---|---|
| Signature extraction | `src-tauri/src/image_processing.rs` (extend `perform_auto_analysis`) | reuse/extend existing stats |
| New backend commands | `src-tauri/src/` (new module e.g. `tone_match.rs`) + register in `lib.rs` `generate_handler!` | new |
| LLM routing | reuse/extend `src-tauri/src/ai_connector.rs` | extend |
| Command names | `src/components/ui/AppProperties.tsx` `Invokes` enum | new entries |
| Apply / animate sliders | `src/store/useEditorStore.ts` (`setAdjustments`/`pushHistory`); a new `useToneMatch` hook | new hook |
| Left tab switcher | new left-panel switcher mirroring `RightPanelSwitcher.tsx`; wire in `App.tsx` left region (`FolderTree` render) | new |
| Workspace UI | new `components/panel/left/ToneMatchPanel.tsx` | new |
| Preset metadata | `src/hooks/usePresets.ts`, `Preset` type, `file_management.rs` preset save/load | extend (optional field) |
| Settings | `app_settings.rs`, `useSettingsStore`, `SettingsPanel.tsx` | extend |
| i18n | `src/i18n/locales/en.json` (+ keys) | new strings |

(Adheres to [03 · Conventions](../03-conventions.md): logic in hooks, IPC via `Invokes`, no
hardcoded strings, three-layer agreement is N/A since no new GPU field — we reuse existing
`Adjustments` end-to-end. Cache invariants unaffected because output is ordinary `Adjustments`.)

## 13. Milestones / roadmap

- **M0 — Spec & schema** (this doc): finalize the tonal-signature descriptor (Appendix A), the LLM
  output JSON schema + ranges (Appendix B), the `signatureDistance` weights (Appendix C), and the
  M1 acceptance-test design (Appendix D).
- **M1 — Local signature engine**: Rust extraction + aggregation + `signatureDistance`; ship the
  **Appendix-D acceptance harness** and pass its gates. No LLM yet — *prove the measurement is
  content-independent before anything else is built.*
- **M2 — LLM proposal (one-shot)**: connector routing, vision+stats request, strict-schema
  validate/clamp, apply to current image, short summary. Mode A only. Instant apply.
- **M3 — Refine loop + failure UX**: render→measure→refine (≤2 passes), best-effort + gap
  explanation (D14).
- **M4 — Teaching layer**: on-demand "Teach me" deep per-slider tutorial; sequential apply
  animations (fast + narrated) with settings toggle (D12).
- **M5 — Preset output**: save-as-preset with reference thumbnails + signature summary metadata
  (D10).
- **M6 — Local-LLM parity & provider config UX**: ensure local (Ollama-style) and cloud keys both
  work through the connector; settings UI; cloud privacy notice.
- **Fast-follow / later**: mode B "push to match" (D13); add-references-from-library (D15);
  multi-image/batch apply; optional auto-LUT export for exact match; LLM visual sanity-check in the
  loop; outlier clustering for inconsistent sets.

## 14. Open questions / risks

- **Signature design is the make-or-break.** If `signatureDistance` isn't truly
  content-independent, matches will chase the reference's *scene* and mislead the teaching. M1 must
  validate this with controlled tests (same grade on different scenes should score near-zero).
- **WB-on-neutral estimation** without subject contamination is non-trivial; may need a simple
  gray-region heuristic. Risk to D5 fidelity.
- **LLM variance**: even schema-constrained, taste varies across providers/models. The numeric gate
  (D9) bounds drift, but explanations may differ in quality — acceptable for v1.
- **Latency/cost**: vision + up to 3 LLM calls (propose, refine, teach) per use. Tiered explanation
  (D8) and instant-apply default (D12) mitigate.
- **Mode B scope creep**: keep B out of v1; only build if A demonstrably falls short.

---

## Appendix A — M0 deliverable: tonal signature descriptor (v1)

This is the **content-independent** descriptor extracted in Rust (M1). Design principle: every
field must be invariant to *what the photo is of* and to *how bright the scene was* — two photos of
different subjects with the same grade must produce near-identical signatures (the M1 acceptance
test). Brightness/exposure is deliberately **excluded** (D5); only the *shape* of the tone response
and the *color posture* are kept.

```jsonc
// ToneSignature v1 — all values normalized; ranges noted. JSON for transport to the LLM.
{
  "version": 1,
  "whiteBalance": {
    // measured on near-neutral pixels only (low-saturation, mid-luma) to avoid subject color
    "tempBiasKelvinish": 0.0,   // -1..1  negative = cooler, positive = warmer (relative, not absolute K)
    "tintBias": 0.0,            // -1..1  negative = green, positive = magenta
    "neutralConfidence": 0.0    //  0..1  how many reliable neutral pixels were found (low ⇒ WB less trustworthy)
  },
  "tone": {
    // SHAPE of the luma response, brightness-normalized (histogram aligned to fixed mid-gray anchor)
    "blackPoint": 0.0,          // 0..1   where shadows begin to lift off 0
    "whitePoint": 1.0,          // 0..1   where highlights clip/roll
    "shadowLift": 0.0,          // -1..1  matte/faded-black tendency (raised shadow floor)
    "highlightRolloff": 0.0,    // -1..1  soft (filmic) vs hard highlight shoulder
    "contrastSlope": 0.0,       // -1..1  global S-curve strength
    "midtoneGamma": 0.0         // -1..1  midtone push independent of endpoints
  },
  "colorGrade": {
    // split-tone signature → maps onto existing `colorGrading` wheels
    "shadows":    { "hue": 0,   "sat": 0.0 },  // hue 0..360, sat 0..1
    "midtones":   { "hue": 0,   "sat": 0.0 },
    "highlights": { "hue": 0,   "sat": 0.0 },
    "balance": 0.0,             // -1..1  shadow↔highlight weighting
    "globalCast": { "hue": 0, "sat": 0.0 }     // overall tint not tied to a tonal zone
  },
  "saturation": {
    "global": 0.0,              // -1..1  overall saturation posture
    "vibranceSkew": 0.0,        // -1..1  protect-skin / boost-muted tendency
    "perHue": {                 // -1..1 each — HSL-band saturation tendencies
      "red": 0.0, "orange": 0.0, "yellow": 0.0, "green": 0.0,
      "aqua": 0.0, "blue": 0.0, "purple": 0.0, "magenta": 0.0
    },
    "perHueLuma": {             // -1..1 each — optional, per-band luminance tendency
      "red": 0.0, "orange": 0.0, "yellow": 0.0, "green": 0.0,
      "aqua": 0.0, "blue": 0.0, "purple": 0.0, "magenta": 0.0
    }
  }
}
```

`signatureDistance(A, B)` = weighted L2 over these fields (weights TBD in M1; WB and colorGrade
likely weighted higher than perHueLuma). **Aggregation** of a reference set = per-field median, with
an outlier flag when any reference's distance to the median exceeds a threshold (D7).

**M1 acceptance test:** apply a known grade (e.g. a LUT or preset) to ≥5 visually different scenes;
all their signatures must cluster (pairwise distance below ε), and an ungraded version must sit far
from the graded cluster. If this fails, the descriptor is wrong — fix before building M2.

## Appendix B — M0 deliverable: LLM output schema (v1)

The LLM **must** return exactly this object (strict JSON, no prose outside it). Rust validates,
drops unknown keys, and clamps every value to the documented `Adjustments` range before applying
(D6). Field names are the real `Adjustments` keys so the proposal maps 1:1 to teachable sliders.

```jsonc
{
  "schemaVersion": 1,
  "mode": "A",                       // "A" honest grade (v1 default) | "B" push-to-match (roadmap)
  "proposedAdjustments": {
    // SUBSET of Adjustments. Exposure/brightness omitted by default (D5); include only if mode B.
    "temperature": 0, "tint": 0,
    "contrast": 0, "highlights": 0, "shadows": 0, "whites": 0, "blacks": 0,
    "vibrance": 0, "saturation": 0,
    "hsl": { /* per-band hue/sat/lum, same shape as Adjustments.hsl */ },
    "colorGrading": { /* shadows/midtones/highlights + balance + blending, same shape as Adjustments */ },
    "curves": null,                  // optional; null unless a tone curve is essential
    "toneMapper": null               // "agx" | "basic" | null (leave as-is)
  },
  "shortSummary": "1–2 sentences naming the tone's signature (always shown).",
  "perAdjustmentRationale": [
    { "field": "temperature", "value": -8,
      "why": "Reference whites sit cool/neutral; your image skews warm, so we pull temp down to match the cast.",
      "teaches": "Look at near-white areas first — they reveal the white-balance intent before anything else." }
    // one entry per non-zero proposed field (drives the per-slider 'Teach me' tutorial, D8)
  ],
  "honestyNotes": [
    // what the target can't reproduce by tone alone (D5/D14) — shown in the result area
    "The reference's glow comes from golden-hour backlight; tone can match color but not that directional light."
  ],
  "confidence": 0.0                  // 0..1 LLM self-rating; low ⇒ surface caution in UI
}
```

**Validation rules (Rust):** unknown keys → dropped; out-of-range → clamped (and noted to the user
if clamping was large); must deserialize into the same struct the GPU pipeline consumes; on parse
failure → one re-ask with the error, else best-effort + gap explanation (D14). The refinement pass
(D9) appends a `measuredGap` object (edited-target signature minus reference signature, per field)
to the next request so the LLM corrects in the right direction.

---

## Appendix C — `signatureDistance` weights (v1)

`signatureDistance(A, B)` is a weighted Euclidean distance over the Appendix-A fields, **after each
field is normalized to roughly `[-1, 1]` (or `[0, 1]`)** so weights express *relative perceptual
importance*, not unit scale. Hues are compared as **circular** differences (shortest arc), scaled by
the band's saturation so a hue swing on a near-gray band barely counts.

```
distance² = Σ  wᵢ · dᵢ²
```

**Weights (sum ≈ 1.0). These are starting values to be tuned against the M1 corpus, not gospel.**

| Group → field | Weight | Why this weight |
|---|---:|---|
| **whiteBalance.tempBias** | 0.16 | WB cast is the single most identity-defining part of a "look"; humans read it instantly. |
| **whiteBalance.tintBias** | 0.08 | Green/magenta matters but less than temp; often subtle in stylistic grades. |
| **tone.contrastSlope** | 0.12 | Overall contrast is a primary stylistic axis (punchy vs. flat). |
| **tone.shadowLift** (matte) | 0.10 | The faded/matte-black tendency is a strong, recognizable signature (film emulation, "moody"). |
| **tone.highlightRolloff** | 0.07 | Soft vs. hard highlights distinguishes filmic looks. |
| **tone.midtoneGamma** | 0.05 | Midtone push; secondary to slope + endpoints. |
| **tone.blackPoint / whitePoint** | 0.03 each (0.06) | Endpoint placement; partly correlated with slope/lift, so down-weighted to avoid double-counting. |
| **colorGrade.shadows {hue,sat}** | 0.09 | Split-toning shadows (e.g. teal shadows) is a hallmark of graded looks. |
| **colorGrade.highlights {hue,sat}** | 0.07 | Highlight tint (e.g. warm/orange highlights) — the other half of teal-and-orange. |
| **colorGrade.midtones {hue,sat}** | 0.04 | Less commonly the defining move; still relevant. |
| **colorGrade.balance + globalCast** | 0.03 | Overall weighting/cast; small. |
| **saturation.global** | 0.06 | Muted vs. vivid is a clear stylistic axis. |
| **saturation.vibranceSkew** | 0.02 | Skin-protect / muted-boost tendency; subtle. |
| **saturation.perHue (8 bands)** | 0.05 total (~0.006 each) | Per-band tendencies matter collectively but no single band should dominate. |
| **saturation.perHueLuma (8 bands)** | 0.02 total | Lowest weight; noisy to estimate, easy to over-fit. Kept low deliberately. |

**Design rationale**
- **WB + contrast + split-tone carry ~0.6 of the weight.** These are what a human names when
  describing a look ("warm, contrasty, teal shadows"), so the metric should agree with intuition —
  and so the *teaching text* lines up with the *measured gap*.
- **Brightness/exposure has weight 0** — it is not in the descriptor at all (D5). The metric is
  blind to how bright the scene was.
- **Per-hue-luma is near-zero** because it is the noisiest feature on real photos and the easiest
  way to make the metric chase content instead of grade (the central risk in §14).
- Hue terms are **saturation-gated**: `dhue² · min(satA, satB)` so a hue reading on an unsaturated
  zone (where hue is ill-defined) contributes almost nothing.

**Acceptance for the weights (part of M1):** with these weights, the corpus test in Appendix D must
pass with a comfortable margin between the "same-grade" and "different-grade" distance
distributions. If it doesn't, re-tune *weights first*, descriptor *fields second*.

## Appendix D — M1 acceptance-test harness

Goal: **prove the signature is content-independent before any LLM work.** M1 ships the Rust
extractor + `signatureDistance` and this harness; M2 does not start until D passes.

### Corpus

- **Scenes**: ≥ 8 visually diverse base images (portrait, landscape, indoor low-light, high-key
  snow, neon night, foliage, skin-tone close-up, architectural) — ideally as RAW/neutral TIFF so
  grades are applied cleanly.
- **Grades**: ≥ 5 distinct, well-separated looks, each expressed as a RapidRAW preset or a LUT:
  e.g. `G1 warm-film-matte`, `G2 teal-orange-punchy`, `G3 cool-clean`, `G4 vintage-faded`,
  `G5 high-contrast-bw-ish`. Plus `G0 = ungraded`.
- Apply every grade to every scene → an `8 × 6` matrix of graded images (export at a fixed size).

### Metrics computed

For all image pairs, compute `signatureDistance`, then bucket:
- **intra-grade** (same grade, different scene) distances — should be **small**.
- **inter-grade** (different grade) distances — should be **large**.
- **vs-ungraded** (graded vs its own G0 scene) — should be **large** (proves we detect the grade,
  not the scene).

### Pass criteria (gates)

1. **Separation**: `max(intra-grade distance)` < `min(inter-grade distance)` — i.e. the two
   distributions don't overlap. (Strong form. Weak fallback: 95th-percentile intra < 5th-percentile
   inter, with the overlap logged.)
2. **Content-independence**: for each grade, the std-dev of intra-grade distance across scenes is
   below a threshold τ (the same look on a portrait vs. a landscape scores nearly the same).
3. **Sensitivity**: every graded image is farther from its ungraded original than from any
   same-grade different-scene image (we react to grade more than to content).
4. **Nearest-neighbor classification**: label each graded image by its nearest neighbor's grade;
   **≥ 95% accuracy**. (This is the headline number to report.)
5. **WB robustness**: criteria 1–4 still hold on a subset where scenes have strongly colored
   subjects (red dress, green field) — guards the WB-on-neutral estimator (the §14 risk).

### Harness shape (proposed)

- A Rust integration test (`src-tauri/tests/tone_signature.rs`) or a small dev-only binary that:
  loads the corpus from a fixtures dir, runs the extractor, builds the distance matrix, computes the
  five metrics, and asserts the gates — emitting a CSV/heatmap for human inspection on failure.
- Corpus images live **outside the repo** (large/licensing); the test reads a path from an env var
  and **skips with a clear message** when absent, so CI stays green without the fixtures while the
  gate is runnable locally. (Document this in the test — don't silently pass.)
- Report artifact: a distance heatmap (scenes × grades) makes overlaps obvious at a glance; attach
  it to the M1 PR.

### If it fails

Tune in this order (cheapest first): (1) `signatureDistance` **weights** (Appendix C); (2) hue
saturation-gating / normalization; (3) the **neutral-pixel selection** for WB; (4) add/remove
**descriptor fields** (Appendix A). Re-run the harness after each change. Only when D passes does the
LLM proposal work (M2) begin.

## Verification

Grounded against: `src/utils/adjustments.ts` (`Adjustments`), `src/hooks/usePresets.ts` +
`AppProperties.tsx` (`UserPreset`/`Preset`/`Invokes`), `src-tauri/src/image_processing.rs`
(`perform_auto_analysis` at ~line 2992 — confirmed it already extracts luma histogram, mean
saturation, center/edge luma), `src-tauri/src/ai_connector.rs` (reqwest-based connector pattern),
`src/components/panel/right/RightPanelSwitcher.tsx` (tab pattern to mirror), and `App.tsx`
left-panel/`FolderTree` rendering. All "touch points" are *proposed* and must be re-verified at
implementation time. Appendices A–D (descriptor, LLM schema, distance weights, acceptance harness)
are v1 proposals; the weights and gates are tuning targets to be validated against the M1 corpus.
Slider ranges cited (most tonal/color = −100..100; exposure = −5..5 EV; color-grade wheel hue =
0..360, blending 0..100) were read from `src/components/adjustments/{Basic,Color}.tsx` and
`src/components/ui/ColorWheel.tsx`; HSL/colorGrading shapes from `src/utils/adjustments.ts`
(`Hsl`, `ColorGradingProps`, `HueSatLum`).
