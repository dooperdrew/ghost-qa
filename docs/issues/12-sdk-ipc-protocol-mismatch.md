---
title: "[Task] Reconcile SDK IPC Protocol (Named Pipes vs Localhost TCP)"
labels: ["agent-task", "docs", "sdk"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-12`
- **Roadmap Phase:** Phase 4 (Architecture Alignment & Polish)
- **Tier:** `T1 Lean`
- **Target Model:** `flash`

## Owned Files (Strict Scope)
- `docs/02-requirements.md`
- `docs/03-architecture.md`
- `crates/sdk/src/lib.rs`

## Invariants & Hard Rules
- Maintain `#![forbid(unsafe_code)]` in `crates/sdk`.

## Root Cause & Evidence
Requirements F3.2 and F9.1 specify `ui-qa-sdk` IPC over Windows Named Pipes (`\\.\pipe\ui-qa-tester-<pid>`). The implementation uses localhost TCP (`TcpListener`).

## Acceptance Criteria
- [ ] Align documentation and implementation (either implement named pipes using `windows` crate without unsafe in SDK, or update specs to specify localhost TCP with loopback authorization).
- [ ] Integration test between `sdk` and `platform` passes cleanly.
