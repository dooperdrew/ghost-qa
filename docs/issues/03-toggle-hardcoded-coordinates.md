---
title: "[Task] Fix Hardcoded (60, 60) Coordinates for UiAction::Toggle"
labels: ["agent-task", "bug", "platform"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-03`
- **Roadmap Phase:** Phase 2 (Correctness & Concurrency Safety)
- **Tier:** `T3 Critical`
- **Target Model:** `pro`

## Owned Files (Strict Scope)
- `crates/platform/src/actuator.rs`
- `crates/core/src/model.rs`

## Invariants & Hard Rules
- Background actions must target the true bounding box center of the interactive element.

## Root Cause & Evidence
In `crates/platform/src/actuator.rs` line 454:
```rust
UiAction::Toggle { .. } => {
    let lparam = LPARAM(((60usize << 16) | 60usize) as isize);
    unsafe {
        let _ = PostMessageW(hwnd, WM_LBUTTONDOWN, WPARAM(MK_LBUTTON_FLAG), lparam);
        let _ = PostMessageW(hwnd, WM_LBUTTONUP, WPARAM(0), lparam);
    }
    Ok(())
}
```
The toggle action hardcodes coordinates `(60, 60)` regardless of where the target checkbox or switch is located.

## Acceptance Criteria
- [ ] `UiAction::Toggle` carries `element_id`, `element_name`, `x`, `y` (or resolves coordinates from target element bounds).
- [ ] First attempts `IUIAutomationTogglePattern::Toggle` via UIA.
- [ ] Falls back to `PostMessageW` targeting the actual control center.
