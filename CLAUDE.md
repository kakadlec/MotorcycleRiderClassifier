# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

**Motorcycle Rider Classifier** is a fork of Frank Sherlock (a local-first image cataloging desktop app). The goal is to build a local-only desktop tool that organizes motorcycle event photos by rider — using windscreen sticker number recognition, visual similarity of rider/helmet/clothing/motorcycle, and manual review workflows.

The inherited Frank Sherlock app (`sherlock/desktop/`) is the technical foundation but is not the final product. **Do not add Frank Sherlock features or modify the desktop app until domain experiments (Phase 1) are complete.** See `AGENTS.md` for the full development mandate.

## Repository Layout

```
sherlock/desktop/          <- Inherited Frank Sherlock app (Tauri v2 + React + Rust)
  src-tauri/src/           <- Rust backend (scan.rs, db.rs, face.rs, etc.)
  src/                     <- React + Vite + TypeScript frontend
_classification/           <- Python PoC for image classification (inherited)
_research_ab_test/         <- A/B benchmark lab for models/OCR (inherited)
_face_ab_test/             <- Benchmark lab for face detection (inherited)
_local/                    <- Local corpus and outputs (gitignored, never commit)
docs/                      <- Planning docs, templates, and phase specs
  motorcycle-rider-classifier-plan.md   <- Master product plan (11 phases)
  phase-0-corpus-ground-truth.md        <- Corpus and ground truth spec
  phase-0-execution-notes.md            <- Phase 0 execution status
  templates/                            <- CSV templates for ground truth annotation
```

**New rider experiment folders** (to be created in Phase 1):
- `_rider_classification/` — quick PoC for rider/event photo classification
- `_rider_research_ab_test/` — OCR, model comparisons, embeddings, retrieval benchmarks
- `_rider_ab_test/` — rider/motorcycle detection, crops, embeddings, clustering benchmarks

## Build and Test Commands

All desktop commands run from `sherlock/desktop/`:

```bash
# Install frontend dependencies
npm install

# Run frontend tests (Vitest)
npm run test

# Run in dev mode
npm run tauri:dev

# Build release binary (AppImage/DMG/MSI)
npm run tauri:build
```

From `sherlock/desktop/src-tauri/`:

```bash
# Run Rust tests
cargo test

# Format check + lint (also run by pre-commit hook)
cargo fmt --check
cargo clippy -- -D warnings
```

The pre-commit hook at `scripts/pre-commit.sh` runs `cargo fmt --check`, `cargo clippy`, `cargo test`, and `npx vitest run`. Install it with `ln -s ../../scripts/pre-commit.sh .git/hooks/pre-commit`.

**Wayland/NVIDIA workaround:**
```bash
WEBKIT_DISABLE_DMABUF_RENDERER=1 GDK_BACKEND=wayland,x11 npm run tauri:dev
```

## Architecture (Inherited App)

The Frank Sherlock desktop app is a **Tauri v2** app with:

- **Rust backend** (`src-tauri/src/`): handles all filesystem, AI, and database operations via Tauri commands. Key modules:
  - `scan.rs` — 4-phase incremental scanner (discovery → thumbnailing → classification → cleanup), cancellable and resumable
  - `db.rs` — SQLite + FTS5 schema, migrations, and query helpers
  - `face.rs` — ONNX face detection (SCRFD) + recognition (ArcFace) + clustering
  - `classify.rs` — Ollama vision + Surya OCR pipeline with 3-attempt fallback strategy
  - `similarity.rs` — dHash perceptual hashing + Union-Find grouping
  - `thumbnail.rs` — thumbnail generation and image-derived cache
  - `platform/` — OS abstraction (GPU, clipboard, paths)
  - `config.rs` — app data path resolution

- **React frontend** (`src/`): TypeScript + Vite. `App.tsx` is the orchestration shell; feature logic lives in hooks and components. TypeScript types must stay aligned with Rust's `#[serde(rename_all = "camelCase")]` serialization.

- **SQLite** at `~/.local/share/frank_sherlock/db/index.sqlite` — do not edit or reorder existing migrations; add new ones instead.

- **Local AI**: Ollama (qwen2.5vl:7b), Surya OCR (isolated Python venv), ONNX Runtime (SCRFD + ArcFace models downloaded on first use).

## Phase Status and Development Sequence

Current status: **Phase 0 complete** (corpus prepared, ground truth spec defined). Phase 1 experiments have not started.

Mandatory sequence:
1. **Phase 1** — build `_rider_classification/`, `_rider_research_ab_test/`, `_rider_ab_test/` experiment labs and measure results against corpus
2. **Phase 2+** — only after Phase 1 results are in, bootstrap the new product from the Frank Sherlock shell

**Never modify `sherlock/desktop/` for domain behavior before Phase 1 experiments exist.**

## Corpus and Ground Truth

The local corpus lives in `_local/rider_corpus/evt_YYYYMMDD_slug/` (gitignored). Ground truth format:

- `photos.csv` — one row per photo: `photo_id`, `relative_path`, `rider_id`, `sticker_number`, `sticker_visibility`, `view_angle`, `quality_flags`, `split`
- `riders.csv` — one row per rider: `rider_id`, `sticker_number`, `visual_summary`
- `hard_cases.csv` — difficult cases: `case_type`, `expected_challenge`, `review_priority`

Templates are in `docs/templates/`. IDs use stable formats: `rider_<event_slug>_<nnn>`, `photo_<event_slug>_<nnnnn>`, `evt_YYYYMMDD_slug`.

**Never commit** real photos, ground truth files with real data, thumbnails, embeddings, SQLite databases, or generated outputs.

## Coding Conventions

- Rust: use `AppError`/`AppResult` and propagate with `?`. Avoid `.unwrap()` in production code.
- Frontend: keep TypeScript types aligned with Rust `#[serde(rename_all = "camelCase")]`.
- `App.tsx` is orchestration only — feature behavior in dedicated hooks and components.
- For new rider Python experiments: scripts are fine for model comparison speed; production behavior moves to the app stack only after validation.

## Architecture Constraints

- **Local-only**: no cloud APIs for classification, OCR, embeddings, or clustering.
- **Read-only source folders**: never write to scanned photo directories.
- **App data** goes under the app-specific local data directory, never next to user photos.
- **Incremental processing**: avoid reprocessing unchanged files.
- **New rider product** gets a fresh schema — do not carry Frank Sherlock migrations forward.

## Key Design Decisions (Recorded)

- Sticker number is a strong signal but not the sole identity source. Within one event, helmet/clothing/motorcycle do not change.
- Visual embedding model selection should detect local GPU/VRAM and recommend the best available model (same philosophy as Frank Sherlock's GPU detection).
- MVP acceptance targets: cluster purity ≥ 0.90, rider recall ≥ 0.70, manual review rate ≤ 35%.
- Open decisions (to be resolved in Phase 1): dedicated object detector vs. heuristic crop; OCR pipeline for sticker reading; export format for organizing photos by rider.
