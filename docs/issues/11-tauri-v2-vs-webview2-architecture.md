---
title: "[Task] Reconcile Dashboard Architecture (ADR-0002: Edge WebView2 vs Tauri v2)"
labels: ["agent-task", "docs", "architecture"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-11`
- **Roadmap Phase:** Phase 4 (Architecture Alignment & Polish)
- **Tier:** `T2 Moderate`
- **Target Model:** `pro`

## Owned Files (Strict Scope)
- `docs/01-vision.md`
- `docs/02-requirements.md`
- `docs/03-architecture.md`
- `docs/decisions/`
- `crates/ui/src/main.rs`
- `crates/ui/Cargo.toml`

## Invariants & Hard Rules
- Architecture specs and actual code must align 100%.

## Root Cause & Evidence
Requirements (F7) and ADR-0001 state that `ui-qa-ui` is built on Tauri v2. In reality, `crates/ui` is a zero-dependency HTTP server that launches `msedge --app=http://127.0.0.1:<port>`. Tauri was never configured or wired.

## Acceptance Criteria
- [ ] Create `docs/decisions/0002-edge-webview2-app-shell.md` documenting the deliberate zero-dependency Edge WebView2 `--app` shell architecture (or implement pure Tauri v2).
- [ ] Update `docs/01`, `docs/02`, `docs/03`, and `docs/05` to reflect the chosen architecture.
