---
title: "[Task] Implement Accurate CPU % Telemetry via GetProcessTimes"
labels: ["agent-task", "feature", "platform", "nfr"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-06`
- **Roadmap Phase:** Phase 3 (Telemetry, Security & Lifecycle)
- **Tier:** `T3 Critical`
- **Target Model:** `flash`

## Owned Files (Strict Scope)
- `crates/platform/src/process.rs`

## Invariants & Hard Rules
- Adhere to NFR-P5 and F1.3 requirements for real-time CPU tracking.

## Root Cause & Evidence
In `crates/platform/src/process.rs` line 192:
```rust
ProcessHealthSample {
    timestamp_ms,
    pid: self.pid,
    working_set_bytes,
    cpu_percent: 0.0,
    is_hung,
    exited_code,
}
```
`cpu_percent` is hardcoded to 0.0.

## Acceptance Criteria
- [ ] `MonitoredTargetProcess` samples `GetProcessTimes` on target process handle.
- [ ] Computes normalized delta CPU % across elapsed wall time and CPU logical core count.
- [ ] Unit test verifies non-zero CPU measurement during compute loops.
