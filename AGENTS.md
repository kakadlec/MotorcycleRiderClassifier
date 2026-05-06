# AGENTS.md

## Project Overview

Motorcycle Rider Classifier is a fork of Frank Sherlock. The current codebase still contains the original Frank Sherlock desktop app and research labs, but the product direction is to become a local-first desktop tool for organizing motorcycle event photos by rider.

Use this file as Codex guidance for work in this repository. Prefer preserving the useful platform pieces from Frank Sherlock while replacing the domain-specific image cataloging behavior with rider-focused experiments and, later, a dedicated app workflow.

## Current Repository State

The main inherited app lives in `sherlock/desktop/`:

- Tauri v2 desktop shell.
- Rust backend in `src-tauri/`.
- React + Vite + TypeScript frontend in `src/`.
- SQLite local database and thumbnail cache.
- Local-only AI integrations through Ollama, Surya OCR, and ONNX Runtime.

The root-level experiment folders are inherited from Frank Sherlock and should be treated as reference material:

- `_classification/`: small Python proof of concept for image classification.
- `_research_ab_test/`: broader benchmark lab for model, OCR, audio, video, and cataloging decisions.
- `_face_ab_test/`: benchmark lab for face detection, embeddings, and clustering.

For the motorcycle domain, create new experiment folders instead of modifying the desktop app first:

- `_rider_classification/`: quick PoC for rider-event photo classification.
- `_rider_research_ab_test/`: benchmark lab for sticker reading, OCR/model comparisons, embeddings, and retrieval.
- `_rider_ab_test/`: benchmark lab for rider/motorcycle detection, crops, embeddings, and clustering.

## Product Direction

The target product should organize photos by motorcycle rider using:

- sticker number recognition on the windscreen when visible;
- rider, helmet, clothing, and motorcycle visual similarity;
- manual review for ambiguous groups;
- cluster merge/split workflows;
- local-only processing and storage.

Do not assume the sticker is always visible or reliable. Treat it as a strong signal when present, not as the only identity source.

Within one event, assume the rider's helmet, clothing, motorcycle, and sticker number do not change. The sticker number is unique per event, and an event may span multiple days.

## Development Sequence

Before making functional changes to `sherlock/desktop/`, validate the domain with real data:

1. Build a representative corpus and ground truth.
2. Run the rider-focused experiments.
3. Choose the MVP pipeline based on measured results.
4. Only then bootstrap the desktop app into the new product.

This avoids turning the fork into an accumulation of unrelated Frank Sherlock features.

## Fork And License Rules

- Preserve the fork provenance and Git history.
- Keep the `GPL-3.0-only` license unless a compatible licensing decision is made.
- Preserve attribution to Frank Sherlock and its original author/repository in user-facing project documentation.
- Do not recreate the Git history to hide the origin of the fork.
- If distributing binaries, keep source availability obligations in mind.

## Architecture Principles

- Local-only: do not introduce cloud APIs for classification, OCR, embeddings, or clustering.
- Read-only source folders: never write to scanned target directories.
- App data belongs under the app-specific local data directory, not next to user photos.
- Incremental processing: avoid reprocessing unchanged files.
- Resume and cancellation matter for large event folders.
- Keep OS-specific behavior isolated in platform modules.
- Prefer measurable experiments over speculative architecture.

## Tech Stack

Inherited app:

- Desktop: Tauri v2.
- Backend: Rust.
- Frontend: React + Vite + TypeScript.
- Database: SQLite + FTS5.
- Local AI: Ollama, Surya OCR, ONNX Runtime.
- Target OS: Linux, macOS, Windows.

For new rider experiments, Python scripts are acceptable when they make model comparison faster. Production desktop behavior should eventually move into the app stack when the approach is proven.

## Build And Test

From `sherlock/desktop/`:

```bash
npm install
npm run test
npm run tauri:dev
npm run tauri:build
```

From `sherlock/desktop/src-tauri/`:

```bash
cargo test
```

If Wayland/NVIDIA causes WebKit issues:

```bash
WEBKIT_DISABLE_DMABUF_RENDERER=1 GDK_BACKEND=wayland,x11 npm run tauri:dev
```

## Coding Conventions

- Rust: use existing `AppError`/`AppResult` patterns and propagate errors with `?`.
- Avoid `.unwrap()` in production code.
- Add focused unit tests for new Rust modules.
- Frontend: keep TypeScript types aligned with Rust `#[serde(rename_all = "camelCase")]`.
- Keep `App.tsx` as orchestration; feature behavior should live in dedicated hooks and components.
- Use existing scanner, thumbnail, cache, and platform patterns before inventing new infrastructure.
- For structured data, use proper parsers or typed data structures instead of ad hoc string handling.

## Database Guidance

For the inherited Frank Sherlock app, respect existing migration rules:

- Do not edit or reorder shipped migrations.
- Add new migrations rather than modifying old ones.
- Test fresh and existing databases when changing schema.
- Do not drop data without explicit user intent.

For the new motorcycle product, prefer a clean schema after the bootstrap phase rather than carrying old Frank Sherlock migrations forward.

Expected new-domain entities include:

- `photos`;
- `photo_assets`;
- `rider_detections`;
- `sticker_reads`;
- `visual_embeddings`;
- `rider_clusters`;
- `cluster_members`;
- `manual_labels`.

## What Not To Do

- Do not start by modifying the desktop app before domain experiments exist.
- Do not preserve generic Frank Sherlock features just because they already exist.
- Do not write generated files, thumbnails, embeddings, local databases, or real photo corpora into Git.
- Do not write to user photo directories.
- Do not remove license or attribution information.
- Do not introduce cloud fallback paths.

## Useful Reference Modules

When the desktop phase begins, these inherited modules are likely useful:

- `scan.rs`: incremental scan, move detection, cancellation, resume.
- `thumbnail.rs`: thumbnail generation and image-derived cache.
- `db.rs`: SQLite patterns, migrations, FTS, and job state.
- `config.rs`: app path resolution.
- `runtime.rs` and `platform/gpu.rs`: local hardware detection.
- `platform/`: OS abstraction.
- `similarity.rs`: simple similarity and grouping patterns.
- `face.rs`: reference for embeddings and clustering workflows, not for the final rider domain directly.
