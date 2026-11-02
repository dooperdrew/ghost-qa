# 01 — Vision: GhostQA 👻

> Status: **APPROVED** | Official Project Name: **GhostQA** (`ghost-qa`)

## Problem
Developing Windows desktop applications that interact with physical connected hardware introduces a painful testing bottleneck:
1. **Manual UI + Hardware Verification Fatigue:** Every code change requires launching the app, manually clicking through tabs/sliders/buttons, watching hardware diagnostic logs (`stdout`, `stderr`, serial files) to confirm the device responded, and visually checking for layout regressions.
2. **Focus-Stealing Automation Tools:** Traditional Windows automation tools (WinAppDriver, FlaUI, Pywinauto, PyAutoGUI) seize the developer's physical mouse cursor and keyboard focus via `SendInput`. You cannot code, join meetings, or use your PC while tests are running. Moreover, they fail on modern GPU-rendered canvases (`egui`, `iced`, `Slint`, `Tauri`, `Flutter`).
3. **Silent Hardware & UX Defects:** A button click might turn green in the UI while emitting a serial communication timeout in `stderr` 200 ms later, or a label might clip/overlap at a certain window size—issues that traditional pass/fail assertions completely miss.

## Goal
Build **GhostQA 👻** — an autonomous, invisible Windows Desktop and Hardware-in-the-Loop (HIL) QA Agent and Live Dashboard. Like a phantom, GhostQA operates hands-free in the background without stealing the user's mouse or keyboard focus, explores and tests applications autonomously, correlates every UI interaction with real-time hardware logs and process telemetry, critiques visual layout and UX via Multimodal AI (Gemini 2.5 Flash or local VLMs), and outputs actionable, agent-ready fix prompts and GitHub Issues.

## Target users
- **Solo developers and engineering teams** building Windows desktop applications in Rust (`iced`, `egui`, `Slint`) or native Win32 that control or monitor connected hardware (USB/HID peripherals, serial sensors, audio/MIDI gear, embedded controllers) on the same workstation where they actively code.

## Guiding principles (in priority order)
1. **Hands-free & Non-intrusive first:** Never hijack the developer's physical mouse cursor or active window focus during normal operation. Capture windows in the background (`Windows.Graphics.Capture` / `PrintWindow`) and drive controls via UIA patterns, direct `HWND` message injection (`PostMessageW`), or the optional SDK IPC bridge.
2. **Evidence-backed UI ↔ Hardware correlation:** Every UI action is bracketed by a before/after visual frame, UIA tree snapshot, process health reading (CPU, RAM, hang status), and a time-correlated slice of `stdout`/`stderr`/diagnostic log output.
3. **Actionable AI UX & Visual Critique:** Don't just report "test failed"—spot clipped text, overlapping widgets, confusing hierarchy, missing loading/disabled states, and unhandled hardware errors, and output concrete fix suggestions that an AI coding agent (Antigravity, Copilot, Claude, Cursor) can immediately apply.
4. **Zero-code black-box by default, deep white-box on demand:** Work out-of-the-box on any compiled `.exe` binary, while offering a tiny optional Rust SDK crate (`ui-qa-sdk`) that `egui`/`iced`/`Slint` apps can include for 100% deterministic widget inspection and synthetic event injection.
5. **Dual AI Backend Freedom:** Support both cloud multimodal models (**Google Gemini API**) for deep visual/UX reasoning and local **OpenAI-compatible Vision endpoints** (Ollama, LM Studio, vLLM) for offline/zero-cost runs.

## Non-goals (v1)
- **Kernel-mode drivers or DLL code injection:** We will not install kernel drivers, global system hooks (`SetWindowsHookEx`), or inject DLLs via `CreateRemoteThread` into target processes (avoids EDR/antivirus false positives).
- **Cross-platform macOS/Linux target capture in v1:** While `crates/core` is 100% OS-agnostic, v1 platform capture and UIA automation target **Windows 10 / 11 (x64)**.
- **Direct hardware bus electrical probing:** Hardware verification in v1 operates by correlating application `stdout`/`stderr`, structured diagnostic log files, and process exit/health states rather than bit-banging raw USB/I2C analyzers directly.
