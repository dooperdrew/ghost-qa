# Consolidated Cross-Project QA Learnings & Engine Guidelines

> **Consolidated on**: 2026-11-01  
> **Source Projects**:
> - [MagicBridge](https://github.com/DooperDrew/magicbridge) (`egui` / `wgpu` 0.31 native desktop daemon)
> - [SonyBridge](https://github.com/DooperDrew/sonybridge) (`Tauri v2` / `WebView2` hybrid desktop application)
> - [The Dognut Corner](https://github.com/DooperDrew/thedognutcorner) (`Next.js 16` / `React 19` full-stack web application)
> - [ui-qa-tester](https://github.com/DooperDrew/ui-qa-tester) (Autonomous Windows UI & Hardware-in-the-Loop QA Agent)

---

## 1. Executive Summary

Over October and November 2026, three distinct production applications were developed and tested across the Windows ecosystem using **`ui-qa-tester`**:
1. A **low-level hardware remapping daemon and native GUI** (`MagicBridge`).
2. A **Tauri v2 Bluetooth RFCOMM audio & ANC controller** (`SonyBridge`).
3. A **multi-tenant, multi-ledger financial inventory platform** (`The Dognut Corner`).

Field audits, real hardware trials (Apple A1843 Magic Keyboard, Sony WH-1000XM3 headphones), and owner reviews revealed critical QA blind spots that headless unit test runners and naive UI crawlers consistently missed. This document consolidates these hard-won lessons into actionable rules for `ui-qa-tester` and downstream application teams.

---

## 2. Core QA Findings & Engine Rules

### 2.1 The "Tofu" & Missing Font Glyph Blind Spot
- **What happened (`MagicBridge` C1, C9):** `ui-qa-tester` reported 0 findings on a screen where tab navigation icons and arrow keys rendered as blank square replacement glyphs (`□` / `\u{FFFD}`). The app compiled cleanly and AccessKit reported control elements, but the fonts lacked glyph coverage.
- **Rule for `ui-qa-tester`:** Added `VIS-GLYPH-TOFU` deterministic heuristic in `crates/core/src/vision_heuristics.rs`. The crawler scans all element names and text nodes for replacement characters (`\u{FFFD}`) and placeholder squares (`□`), immediately raising a High-severity visual bug.
- **Rule for Application Developers:** Always bundle explicit font fallback definitions (`egui::FontDefinitions`, `Segoe UI Symbol`, `Noto Sans Symbols`) instead of relying solely on default system sans fonts.

### 2.2 Default Launch Window Geometry vs. Artificial Resizing
- **What happened (`MagicBridge` C2):** Testing a GUI at an arbitrary or maximized window size reported clean layouts, but when the owner launched the `.exe` at its default `860 px` width, right-edge elements (numpad enter, status clusters) were visibly clipped.
- **Rule for `ui-qa-tester`:** The orchestrator must capture and test screens at the target's **natural default startup window bounds** before any resizing or scrolling.
- **Rule for Application Developers:** Never assume desktop windows are maximized. Always wrap horizontal widget strips in responsive containers (`egui::ScrollArea`, `iced::widget::scrollable`, CSS container queries).

### 2.3 Raw Unmapped Pre-Remap Hardware Truth vs. Synthesised Telemetry
- **What happened (`MagicBridge` B1–B4):** The in-app hardware QA tool evaluated keypresses from `egui` text events *after* Windows layout remapping and synthesized scancodes from lookup tables (`default_scancode_and_extended(vk)`). Consequently, remapped keys could never be verified, numpad keys collided with main-row keys, and reports displayed fabricated data (violating *Claims Follow Evidence*).
- **Rule for `ui-qa-tester`:** Ground hardware assertions strictly in **raw unmapped physical events** (`scancode`, `LLKHF_EXTENDED`, device handle) captured before translation. Never synthesize measured values from defaults.
- **Rule for Application Developers:** Low-level hooks (`WH_KEYBOARD_LL`) must run in headless background daemons, never inside GUI event loops (`winit`/`egui`), to avoid Windows message timeout lockups and PC freezes.

### 2.4 Headless Invariant Passing vs. Dark-Mode Visual Regressions
- **What happened (`The Dognut Corner` #26, #29):** 60/60 automated unit and database tests passed with zero errors, but an opaque `#ccc` SVG cursor overlay in Recharts completely blinded users by obscuring chart bars in dark mode.
- **Rule for `ui-qa-tester`:** Mathematical unit suites must always be paired with **real visual frame capture and color contrast analysis**. The contrast heuristic in `vision_heuristics.rs` uses extreme percentile sampling (`0.5%` and `99.5%`) to evaluate micro-labels and dark-mode contrast.
- **Rule for Application Developers:** Never certify a UI task as done without visual screenshot evidence across both dark and light modes.

### 2.5 Multi-HWND & Asynchronous UIA Hydration
- **What happened (`SonyBridge` / Tauri v2):** Embedded Chromium / WebView2 frames (`Chrome_RenderWidgetHostHWND`) defer accessibility tree hydration until queried, and ignore click messages sent to the parent outer window.
- **Rule for `ui-qa-tester`:** In `actuator.rs`, the actuator detects `Chrome_RenderWidgetHostHWND`, warms up UIA with a brief 250ms hydration window, and translates screen points via `ScreenToClient(render_hwnd)` before dispatching `PostMessageW`.

### 2.6 Active Tab & Toggle Idempotency
- **What happened (`MagicBridge` / `SonyBridge` RFC):** Re-clicking an already-active tab or toggle button produces zero visual pixel diff (`dHash == 0`) and zero hardware logs, causing the dead button detector (`UX-DEAD-BTN`) to flag false-positive UX friction warnings.
- **Rule for `ui-qa-tester`:** In `actuator.rs` and `crawler.rs`, the crawler queries `UIA_SelectionItemPatternId` and `UIA_TogglePatternId` to verify `is_selected`. If a control is already selected or idempotent, zero-change transitions are suppressed from dead-button warnings.

### 2.7 Aggressive Process-Tree Teardown for Tray Daemons
- **What happened (`MagicBridge` / `SonyBridge` / `ui-qa-tester`):** Hardware controller apps minimize to the Windows system tray on close instead of exiting. When a new test or release build was executed, the file was locked by running background processes (`Access is denied (os error 5)`).
- **Rule for `ui-qa-tester`:** Implemented `teardown_strategy = "force_kill"` in `process.rs` using `taskkill /F /T /PID` to terminate the complete process tree on session end.

---

## 3. Cross-Project QA Integration Matrix

| Capability / Check | MagicBridge (Native Rust) | SonyBridge (Tauri v2) | The Dognut Corner (Next.js 16) | ui-qa-tester Native Engine |
| :--- | :--- | :--- | :--- | :--- |
| **Window Origin Calibration** | DWM frame bounds calibration (`DwmGetWindowAttribute`) | Parent-to-Child HWND translation | Browser viewport & scroll offset | `actuator.rs` (`DWMWA_EXTENDED_FRAME_BOUNDS`) |
| **Input Simulation** | `PostMessageW` (client coords) | `PostMessageW` (`ScreenToClient`) | Synthetic DOM MouseEvent / Playwright | 3-Tier non-intrusive actuation |
| **Hardware Correlation** | Named pipes & raw HID packets | Bluetooth RFCOMM / WinRT audio status | Stripe / Barcode / File I/O | `correlator.rs` (`[-50ms, +1500ms]` window) |
| **Missing Glyph Detection** | `egui` missing glyph square `□` | Web font fallback failure | UTF-8 replacement character `\u{FFFD}` | `vision_heuristics.rs` (`VIS-GLYPH-TOFU`) |
| **Action Trace Replay** | `action-trace.json` | `action-trace.json` | `action-trace.json` | `ui-qa-cli replay --trace <path>` |
| **CI/CD Quality Gate** | `--fail-on high` | `--fail-on high` | `--fail-on high` | Non-zero exit code on severe findings |

---

## 4. Verification & Ongoing Governance
1. **Rule of Evidence**: A feature or bug fix is "complete" only when an end-to-end call path from physical event or UI interaction exists and is verified with empirical screenshots or logs.
2. **Release Hygiene**: Release artifacts must always be staged into isolated directories (`dist/v<Version>/`) and published to GitHub Releases alongside checksum manifests.
3. **Continuous Watch**: Teams should run `ui-qa-cli watch --config qa-session.toml` during active feature development to capture build-to-build delta reports (`delta.md`) before merging PRs.
