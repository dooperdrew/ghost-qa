---
title: "[Task] Fix UIA Double-Click Fallthrough in dispatch_action_background"
labels: ["agent-task", "bug", "platform"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-02`
- **Roadmap Phase:** Phase 2 (Correctness & Concurrency Safety)
- **Tier:** `T3 Critical`
- **Target Model:** `pro`

## Owned Files (Strict Scope)
- `crates/platform/src/actuator.rs`

## Invariants & Hard Rules
- `unsafe` permitted only with `// SAFETY:` explanations.
- Never dispatch secondary physical-style clicks when Tier 1 UIA pattern invocation succeeds.

## Root Cause & Evidence
In `crates/platform/src/actuator.rs` line 406:
```rust
let _ = !element_name.trim().is_empty() && try_uia_invoke_by_name(hwnd, element_name);
```
The boolean result of `try_uia_invoke_by_name` is discarded. The code then falls through to line 407–440 and executes `WM_MOUSEMOVE` + `WM_LBUTTONDOWN` + `WM_LBUTTONUP` on the child render widget. This causes controls (e.g. tabs, buttons, checkboxes) to be triggered twice.

## Acceptance Criteria
- [ ] If `try_uia_invoke_by_name` returns `true`, `dispatch_action_background` returns `Ok(())` immediately.
- [ ] Tier 2 `PostMessageW` is used strictly as a fallback when UIA invoke fails or is not supported.
- [ ] Integration test verifies single-event delivery on buttons.
