---
title: "[Task] Eliminate static mut Data Race in DummyWindowHandle"
labels: ["agent-task", "bug", "probe", "safety"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-04`
- **Roadmap Phase:** Phase 2 (Correctness & Concurrency Safety)
- **Tier:** `T3 Critical`
- **Target Model:** `pro`

## Owned Files (Strict Scope)
- `crates/probe/src/dummy_window.rs`

## Invariants & Hard Rules
- Zero `static mut` without synchronization.
- Thread-safe Win32 instance data association using `GWLP_USERDATA`.

## Root Cause & Evidence
In `crates/probe/src/dummy_window.rs` line 34 & 52:
```rust
static mut GLOBAL_TELEMETRY: Option<*const DummyWindowTelemetry> = None;
```
If multiple tests or probe instances run concurrently, simultaneous writes and reads to `GLOBAL_TELEMETRY` cause data races and undefined behavior.

## Acceptance Criteria
- [ ] `GLOBAL_TELEMETRY` is removed.
- [ ] Telemetry `Arc` or raw pointer is passed via `CreateWindowExW` `lpParam` and bound via `SetWindowLongPtrW(hwnd, GWLP_USERDATA, ...)`.
- [ ] `dummy_wnd_proc` retrieves telemetry safely per-window via `GetWindowLongPtrW`.
- [ ] Multi-threaded test spawning 2 dummy windows simultaneously passes without data races.
