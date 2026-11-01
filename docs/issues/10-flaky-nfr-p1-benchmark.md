---
title: "[Task] Fix Flaky NFR-P1 Capture Benchmark in Hardening Suite"
labels: ["agent-task", "test", "platform", "nfr"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-10`
- **Roadmap Phase:** Phase 1 (Repository Stabilization)
- **Tier:** `T2 Moderate`
- **Target Model:** `flash`

## Owned Files (Strict Scope)
- `crates/platform/tests/hardening_tests.rs`

## Invariants & Hard Rules
- NFR-P1: Background capture latency must meet `p95 <= 50 ms` under stable conditions.
- Benchmark tests must account for cold-start GDI/DWM initialization.

## Root Cause & Evidence
In `crates/platform/tests/hardening_tests.rs` lines 25–35:
```rust
for _ in 0..10 {
    let cap = capture_hwnd_background(win.hwnd_raw).expect("capture background");
    cap_latencies.push(cap.latency_ms);
}
let p95_cap = cap_latencies[(cap_latencies.len() * 95 / 100).min(cap_latencies.len() - 1)];
```
1. 10 samples with index `9` evaluates the maximum sample (100th percentile / single spike).
2. The cold-start first frame often takes 50–70 ms during concurrent test compilation.
3. No warm-up frames are captured.

## Acceptance Criteria
- [ ] Perform 2 unmeasured warm-up captures to prime DWM allocation.
- [ ] Record 30 captures and calculate true 95th percentile.
- [ ] Test consistently passes in `cargo test --workspace`.
