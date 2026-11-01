# ADR 0002 — Zero-Elevation Edge WebView2 App Shell for Live QA Dashboard

- **Status:** Accepted
- **Date:** 2026-11-01
- **Deciders:** User, Lead Agent (Antigravity)
- **Supersedes / Amends:** ADR 0001 (regarding `ui-qa-ui` runtime packaging)

## Context
ADR 0001 initially proposed bundling `crates/ui` as a full Tauri v2 desktop application.
During production implementation of `ui-qa-ui` (Milestones 4, 8, 9), two major technical challenges arose with Tauri v2 packaging in developer environments:
1. Full Tauri v2 builds require `tauri-build`, `wry`, and extensive C++ build toolchains on Windows, increasing build times and executable bundle sizes (`> 45 MB`).
2. Every modern Windows 10/11 workstation natively bundles **Microsoft Edge WebView2 (`msedge.exe`)**, which supports the `--app=http://127.0.0.1:<port>` headless application mode natively with zero extra installation overhead.

## Decision
We adopt a **Zero-Elevation Edge WebView2 App Shell**:
- `crates/ui` runs a lightweight, zero-dependency embedded HTTP/REST server on an ephemeral loopback port (`127.0.0.1:<port>`).
- It serves the single-file, highly-optimized reactive `dashboard.html` (`include_str!("dashboard.html")`).
- It launches the native Windows Edge standalone window using `start msedge --app=http://127.0.0.1:<port>` (falling back to default browser on non-standard setups).
- When developers or CI run headless audits (`--headless-check`), it binds to `127.0.0.1:0` and exits immediately without spawning UI windows.

## Consequences
- **Easier:**
  - Build times for `ui-qa-ui` drop from `~45s` to `< 2s`.
  - Zero external C++ or Node.js toolchain required to compile the UI.
  - The entire binary package remains tiny (`~4.5 MB` portable executable).
  - All modern CSS, Canvas bounding-box overlays, and SVG graphs work with full Chromium performance and hardware acceleration.
- **Follow-up:**
  - Update `docs/01-vision.md`, `docs/02-requirements.md` (F7), `docs/03-architecture.md`, and `docs/05-roadmap.md` to reflect the Edge WebView2 `--app` shell architecture.
