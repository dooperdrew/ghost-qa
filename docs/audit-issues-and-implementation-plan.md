# Comprehensive Repository Audit: Issues, Inconsistencies & Phased Implementation Plan

> **Date:** 2026-11-01  
> **Repository:** `ui-qa-tester` (`https://github.com/DooperDrew/ui-qa-tester`)  
> **Status:** Actionable Backlog & Execution Plan  

---

## Executive Summary

A deep empirical and architectural audit of the `ui-qa-tester` codebase was conducted across all crates (`core`, `platform`, `ai`, `sdk`, `cli`, `ui`, `probe`), tests, scripts, locales, and documentation (`docs/01`–`docs/08`). 

The audit identified **15 concrete issues** spanning five critical areas:
1. **Active Workspace & Compilation Breakage (from incomplete/hallucinated changes)**
2. **Runtime Correctness & Action Injection Bugs**
3. **Safety, Resource Leaks & Concurrency Race Conditions**
4. **Security & Secret Hygiene Violations**
5. **Architectural & Specification Inconsistencies**

Below is the prioritized inventory of dedicated, detailed issues followed by a 4-phase implementation plan structured for autonomous or paired execution.

---

## Detailed Issue Catalog

```
                                  CRITICALITY MATRIX
┌──────────────────────┬──────────────────────────────────────────────────────────────┐
│ Critical (Blocker)   │ ISSUE-01 (Workspace Breakage), ISSUE-02 (UIA Double-Click), │
│                      │ ISSUE-03 (Toggle Hardcoded 60,60), ISSUE-08 (Static Mut UB) │
├──────────────────────┼──────────────────────────────────────────────────────────────┤
│ High (Bug / Leak)    │ ISSUE-04 (Dummy Window Collisions), ISSUE-05 (COM Leak),     │
│                      │ ISSUE-06 (Hardcoded CPU), ISSUE-07 (Job Object Orphan),      │
│                      │ ISSUE-09 (Gemini API Key in URL), ISSUE-10 (Flaky p95 Test) │
├──────────────────────┼──────────────────────────────────────────────────────────────┤
│ Medium (Spec / Arch) │ ISSUE-11 (Tauri v2 vs HTTP), ISSUE-12 (TCP vs Named Pipe),   │
│                      │ ISSUE-13 (CLI i18n Bypass), ISSUE-14 (Inefficient Tailer)    │
├──────────────────────┼──────────────────────────────────────────────────────────────┤
│ Low (Clean-up)       │ ISSUE-15 (Unsound C-Level Spike Crates)                      │
└──────────────────────┴──────────────────────────────────────────────────────────────┘
```

---

### [ISSUE-01] Workspace Compilation Breakage: Corrupted `Cargo.toml`, `config.rs`, and `cli/main.rs`
- **Category:** Build & Workspace Integrity
- **Severity:** `Critical`
- **Tier:** `T1 Lean` | **Target Model:** `flash`
- **Impacted Files:**
  - `Cargo.toml`
  - `crates/core/src/config.rs`
  - `crates/cli/src/main.rs`
  - `crates/cli/Cargo.toml`
- **Evidence:**
  ```toml
  # Cargo.toml: Lines 1-20
  [workspace]
  resolver = "2"
  members = [ ... ]
  resolver = "2"
      "crates/video",
      "crates/heuristic_sdk",
  ```
  ```rust
  // crates/core/src/config.rs: Lines 85-89
  fn default_false() -> bool { false }

      "force_kill".to_string()
  }
  ```
  ```rust
  // crates/cli/src/main.rs: Lines 303 & 487
  if let Some(v) = cfg.enable_video_mode { // error: cfg not yet declared!
  bundle.report.media.push(...)           // error: QaSessionReport has no `media` field!
  ```
- **Root Cause:** A prior conversation turn attempted incremental text edits without verifying compiler output, resulting in syntax errors, duplicate `resolver` declarations, truncated functions in `config.rs`, and a call to a non-existent `media` field in `report.rs`.
- **Concrete Fix:**
  1. Clean `Cargo.toml` workspace root: ensure single valid `members = [...]` array and single `resolver = "2"`.
  2. Restore `default_teardown_strategy() -> String` in `crates/core/src/config.rs` and properly close `QaSessionConfig`.
  3. Remove duplicate `Commands::Run` variant and invalid `media` calls in `crates/cli/src/main.rs`.
  4. Ensure `cargo check --workspace` passes cleanly.
- **Acceptance Criteria:** `cargo check --workspace --all-targets` exits 0.

---

### [ISSUE-02] Double-Click / Duplicate Action Dispatch on UIA Invoke
- **Category:** Runtime Correctness
- **Severity:** `Critical`
- **Tier:** `T3 Critical` | **Target Model:** `pro`
- **Impacted Files:** `crates/platform/src/actuator.rs` (Lines 403–442)
- **Evidence:**
  ```rust
  UiAction::Click { element_name, x, y, .. } => {
      let _ = !element_name.trim().is_empty() && try_uia_invoke_by_name(hwnd, element_name);
      let target_hwnd = find_render_widget_target(hwnd);
      // ... falls through and sends WM_MOUSEMOVE + WM_LBUTTONDOWN + WM_LBUTTONUP anyway!
  ```
- **Root Cause:** The return value of `try_uia_invoke_by_name(hwnd, element_name)` is discarded with `let _ = ...`. Even when UIA Tier 1 invoke succeeds, execution immediately proceeds to Tier 2 (`PostMessageW`), dispatching a physical-style mouse click at coordinates `(x, y)`. This double-clicks toggles (turning them immediately off) or clicks another control at the same coordinate.
- **Concrete Fix:**
  ```rust
  if !element_name.trim().is_empty() && try_uia_invoke_by_name(hwnd, element_name) {
      return Ok(());
  }
  ```
- **Acceptance Criteria:** When UIA invoke succeeds, no subsequent `WM_LBUTTONDOWN` or `WM_LBUTTONUP` messages are enqueued.

---

### [ISSUE-03] Hardcoded (60, 60) Coordinates for `UiAction::Toggle`
- **Category:** Runtime Correctness
- **Severity:** `Critical`
- **Tier:** `T3 Critical` | **Target Model:** `pro`
- **Impacted Files:** `crates/platform/src/actuator.rs` (Lines 453–460)
- **Evidence:**
  ```rust
  UiAction::Toggle { .. } => {
      let lparam = LPARAM(((60usize << 16) | 60usize) as isize);
      // SAFETY: Posting synthetic click to toggle target control.
      unsafe {
          let _ = PostMessageW(hwnd, WM_LBUTTONDOWN, WPARAM(MK_LBUTTON_FLAG), lparam);
          let _ = PostMessageW(hwnd, WM_LBUTTONUP, WPARAM(0), lparam);
      }
      Ok(())
  }
  ```
- **Root Cause:** `UiAction::Toggle` ignores the target element ID and its bounding rectangle entirely, hardcoding `(x=60, y=60)`. Any checkbox or toggle switch located elsewhere on screen fails to receive the action.
- **Concrete Fix:**
  1. Update `UiAction::Toggle` to carry `(x, y)` coordinates or look up the control's bounding box center in client coordinates.
  2. First attempt `IUIAutomationTogglePattern::Toggle` via UIA.
  3. If UIA toggle is unavailable, dispatch `PostMessageW` clicks to the control's true center coordinates.
- **Acceptance Criteria:** `UiAction::Toggle` activates controls located at arbitrary window coordinates.

---

### [ISSUE-04] Data Race and Undefined Behavior via `static mut GLOBAL_TELEMETRY`
- **Category:** Memory Safety & Concurrency
- **Severity:** `Critical`
- **Tier:** `T3 Critical` | **Target Model:** `pro`
- **Impacted Files:** `crates/probe/src/dummy_window.rs` (Lines 34, 52, 172)
- **Evidence:**
  ```rust
  static mut GLOBAL_TELEMETRY: Option<*const DummyWindowTelemetry> = None;
  // ...
  unsafe {
      GLOBAL_TELEMETRY = Some(Arc::as_ptr(&telemetry_clone));
  }
  ```
- **Root Cause:** A mutable static without thread synchronization is used to bridge state into the Win32 window procedure. When tests or probes run concurrently, simultaneous writes and reads result in a data race and undefined behavior.
- **Concrete Fix:**
  1. Remove `GLOBAL_TELEMETRY`.
  2. Pass `Box::into_raw(Box::new(telemetry_clone))` as the `lpParam` argument of `CreateWindowExW`.
  3. Handle `WM_NCCREATE` in `dummy_wnd_proc` to call `SetWindowLongPtrW(hwnd, GWLP_USERDATA, p_telemetry)`.
  4. Query `GetWindowLongPtrW(hwnd, GWLP_USERDATA)` inside message handlers for clean, multi-instance window isolation.
- **Acceptance Criteria:** Zero mutable statics in `crates/probe`; concurrent probe executions operate independently without cross-talk.

---

### [ISSUE-05] COM Reference Count Leak (`CoInitializeEx` without `CoUninitialize`)
- **Category:** Resource Leaks
- **Severity:** `High`
- **Tier:** `T3 Critical` | **Target Model:** `pro`
- **Impacted Files:** `crates/platform/src/actuator.rs` (Lines 110, 468)
- **Evidence:**
  ```rust
  pub fn inspect_window_uia_elements(hwnd_raw: isize) -> Result<Vec<UiElement>, String> {
      // SAFETY: Initializing COM in MTA mode on current thread.
      unsafe {
          let _ = CoInitializeEx(None, COINIT_MULTITHREADED);
      }
      // ... returns without ever calling CoUninitialize!
  ```
- **Root Cause:** `CoInitializeEx` increments the thread's COM apartment reference count on every observation step (potentially dozens of times per QA crawl). The corresponding `CoUninitialize` is never called, violating Win32/COM lifecycle contracts and AGENTS.md §7.
- **Concrete Fix:**
  1. Implement an RAII guard `struct ComGuard;` whose `Drop` calls `unsafe { CoUninitialize(); }` if initialization succeeded.
  2. Use `ComGuard` at the entry point of `inspect_window_uia_elements` and `try_uia_invoke_by_name`.
- **Acceptance Criteria:** Every `CoInitializeEx` call has an RAII-guaranteed `CoUninitialize`.

---

### [ISSUE-06] Process CPU Health Sample Hardcoded to 0.0%
- **Category:** Requirements & Telemetry Defect
- **Severity:** `High`
- **Tier:** `T3 Critical` | **Target Model:** `flash`
- **Impacted Files:** `crates/platform/src/process.rs` (Line 192)
- **Evidence:**
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
- **Root Cause:** Requirement F1.3 and NFR-P5 mandate CPU utilization tracking to catch infinite compute loops and frame-rate drops. The implementation hardcodes `0.0`.
- **Concrete Fix:**
  1. Store the previous `(Instant, FILETIME_kernel, FILETIME_user)` sample inside `MonitoredTargetProcess`.
  2. Query `GetProcessTimes` on the open process handle.
  3. Compute $\Delta \text{CPU} = \frac{\Delta \text{kernel} + \Delta \text{user}}{\Delta \text{wall} \times \text{cores}} \times 100\%$.
- **Acceptance Criteria:** `sample_health()` reports non-zero CPU percentages under active workload.

---

### [ISSUE-07] Child Process Orphan Risk: Missing Win32 Job Object
- **Category:** Process Lifecycle & Safety
- **Severity:** `High`
- **Tier:** `T3 Critical` | **Target Model:** `pro`
- **Impacted Files:** `crates/platform/src/process.rs` (Lines 28–88)
- **Evidence:** `Win32_System_JobObjects` is enabled in `Cargo.toml`, but `CreateJobObjectW` and `SetInformationJobObject` are never invoked in `spawn_exe`.
- **Root Cause:** Requirement NFR-S3 states: "If ui-qa-tester crashes or is aborted, any child target process spawned by the runner must be cleanly terminated via Win32 Job Object `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`". Currently, abnormal termination leaves spawned child `.exe` processes running indefinitely.
- **Concrete Fix:**
  1. On target launch, create a Win32 Job Object using `CreateJobObjectW`.
  2. Set `JOBOBJECT_EXTENDED_LIMIT_INFORMATION` with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`.
  3. Assign the spawned process handle to the job object with `AssignProcessToJobObject`.
- **Acceptance Criteria:** Killing `ui-qa-cli` abruptly automatically terminates all descendant target processes.

---

### [ISSUE-08] Test Concurrency Collisions: Static Window Titles & Classes
- **Category:** Test Hygiene & Reliability
- **Severity:** `High`
- **Tier:** `T2 Moderate` | **Target Model:** `flash`
- **Impacted Files:**
  - `crates/platform/src/selftest_window.rs` (Line 48, 66)
  - `crates/probe/src/dummy_window.rs` (Line 65, 83)
  - `crates/platform/src/orchestrator.rs` (Line 78)
- **Evidence:** Both `SelfTestWindow` and `DummyWindowHandle` use hardcoded class names (`UiQaSelfTestHardwareWindowClass`) and window titles (`Hardware Controller Demo (Target App)`).
- **Root Cause:** When `cargo test --workspace -- --test-threads=auto` runs, integration tests spawn windows with identical titles. `list_top_level_windows().find(...)` attaches to the wrong window across test threads, violating AGENTS.md §7.
- **Concrete Fix:**
  1. Append a random UUID or process/thread ID to the window class name and title (e.g. `format!("UiQaSelfTestHardwareWindowClass-{}", Uuid::new_v4())`).
  2. Store and query exact `HWND` rather than fuzzy title matching during automated test sessions.
- **Acceptance Criteria:** `cargo test --workspace -- --test-threads=auto` runs concurrently without cross-test window interference.

---

### [ISSUE-09] Secret Hygiene Violation: Gemini API Key in Request URL
- **Category:** Security & Privacy
- **Severity:** `High`
- **Tier:** `T2 Moderate` | **Target Model:** `flash`
- **Impacted Files:** `crates/ai/src/lib.rs` (Line 161)
- **Evidence:**
  ```rust
  let url = format!(
      "https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key={api_key}"
  );
  ```
- **Root Cause:** Passing API credentials in query parameters exposes keys in HTTP client logs, proxy logs, and error strings (e.g. `reqwest::Error` stringifies the full URL on connection errors), violating NFR-S4 and AGENTS.md §7.
- **Concrete Fix:**
  1. Remove `?key={api_key}` from the URL string.
  2. Add `.header("x-goog-api-key", api_key)` to the request builder.
- **Acceptance Criteria:** Gemini API keys are never embedded in URL query parameters.

---

### [ISSUE-10] Flaky NFR-P1 Capture Benchmark in Hardening Suite
- **Category:** QA & Verification Gate
- **Severity:** `High`
- **Tier:** `T2 Moderate` | **Target Model:** `flash`
- **Impacted Files:** `crates/platform/tests/hardening_tests.rs` (Lines 25–35)
- **Evidence:**
  ```rust
  for _ in 0..10 {
      let cap = capture_hwnd_background(win.hwnd_raw).expect("capture background");
      cap_latencies.push(cap.latency_ms);
  }
  cap_latencies.sort_by(...);
  let p95_cap = cap_latencies[(cap_latencies.len() * 95 / 100).min(cap_latencies.len() - 1)];
  assert!(p95_cap <= 50.0, "NFR-P1 budget violated: p95 capture latency {p95_cap:.2} ms > 50 ms");
  ```
- **Root Cause:**
  1. With only 10 iterations, index `10 * 95 / 100 = 9` is the *maximum* (100th percentile) sample.
  2. Cold-start initialization (first GDI/DWM allocation) spikes to 40–70 ms on busy CI/dev machines.
  3. Zero warm-up frames are captured prior to recording.
- **Concrete Fix:**
  1. Perform 2 unrecorded warm-up captures to prime DWM buffers.
  2. Sample 30 iterations and compute true 95th percentile.
- **Acceptance Criteria:** `hardening_tests` reliably passes under concurrent workspace load.

---

### [ISSUE-11] Architecture Discrepancy: Tauri v2 vs Custom HTTP/WebView2 Shell
- **Category:** Documentation & Architecture Alignment
- **Severity:** `Medium`
- **Tier:** `T2 Moderate` | **Target Model:** `pro`
- **Impacted Files:**
  - `docs/01-vision.md`, `docs/02-requirements.md` (F7), `docs/03-architecture.md`
  - `docs/decisions/0001-rust-tauri-v2-uia-wgc-architecture.md`
  - `crates/ui/src/main.rs`, `crates/ui/Cargo.toml`
- **Root Cause:** Architectural specifications declare `ui-qa-ui` as a Tauri v2 application. However, `crates/ui` is implemented as a lightweight HTTP server serving `dashboard.html` over `TcpListener` and spawning `msedge --app=http://127.0.0.1:<port>`.
- **Options / Resolution:**
  - *Option A (Recommended for simplicity & low footprint):* Formalize the Edge WebView2 `--app` shell architecture in an ADR (`0002-edge-webview2-app-shell.md`), updating requirements to reflect the zero-dependency webview shell that requires no Tauri build toolchain.
  - *Option B (Pure Tauri v2):* Add `tauri` and `tauri-build` to `crates/ui`, configure `tauri.conf.json`, and expose commands via `tauri::generate_handler![]`.
- **Acceptance Criteria:** Documentation, ADRs, and implementation agree 100% on the dashboard runtime architecture.

---

### [ISSUE-12] SDK Protocol Inconsistency: Named Pipes vs Localhost TCP
- **Category:** Specification Consistency
- **Severity:** `Medium`
- **Tier:** `T1 Lean` | **Target Model:** `flash`
- **Impacted Files:**
  - `docs/02-requirements.md` (F3.2, F9.1)
  - `docs/03-architecture.md` (§2.3)
  - `crates/sdk/src/lib.rs` (Lines 10, 63)
- **Root Cause:** Docs state that `ui-qa-sdk` uses Windows Named Pipes (`\\.\pipe\ui-qa-tester-<pid>`). The code implements a localhost `TcpListener` on `127.0.0.1:0`.
- **Resolution:** Either update `crates/sdk` to use `windows::Win32::System::Pipes`, or update requirements and architecture docs to specify localhost TCP with loopback authorization.
- **Acceptance Criteria:** SDK IPC mechanism is uniformly documented and tested.

---

### [ISSUE-13] Hardcoded English Strings in CLI (Bypassing `i18n`)
- **Category:** Internationalization & Code Quality
- **Severity:** `Medium`
- **Tier:** `T1 Lean` | **Target Model:** `flash_lite`
- **Impacted Files:** `crates/cli/src/main.rs` (Lines 219, 234–237, 263, 274, 281, 293, 310, 405–417, 443–465)
- **Root Cause:** NFR-Q2 and AGENTS.md §7 mandate that user-facing strings go through `locales/en.json` and `locales/it.json`. In `crates/cli`, only a single string (`cli.selftest.pass`) uses `i18n.t()`; all other terminal outputs are hardcoded in English.
- **Concrete Fix:** Add missing CLI message keys to `locales/en.json` and `locales/it.json` and route all CLI user feedback through `i18n.t(...)`.
- **Acceptance Criteria:** `ui-qa-cli --locale it run ...` outputs translated messages.

---

### [ISSUE-14] Inefficient 50ms File Re-Opening in Log Tailers
- **Category:** Performance & Resource Management
- **Severity:** `Medium`
- **Tier:** `T2 Moderate` | **Target Model:** `flash`
- **Impacted Files:** `crates/platform/src/process.rs` (Lines 110–142)
- **Root Cause:** In `start_file_tailers`, `File::open(&path)` is executed in every loop iteration (20 times per second per file). This causes unnecessary filesystem churn and fails if the target app opens the file without `FILE_SHARE_READ`.
- **Concrete Fix:** Keep the `File` handle open across iterations; detect log rotation by checking inode/metadata length, re-opening only when the file is truncated or replaced.
- **Acceptance Criteria:** Log tailer maintains a persistent handle while smoothly handling rotation.

---

### [ISSUE-15] Unsound C-Level Video & Heuristic SDK Implementations
- **Category:** Code Quality & Soundness
- **Severity:** `Low`
- **Tier:** `T2 Moderate` | **Target Model:** `flash`
- **Impacted Files:**
  - `crates/video/Cargo.toml` & `crates/video/src/lib.rs`
  - `crates/heuristic_sdk/Cargo.toml` & `crates/heuristic_sdk/src/lib.rs`
- **Root Cause:**
  - `crates/video/Cargo.toml` lists `ffmpeg-next = "6"` (requiring native C FFmpeg libraries on Windows), yet `lib.rs` simply launches `Command::new("ffmpeg")`.
  - `crates/heuristic_sdk/src/lib.rs` passes raw trait objects `*mut dyn Heuristic` across `.dll` boundaries, which is ABI-unsound in Rust due to unstandardized vtables and separate DLL allocators.
- **Concrete Fix:**
  1. Remove `ffmpeg-next` from `crates/video/Cargo.toml` and rely purely on the CLI process invocation or pure-Rust encoder.
  2. Replace dynamic library loading in `crates/heuristic_sdk` with a C-ABI plugin interface (`#[repr(C)]` struct of function pointers and JSON string buffers) or compile-time registry.
- **Acceptance Criteria:** Crates compile cleanly without external C library prerequisites.

---

## Phased Implementation Roadmap

```mermaid
flowchart TD
    subgraph P1["Phase 1: Repository Stabilization & Verification Gate"]
        I01["ISSUE-01: Clean Cargo.toml, config.rs & cli/main.rs"]
        I10["ISSUE-10: Fix Flaky Hardening Benchmark"]
        G1["Fast & Full Verification Gate Passing"]
        I01 --> G1
        I10 --> G1
    end

    subgraph P2["Phase 2: Correctness & Concurrency Safety"]
        I02["ISSUE-02: Fix UIA Double-Click Fallthrough"]
        I03["ISSUE-03: True Coordinates for UiAction::Toggle"]
        I04["ISSUE-04: Replace static mut with GWLP_USERDATA"]
        I05["ISSUE-05: RAII ComGuard for CoUninitialize"]
        I08["ISSUE-08: UUID-Isolated Windows for Parallel Tests"]
        P1 --> P2
    end

    subgraph P3["Phase 3: Telemetry, Security & Process Lifecycle"]
        I06["ISSUE-06: Accurate CPU % via GetProcessTimes"]
        I07["ISSUE-07: Win32 Job Object Kill-On-Close"]
        I09["ISSUE-09: Gemini x-goog-api-key Header"]
        I14["ISSUE-14: Persistent File Tailer Handle"]
        P2 --> P3
    end

    subgraph P4["Phase 4: Architecture Alignment & Polish"]
        I11["ISSUE-11: ADR for Edge WebView2 Shell vs Tauri"]
        I12["ISSUE-12: SDK IPC Documentation/Implementation Sync"]
        I13["ISSUE-13: CLI String Localization in en/it"]
        I15["ISSUE-15: Clean Video & Plugin SDK Architecture"]
        P3 --> P4
    end
```

---

### Phase 1: Repository Stabilization & Verification Gate
**Goal:** Restore the repository to a clean, 100% building state with passing tests.
1. **Task 1.1:** Restore `Cargo.toml` workspace members and dependency definitions.
2. **Task 1.2:** Restore `crates/core/src/config.rs` (`default_teardown_strategy`, proper struct closing).
3. **Task 1.3:** Fix `crates/cli/src/main.rs` duplicate `Commands::Run` and syntax errors.
4. **Task 1.4:** Update `crates/platform/tests/hardening_tests.rs` with warm-up captures and 30-sample p95.
5. **Gate Check:** Run `powershell -ExecutionPolicy Bypass -File .\scripts\verify-gate.ps1 -Mode Full`.

### Phase 2: Correctness & Concurrency Safety
**Goal:** Eliminate input injection bugs, race conditions, and COM leaks.
1. **Task 2.1:** Fix `try_uia_invoke_by_name` return handling in `actuator.rs` to prevent double-clicking.
2. **Task 2.2:** Update `UiAction::Toggle` to use targeted coordinates and UIA toggle pattern.
3. **Task 2.3:** Refactor `crates/probe/src/dummy_window.rs` to use `GWLP_USERDATA` instead of `static mut`.
4. **Task 2.4:** Add RAII `ComGuard` in `actuator.rs` ensuring `CoUninitialize` on all paths.
5. **Task 2.5:** Isolate test window class names with UUIDs in `selftest_window.rs`.
6. **Gate Check:** Run `cargo test --workspace -- --test-threads=auto`.

### Phase 3: Telemetry, Security & Process Lifecycle
**Goal:** Implement true CPU sampling, orphan process protection, and credential hygiene.
1. **Task 3.1:** Implement `GetProcessTimes` delta calculation in `crates/platform/src/process.rs`.
2. **Task 3.2:** Attach Win32 Job Object with `KILL_ON_JOB_CLOSE` in `spawn_exe`.
3. **Task 3.3:** Move Gemini API key to `x-goog-api-key` header in `crates/ai/src/lib.rs`.
4. **Task 3.4:** Optimize log tailer with persistent file handle in `process.rs`.
5. **Gate Check:** Run self-test CLI with synthetic load.

### Phase 4: Architecture Alignment & Polish
**Goal:** Resolve architectural discrepancies and complete internationalization.
1. **Task 4.1:** Document the Edge WebView2 `--app` shell in ADR-0002 or wire pure Tauri v2 manifests.
2. **Task 4.2:** Reconcile SDK named pipe specification vs TCP implementation.
3. **Task 4.3:** Extract all hardcoded strings in `crates/cli/src/main.rs` into `locales/en.json` and `locales/it.json`.
4. **Task 4.4:** Refactor `crates/video` to use external `ffmpeg` CLI cleanly without native C deps.
5. **Gate Check:** Complete `M5.4` / release gate verification.
