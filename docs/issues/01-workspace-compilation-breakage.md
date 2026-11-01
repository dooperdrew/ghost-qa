---
title: "[Task] Fix Workspace Compilation Breakage in Cargo.toml, config.rs & cli/main.rs"
labels: ["agent-task", "bug", "build"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-01`
- **Roadmap Phase:** Phase 1 (Repository Stabilization)
- **Tier:** `T1 Lean`
- **Target Model:** `flash`

## Owned Files (Strict Scope)
- `Cargo.toml`
- `crates/core/src/config.rs`
- `crates/cli/src/main.rs`
- `crates/cli/Cargo.toml`

## Invariants & Hard Rules
- `#![forbid(unsafe_code)]` in `crates/core` and `crates/cli`.
- `Cargo.toml` must declare a clean, single `[workspace]` block without duplicate keys or dangling member strings.
- `QaSessionConfig` must be fully well-formed and pass serde roundtrip tests.

## Root Cause & Evidence
1. `Cargo.toml` lines 1–25 have duplicate `resolver = "2"` and dangling strings `"crates/video", "crates/heuristic_sdk"` outside the `members` array.
2. `crates/core/src/config.rs` lines 85–90 has truncated `default_teardown_strategy()` and deleted closing `}` for `struct QaSessionConfig`.
3. `crates/cli/src/main.rs` contains duplicate `Commands::Run` variants and calls `bundle.report.media.push(...)` where `media` does not exist on `QaSessionReport`.

## Acceptance Criteria
- [ ] `Cargo.toml` syntax is valid with single `[workspace]` table.
- [ ] `crates/core/src/config.rs` compiles cleanly and unit tests pass.
- [ ] `crates/cli/src/main.rs` compiles without warnings.
- [ ] `cargo check --workspace --all-targets` exits 0.
