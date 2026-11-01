---
title: "[Task] Resolve Parallel Test Window Collisions with Unique UUID Classes"
labels: ["agent-task", "bug", "test", "concurrency"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-08`
- **Roadmap Phase:** Phase 2 (Correctness & Concurrency Safety)
- **Tier:** `T2 Moderate`
- **Target Model:** `flash`

## Owned Files (Strict Scope)
- `crates/platform/src/selftest_window.rs`
- `crates/probe/src/dummy_window.rs`
- `crates/platform/src/orchestrator.rs`

## Invariants & Hard Rules
- AGENTS.md §7: "Tests touching global OS resources (named pipes, temp log files, dummy windows) must use unique UUID names and timeout protection to prevent test hangs."

## Root Cause & Evidence
`SelfTestWindow` and `DummyWindowHandle` use hardcoded window class names (`UiQaSelfTestHardwareWindowClass`) and titles (`Hardware Controller Demo (Target App)`). Running `cargo test --workspace` with multiple test threads causes concurrent tests to find each other's windows via `list_top_level_windows()`.

## Acceptance Criteria
- [ ] Window class names and titles incorporate a unique UUID suffix per spawn instance.
- [ ] Tests and orchestrator reference exact `HWND` handles rather than fuzzy string matching.
- [ ] Concurrent execution of self-test suites succeeds without window collisions.
