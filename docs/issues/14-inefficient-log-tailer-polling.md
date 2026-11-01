---
title: "[Task] Optimize Diagnostic Log File Tailer with Persistent File Handles"
labels: ["agent-task", "performance", "platform"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-14`
- **Roadmap Phase:** Phase 3 (Telemetry, Security & Lifecycle)
- **Tier:** `T2 Moderate`
- **Target Model:** `flash`

## Owned Files (Strict Scope)
- `crates/platform/src/process.rs`

## Invariants & Hard Rules
- Log tailer must handle file rotation, truncations, and multi-process file sharing without pegging CPU.

## Root Cause & Evidence
In `crates/platform/src/process.rs` line 113:
```rust
while !stop_c.load(...) {
    if let Ok(mut file) = File::open(&path) { ... }
    thread::sleep(Duration::from_millis(50));
}
```
`File::open` is called 20 times per second per tailed file, creating filesystem churn.

## Acceptance Criteria
- [ ] Keep the file handle open across iterations.
- [ ] Check file metadata length/inode to detect rotation, re-opening only on truncation/deletion.
- [ ] Log tailing unit test verifies continuous streaming with low CPU.
