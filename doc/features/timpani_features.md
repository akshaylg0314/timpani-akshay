<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# timpani Feature Specification

**Document Information:**
- **Issuing Author:** LGSI-KarumuriHari(Eclipse timpani Team)
- **Configuration ID:** timpani-feature-spec
- **Document Status:** Draft
- **Last Updated:** 2026-05-14

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0c | 2026-05-14 | Removed implementation status section  | LGSI-KarumuriHari | - |
| 0.0b | 2026-05-13 | Added system block diagram and feature breakdown table | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial feature specification | Eclipse timpani Team | - |

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Feature Breakdown Table](#feature-breakdown-table)
3. [Feature Descriptions](#feature-descriptions)

---

## System Overview

Eclipse timpani is a distributed real-time task orchestration framework consisting of three main components:

- **timpani-o (Orchestrator):** Global scheduler that manages workloads across multiple nodes
- **timpani-n (Node Executor):** Local executor that runs time-triggered tasks with real-time guarantees
- **sample-apps:** Sample applications and workload generators for testing and demonstration

**Note:** For detailed system block diagrams and component architecture, please refer to the [High-Level Design (HLD) documents](../architecture/HLD/).

---

## Feature Breakdown Table

The following table shows the 3-level feature breakdown for Eclipse timpani system components.

### Table 1: timpani System Features

| Level 1 | Level 2 | Level 3 | Descriptions |
|---------|---------|---------|--------------|
| **timpani-o**<br/>(Global Orchestrator) | **Interface Layer** | D-Bus Server (replaced) | Legacy D-Bus interface replaced by gRPC for node communication |
| | | gRPC Server | Modern gRPC service endpoint on port 50054 for Pullpiri and node communication |
| | | Fault Client | gRPC client for reporting deadline misses and fault events to Pullpiri orchestrator |
| | **Core Processing Layer** | SchedInfoService impl | Implementation of gRPC SchedInfo service for receiving and processing workload schedules |
| | | Hyperperiod Manager | Calculates LCM of task periods for hyperperiod determination and schedule validation |
| | | Global Scheduler | Allocates tasks to nodes and CPUs using real-time scheduling algorithms (Rate Monotonic, EDF) |
| | | NodeConfigManager | Loads and manages node hardware specifications from YAML configuration files |
| | **Data Management Layer** | Task Converter | Converts between Protocol Buffer task representations and internal scheduling data structures |
| | | SchedInfo Map | Manages mapping and storage of scheduling information for active workload sets |
| | | Scheduler Utils | Provides feasibility checks, Liu & Layland bounds, and CPU utilization calculations |
| | **Storage Layer** | Schedule State | Maintains current scheduling state and task allocations across nodes |
| | | HyperPeriod Info | Stores calculated hyperperiod information for periodic task sets |
| | | Node Config Files | YAML configuration files containing node hardware specifications and capabilities |
| **timpani-n**<br/>(Node Executor) | **BPF Monitoring** | Scheduler Monitoring | eBPF program (schedstat.bpf.c) tracks scheduler events via tracepoints |
| | | Signal Monitoring | eBPF program (sigwait.bpf.c) monitors signal delivery and deadlines |
| | | BPF Ring Buffer | Kernel-to-userspace data transfer for monitoring statistics |
| | **Core Layer** | Main Controller | Program entry point, coordinates initialization and main execution loop |
| | | Configuration Manager | CLI parsing with Clap, configuration validation, defaults management |
| | | Context Structure | Global runtime state management (internal.h) |
| | **Execution Layer** | Task Manager | Task list management, activation scheduling, state tracking |
| | | RT Scheduler | CPU affinity assignment, RT priority configuration, sched_setattr() syscalls |
| | | Timer Manager | POSIX timer management, periodic activation timing |
| | | Signal Handler | SIGALRM handling, task signal delivery, shutdown signal processing |
| | **Communication Layer** | libtrpc Client | Legacy D-Bus communication client for timpani-o integration |
| | | gRPC Client (Rust) | Modern gRPC client implementation for schedule retrieval and sync |
| | | Schedule Receiver | Receives workload schedules from timpani-o orchestrator |
| | **System Interface** | Linux Scheduler | Integration with SCHED_DEADLINE real-time scheduling policy |
| | | CPU Affinity Control | CPU core assignment and affinity management for tasks |
| | | POSIX Timers | Timer_create, timer_settime for periodic task activation |
| **sample-apps**<br/>(Workload Generator) | **Workload Library** | libttsched | Time-triggered scheduling library for sample applications |
| | | Task Primitives | Task initialization, execution, and termination functions |
| | **Sample Applications** | Periodic Tasks | Configurable periodic workload generators with CPU burn loops |
| | | Aperiodic Tasks | Event-driven workload generators for mixed-criticality testing |
| | | Multi-threaded Apps | Parallel execution workloads for multi-core testing |
| | **Testing Tools** | WCET Analyzer | Worst-Case Execution Time measurement and analysis tools |
| | | Workload Profiler | CPU utilization and response time profiling utilities |
| | | Deadline Monitor | Deadline miss detection and reporting for validation |
| | **Build System** | CMake Configuration | Cross-compilation support for x86_64, aarch64, armhf |
| | | Docker Support | Containerized build environments (Ubuntu, CentOS) |
| | | Integration Scripts | Automated build and test execution scripts |

---

## Feature Descriptions

### timpani-o (Global Orchestrator)

#### Interface Layer
The interface layer provides external communication endpoints for the global orchestrator. The legacy D-Bus protocol has been replaced by modern gRPC for improved performance and type safety.

**Key Features:**
- **D-Bus Server (replaced)**: Legacy interface that has been replaced by gRPC in the Rust implementation
- **gRPC Server**: Modern high-performance RPC server on port 50054 using Tonic framework
- **Fault Client**: Reports deadline misses and fault events to Pullpiri orchestrator

#### Core Processing Layer
The core processing layer implements the main scheduling logic and workload management functionality.

**Key Features:**
- **SchedInfoService impl**: Implements gRPC service for receiving workload schedules from Pullpiri
- **Hyperperiod Manager**: LCM calculation for periodic task sets and schedule validation
- **Global Scheduler**: Rate Monotonic (RM) and Earliest Deadline First (EDF) task allocation algorithms
- **NodeConfigManager**: YAML-based node specification loading and hardware capability management

#### Data Management Layer
Handles data transformation, mapping, and utility functions for scheduling operations.

**Key Features:**
- **Task Converter**: Protocol Buffer to internal data structure conversion and validation
- **SchedInfo Map**: Efficient mapping and lookup of scheduling information for active workloads
- **Scheduler Utils**: Liu & Layland schedulability bounds, feasibility analysis, and utilization calculations

#### Storage Layer
Manages persistent and runtime state storage for scheduling information.

**Key Features:**
- **Schedule State**: Current task allocations, node assignments, and execution state
- **HyperPeriod Info**: Calculated LCM values and hyperperiod metadata for task sets
- **Node Config Files**: YAML configuration files with node hardware specifications (CPU, memory, architecture)

### timpani-n (Node Executor)

#### BPF Monitoring
Provides kernel-level monitoring of scheduler events and signal delivery using eBPF technology.

**Key Features:**
- **Scheduler Monitoring**: Tracks scheduling latency and context switches
- **Signal Monitoring**: Monitors signal delivery timing for deadline detection
- **BPF Ring Buffer**: High-performance kernel-to-userspace data transfer

#### Core Layer
Central coordination and configuration management for the node executor.

**Key Features:**
- **Main Controller**: Initialization, event loop, shutdown coordination
- **Configuration Manager**: Command-line parsing, validation, defaults
- **Context Structure**: Global state, task lists, runtime information

#### Execution Layer
Manages task lifecycle, real-time scheduling, and timer-based activation.

**Key Features:**
- **Task Manager**: Task creation, activation, completion tracking
- **RT Scheduler**: SCHED_DEADLINE policy, CPU affinity, priority assignment
- **Timer Manager**: POSIX timer management for periodic activation
- **Signal Handler**: SIGALRM processing, graceful shutdown

#### Communication Layer
Handles communication with timpani-o orchestrator.

**Key Features:**
- **libtrpc Client** (Legacy): D-Bus-based RPC client
- **gRPC Client** (Rust): Modern gRPC implementation
- **Schedule Receiver**: Workload schedule retrieval and parsing

#### System Interface
Low-level integration with Linux kernel scheduling and timing facilities.

**Key Features:**
- **Linux Scheduler**: SCHED_DEADLINE integration for real-time guarantees
- **CPU Affinity**: Core assignment for task isolation
- **POSIX Timers**: Timer_create/timer_settime for periodic activation

### sample-apps (Workload Generator)

#### Workload Library
Provides reusable components for creating test workloads.

**Key Features:**
- **libttsched**: Time-triggered scheduling primitives
- **Task API**: Initialization, execution, cleanup interfaces
- **Configuration**: Period, deadline, WCET specification

#### Sample Applications
Pre-built workload generators for testing and demonstration.

**Key Features:**
- **Periodic Tasks**: Fixed-period CPU-bound workloads
- **Aperiodic Tasks**: Event-driven sporadic workloads
- **Multi-threaded**: Parallel execution patterns

#### Testing Tools
Analysis and validation utilities for real-time performance.

**Key Features:**
- **WCET Analyzer**: Execution time measurement and statistics
- **Workload Profiler**: CPU usage and timing analysis
- **Deadline Monitor**: Deadline miss detection and logging

#### Build System
Cross-platform build and deployment infrastructure.

**Key Features:**
- **CMake**: Multi-architecture build configuration
- **Docker**: Reproducible build environments
- **CI/CD Integration**: Automated testing and validation

---

## Related Documentation

- [timpani Architecture](../architecture/timpani_architecture.md)
- [timpani-o LLD Documents](../architecture/LLD/timpani-o/)
- [timpani-n LLD Documents](../architecture/LLD/timpani-n/)
- [timpani Requirements](requirements/timpani_requirements.md)
- [API Documentation](../docs/api.md)

---

## References

1. Eclipse timpani Project Documentation
2. Real-Time Systems Design Patterns
3. Liu & Layland Schedulability Analysis
4. eBPF Programming Guide
5. gRPC Protocol Documentation
