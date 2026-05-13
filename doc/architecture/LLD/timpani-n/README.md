<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# timpani-n Low-Level Design (LLD) Documentation

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-n-lld-index
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD documentation set | Eclipse timpani Team | - |

---

**Project:** Eclipse Timpani - Real-Time Task Orchestration Framework
**Component:** timpani-n (Node Executor)
**Migration:** C → Rust (In Progress - Initialization Phase Only)
**Status:** 🔄 Milestone 2 In Progress
**Document Set Version:** 1.0
**Last Updated:** May 12, 2026

---

## Overview

This directory contains 10 Low-Level Design (LLD) documents for timpani-n (node executor) components. **Unlike timpani-o**, these documents are primarily **AS-IS focused** because the Rust implementation is still in early development (only initialization/configuration complete).

### Document Structure
- **AS-IS (C Implementation):** Comprehensive documentation from `timpani-n/src/` (legacy C code)
- **WILL-BE (Rust Implementation):** Limited to what's actually implemented in `timpani_rust/timpani-n/` (config, CLI, initialization structure only)
- **Status Markers:**
  - ✅ Complete in Rust
  - 🔄 Partially implemented
  - ⏸️ Not started (planned)

---

## Document Index

### Core System Components

| # | Component | C Status | Rust Status | Description |
|---|-----------|----------|-------------|-------------|
| [01](01-initialization-main.md) | **Initialization & Main** | ✅ Complete | 🔄 Partial | Entry point, CLI parsing, initialization flow |
| [02](02-configuration-management.md) | **Configuration Management** | ✅ Complete | ✅ Complete | Config parsing, validation, defaults |
| [03](03-time-trigger-core.md) | **Time Trigger Core** | ✅ Complete | ⏸️ Not Started | Event loop, hyperperiod, timer management |

### Task & Scheduling

| # | Component | C Status | Rust Status | Description |
|---|-----------|----------|-------------|-------------|
| [04](04-task-management.md) | **Task Management** | ✅ Complete | ⏸️ Not Started | Task list, activation scheduling, lifecycle |
| [05](05-realtime-scheduling.md) | **Real-Time Scheduling** | ✅ Complete | ⏸️ Not Started | CPU affinity, RT priority, `sched_setattr()` |
| [06](06-signal-handling.md) | **Signal Handling** | ✅ Complete | ⏸️ Not Started | `SIGALRM`, `rt_sigtimedwait()`, deadline detection |

### Monitoring & Communication

| # | Component | C Status | Rust Status | Description |
|---|-----------|----------|-------------|-------------|
| [07](07-ebpf-monitoring.md) | **eBPF Monitoring** | ✅ Complete | ⏸️ Not Started | `sigwait.bpf.c`, `schedstat.bpf.c`, ring buffer events |
| [08](08-communication-libtrpc.md) | **Communication (gRPC)** | ✅ Complete | ✅ Complete | D-Bus → gRPC, `NodeClient`, schedule retrieval |

### Support Components

| # | Component | C Status | Rust Status | Description |
|---|-----------|----------|-------------|-------------|
| [09](09-resource-management.md) | **Resource Management** | ✅ Complete | ⏸️ Not Started | Cleanup, global state, graceful shutdown |
| [10](10-data-structures.md) | **Data Structures** | ✅ Complete | 🔄 Partial | `context`, `time_trigger`, `task_info` |

---

## Current Implementation Status

### ✅ **Fully Implemented in Rust**
- ✅ **CLI Parsing** (clap-based argument parsing)
- ✅ **Configuration** (Config struct, validation, defaults)
- ✅ **Logging** (tracing-based with multiple levels)
- ✅ **Error Handling** (TimpaniError enum, structured errors)
- ✅ **Build System** (Cargo, build.rs for proto compilation)
- ✅ **gRPC Communication** (NodeClient, GetSchedInfo, SyncTimer, ReportDMiss)

### 🔄 **Partially Implemented in Rust**
- 🔄 **Initialization Flow** (structure exists, runtime loop not implemented)
- 🔄 **Context Management** (data structures defined, initialization TBD)

### ⏸️ **Not Yet Started in Rust**
- ⏸️ **Time-Triggered Execution**
- ⏸️ **Real-Time Scheduling** (CPU affinity, RT priority)
- ⏸️ **Signal Handling** (SIGALRM, rt_sigtimedwait)
- ⏸️ **eBPF Integration** (BPF program loading, ring buffer polling)
- ⏸️ **Hyperperiod Management**
- ⏸️ **Task Execution Loop**

---

## Key Differences from timpani-o LLD

| Aspect | timpani-o LLD | timpani-n LLD |
|--------|---------------|---------------|
| **Rust Status** | ✅ Complete (M1) | 🔄 Initialization only (M2 in progress) |
| **Focus** | AS-IS vs WILL-BE comparison | Primarily AS-IS (C documentation) |
| **Component Source** | `component-specifications.md` | Architecture docs + source code analysis |
| **WILL-BE Sections** | Comprehensive Rust code | Limited to config/CLI only |
| **Verification** | Against completed Rust impl | Against C implementation primarily |

---

## timpani-n Architecture

### System Role
timpani-n is the **node executor** in the distributed Timpani system:
- **Receives** scheduled tasks from timpani-o (global orchestrator)
- **Executes** time-triggered tasks with real-time guarantees
- **Monitors** task execution via eBPF
- **Reports** deadline misses back to timpani-o

### High-Level Flow

```
timpani-o (Orchestrator)
  ↓ (gRPC: GetSchedInfo, SyncTimer, ReportDMiss)
timpani-n (Node Executor)
  ↓ (Load eBPF programs)
Linux Kernel (eBPF hooks)
  ↓ (Signal tasks)
Task Processes (exprocs)
  ↓ (Ring buffer events)
timpani-n (Deadline monitoring)
  ↓ (Report deadline miss via gRPC)
timpani-o → Fault Manager
```

---

## Technology Stack

### C Implementation (Legacy)
- **Language:** C (ISO C11)
- **Build:** CMake
- **eBPF:** libbpf, CO-RE (Compile Once, Run Everywhere)
- **Communication:** libtrpc (D-Bus over TCP)
- **Monitoring:** Ring buffer, tracepoints
- **Dependencies:** libsystemd, libelf, libyaml

### Rust Implementation (In Progress)
- **Language:** Rust 1.70+
- **Build:** Cargo
- **Async:** Tokio ✅
- **CLI:** clap ✅
- **Logging:** tracing ✅
- **Errors:** thiserror, anyhow ✅
- **Communication:** Tonic (gRPC) ✅
- **Protobuf:** prost ✅
- **Planned:** aya (eBPF)

---

## Document Conventions

### AS-IS (C Implementation)
- **Comprehensive:** Full documentation based on actual C code
- **Source:** `timpani-n/src/*.c`, `doc/architecture/timpani-n/`
- **Verified:** Against legacy implementation

### WILL-BE (Rust Implementation)
- **Limited:** Only what's actually implemented
- **Status Tags:**
  - ✅ **Implemented:** Code exists and works
  - 🔄 **Partial:** Structure exists, logic TBD
  - ⏸️ **Planned:** Not yet started, design TBD
  - 📋 **Design Phase:** Architecture defined, no code yet

### Code Examples
- **C Code:** Marked with `c` language tag
- **Rust Code:** Marked with `rust` language tag
- **Pseudo-Code:** Marked with `text` for design concepts

---

## Reading Guide

### For C Implementation Understanding
Start with these to understand the legacy system:
1. [03 - Time Trigger Core](03-time-trigger-core.md) - Main execution loop
2. [07 - eBPF Monitoring](07-ebpf-monitoring.md) - Deadline detection mechanism
3. [08 - Communication](08-communication-libtrpc.md) - Interaction with timpani-o

### For Rust Migration Status
Check these to see what's been ported:
1. [01 - Initialization](01-initialization-main.md) - Entry point (partial)
2. [02 - Configuration](02-configuration-management.md) - Config system (complete)
3. [10 - Data Structures](10-data-structures.md) - Type definitions (partial)

### For Architecture Understanding
1. [03 - Time Trigger Core](03-time-trigger-core.md) - Hyperperiod concept
2. [04 - Task Management](04-task-management.md) - Task activation
3. [06 - Signal Handling](06-signal-handling.md) - Time-triggered signaling

---

## Authenticated Source Documents

### Legacy C Documentation

| Document | Path | Purpose |
|----------|------|---------|
| **Architecture** | `doc/architecture/timpani-n/architecture.md` | System architecture and components |
| **Block Diagrams** | `doc/architecture/timpani-n/block-diagram.md` | Component relationships |
| **Flow Diagrams** | `doc/architecture/timpani-n/flow-diagram.md` | Execution sequences |
| **README** | `doc/architecture/timpani-n/README.md` | Quick start and overview |

### C Implementation

| Source | Path | Purpose |
|--------|------|---------|
| **Main** | `timpani-n/src/main.c` | Entry point and main loop |
| **Core** | `timpani-n/src/core.c` | Event processing, epoll loop |
| **Config** | `timpani-n/src/config.c` | CLI parsing, validation |
| **Hyperperiod** | `timpani-n/src/hyperperiod.c` | LCM calculation, timer setup |
| **Task** | `timpani-n/src/task.c` | Task list management |
| **Sched** | `timpani-n/src/sched.c` | CPU affinity, RT priority |
| **Signal** | `timpani-n/src/signal.c` | Signal handlers |
| **BPF** | `timpani-n/src/sigwait.bpf.c`, `schedstat.bpf.c` | eBPF programs |
| **TRPC** | `timpani-n/src/trpc.c` | D-Bus communication |
| **Cleanup** | `timpani-n/src/cleanup.c` | Resource cleanup |

### Rust Implementation (Partial)

| Source | Path | Status |
|--------|------|--------|
| **Main** | `timpani_rust/timpani-n/src/main.rs` | ✅ Entry point |
| **Config** | `timpani_rust/timpani-n/src/config/mod.rs` | ✅ Complete |
| **Lib** | `timpani_rust/timpani-n/src/lib.rs` | 🔄 Structure only |
| **Context** | `timpani_rust/timpani-n/src/context/mod.rs` | ⏸️ Planned |
| **gRPC** | `timpani_rust/timpani-n/src/grpc/mod.rs` | ⏸️ Planned |
| **Sched** | `timpani_rust/timpani-n/src/sched/mod.rs` | ⏸️ Planned |
| **Signal** | `timpani_rust/timpani-n/src/signal/mod.rs` | ⏸️ Planned |

---

## Terminology

| Term | Definition |
|------|------------|
| **timpani-n** | Node executor - runs on each compute node |
| **timpani-o** | Global orchestrator - distributes tasks to nodes |
| **Time-Triggered** | Tasks activated by timer signals, not events |
| **Hyperperiod** | LCM of all task periods (smallest repeating window) |
| **eBPF** | Extended Berkeley Packet Filter (kernel monitoring) |
| **libtrpc** | Custom D-Bus RPC library for Timpani communication |
| **exprocs** | Example task processes used for testing |
| **SIGALRM** | Alarm signal used for timer-based activation |
| **rt_sigtimedwait()** | System call for waiting on real-time signals |
| **Ring Buffer** | Kernel data structure for eBPF event delivery |
| **CPU Affinity** | Binding a task to specific CPU cores |
| **RT Priority** | Real-time priority (1-99 for SCHED_FIFO/RR) |

---


## Important Notes

### Documentation Purpose
These LLD documents serve as:
1. **Reference** for the legacy C implementation
2. **Migration Guide** for Rust developers
3. **Comparison** showing C vs Rust approaches (when implemented)
4. **Design Specification** for incomplete Rust features

### AS-IS Focus Rationale
- **Rust implementation is incomplete** (initialization phase only)
- **C code is the source of truth** for behavior
- **Will-Be sections will expand** as Rust implementation progresses
- **Documents will be updated** as each component is migrated

### Verification Status
- **AS-IS sections:** ✅ Verified against C source code
- **WILL-BE sections:** ✅ Verified against Rust code where it exists
- **Planned sections:** 📋 Design only, no verification possible yet

---

**Document Set Version:** 1.0
**Status:** 🔄 In Progress (2/10 components have Rust implementation)
**Last Review:** May 12, 2026
**Next Update:** After M2 completion (Rust runtime loop implementation)

---

## Feedback & Updates

These documents will be updated as the Rust migration progresses:
- **After each component migration:** Update corresponding LLD with WILL-BE section
- **After major design decisions:** Add design decision rationale
- **After testing:** Add test coverage notes
- **After M2 completion:** Comprehensive review and update

**Contact:** Timpani Development Team
**Repository:** Eclipse Timpani GitHub
