---
title: "[Task] Fix COM Apartment Leak via RAII ComGuard in actuator.rs"
labels: ["agent-task", "bug", "platform", "resource-leak"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-05`
- **Roadmap Phase:** Phase 2 (Correctness & Concurrency Safety)
- **Tier:** `T3 Critical`
- **Target Model:** `pro`

## Owned Files (Strict Scope)
- `crates/platform/src/actuator.rs`

## Invariants & Hard Rules
- Every `CoInitializeEx` must have a corresponding `CoUninitialize` executed via RAII.
- Respect COM thread apartment contracts.

## Root Cause & Evidence
In `crates/platform/src/actuator.rs` line 110 & 468:
```rust
unsafe {
    let _ = CoInitializeEx(None, COINIT_MULTITHREADED);
}
```
`CoInitializeEx` is called in `inspect_window_uia_elements` and `try_uia_invoke_by_name` on every step. `CoUninitialize` is never called, leaking thread-local COM state and apartment reference counts.

## Acceptance Criteria
- [ ] RAII `ComGuard` struct calls `CoUninitialize` on drop when initialization succeeded.
- [ ] All functions in `actuator.rs` initialize and clean up COM properly.
