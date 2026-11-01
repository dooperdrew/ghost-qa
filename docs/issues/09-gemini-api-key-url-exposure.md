---
title: "[Task] Secure Gemini API Key via x-goog-api-key Request Header"
labels: ["agent-task", "security", "ai"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-09`
- **Roadmap Phase:** Phase 3 (Telemetry, Security & Lifecycle)
- **Tier:** `T2 Moderate`
- **Target Model:** `flash`

## Owned Files (Strict Scope)
- `crates/ai/src/lib.rs`

## Invariants & Hard Rules
- NFR-S4: API keys must never be logged, formatted into URLs, or exposed in error messages.

## Root Cause & Evidence
In `crates/ai/src/lib.rs` line 161:
```rust
let url = format!(
    "https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key={api_key}"
);
```
Passing the API key in the URL query parameters causes `reqwest::Error` stringification to expose the key in logs if network errors occur.

## Acceptance Criteria
- [ ] Remove `?key={api_key}` from the request URL.
- [ ] Add `.header("x-goog-api-key", api_key)` to HTTP POST request.
- [ ] Error messages and logs are verified free of raw credential strings.
