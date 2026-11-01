---
title: "[Task] Localize CLI User-Facing Terminal Output in en/it"
labels: ["agent-task", "i18n", "cli"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-13`
- **Roadmap Phase:** Phase 4 (Architecture Alignment & Polish)
- **Tier:** `T1 Lean`
- **Target Model:** `flash_lite`

## Owned Files (Strict Scope)
- `crates/cli/src/main.rs`
- `locales/en.json`
- `locales/it.json`

## Invariants & Hard Rules
- NFR-Q2 and AGENTS.md §7: All user-facing terminal output must be loaded from `locales/en.json` and `locales/it.json`.

## Root Cause & Evidence
In `crates/cli/src/main.rs`, over 15 `println!` and `eprintln!` calls print hardcoded English strings. Only `cli.selftest.pass` is loaded from `I18nCatalog`.

## Acceptance Criteria
- [ ] Add CLI message keys to `locales/en.json` and `locales/it.json`.
- [ ] Replace hardcoded strings in `crates/cli/src/main.rs` with `i18n.t(...)` calls.
- [ ] Running `cargo run -p ui-qa-cli -- --locale it self-test` prints localized Italian output.
