---
title: "[Task] Prevent Target Process Orphans using Win32 Job Objects"
labels: ["agent-task", "safety", "platform"]
---

## Task ID & Complexity Tier
- **Issue ID:** `ISSUE-07`
- **Roadmap Phase:** Phase 3 (Telemetry, Security & Lifecycle)
- **Tier:** `T3 Critical`
- **Target Model:** `pro`

## Owned Files (Strict Scope)
- `crates/platform/src/process.rs`

## Invariants & Hard Rules
- NFR-S3: If `ui-qa-tester` crashes or terminates abnormally, any spawned child process must terminate immediately via the OS kernel without leaving orphaned background processes.

## Root Cause & Evidence
Requirement NFR-S3 specifies `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`. In `crates/platform/src/process.rs`, child processes are spawned purely using standard `Command::spawn()`. If the runner process crashes or is killed with SIGKILL, child target `.exe` processes remain orphaned.

## Acceptance Criteria
- [ ] `CreateJobObjectW` creates a job object on target launch.
- [ ] Configures `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`.
- [ ] Assigns child process handle via `AssignProcessToJobObject`.
- [ ] Process cleanup test confirms child terminates upon parent handle closure.
