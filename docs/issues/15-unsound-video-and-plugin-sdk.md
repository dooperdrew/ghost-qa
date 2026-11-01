---
title: "[Task] Refactor Video Recording and Heuristic Plugin Architecture"
labels: ["agent-task", "refactor", "plugins"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-15`
- **Roadmap Phase:** Phase 4 (Architecture Alignment & Polish)
- **Tier:** `T2 Moderate`
- **Target Model:** `flash`

## Owned Files (Strict Scope)
- `crates/video/`
- `crates/heuristic_sdk/`
- `Cargo.toml`

## Invariants & Hard Rules
- Avoid native C dependency requirements (`ffmpeg-next`) on standard developer machines.
- Avoid passing raw Rust trait objects (`*mut dyn Trait`) across DLL boundaries.

## Root Cause & Evidence
1. `crates/video/Cargo.toml` pulls in `ffmpeg-next = "6"`, which requires C FFmpeg libraries installed on Windows via pkg-config, while the implementation in `lib.rs` simply launches the external `ffmpeg` executable.
2. `crates/heuristic_sdk/src/lib.rs` attempts dynamic loading using `*mut dyn Heuristic`, which is ABI-unsound in Rust.

## Acceptance Criteria
- [ ] Remove `ffmpeg-next` native C dependency; invoke `ffmpeg` CLI cleanly or use pure-Rust encoder.
- [ ] Redesign plugin interface using C-ABI (`#[repr(C)]` or JSON-over-pipe) or compile-time registration.
- [ ] Crates build cleanly on standard Windows toolchains without external C SDKs.
