<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# timpani Rust API Documentation

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-api-reference
- **Document Status:** Published
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0a | 2026-05-13 | Initial API documentation | Eclipse timpani Team | - |

---

This document describes the gRPC API and Rust module interfaces for timpani's Rust implementation.

## Table of Contents
1. [Overview](#overview)
2. [gRPC Services](#grpc-services)
3. [timpani-o Public API](#timpani-o-public-api)
4. [timpani-n Public API](#timpani-n-public-api)
5. [Common Types](#common-types)
6. [Error Handling](#error-handling)

---

## Overview

timpani Rust replaces the D-Bus communication layer from the C/C++ implementation with gRPC/Protobuf for inter-component communication.

**Architecture:**
```
┌─────────────┐                   ┌─────────────┐
│   Pullpiri  │◄──gRPC/SchedInfo─►│ timpani-o   │
│ Orchestrator│                    │  (Global)   │
└─────────────┘                    └─────┬───────┘
                                         │ gRPC/NodeService
                                  ┌──────┴───────┬───────────┐
                                  │              │           │
                              ┌───▼───┐      ┌───▼───┐  ┌───▼───┐
                              │Node 1 │      │Node 2 │  │Node N │
                              │(T-N)  │      │(T-N)  │  │(T-N)  │
                              └───────┘      └───────┘  └───────┘
```

---

## gRPC Services

### 1. SchedInfoService (Pullpiri ↔ timpani-o)

Defined in: `timpani_rust/timpani-o/proto/schedinfo.proto`

#### SchedInfoService
Allows orchestrators to submit workloads to timpani-o.

**Methods:**
```protobuf
service SchedInfoService {
  // Submit a new workload schedule
  rpc AddSchedInfo (SchedInfo) returns (Response) {}
}
```

**Request: SchedInfo**
```protobuf
message SchedInfo {
  string workload_id = 1;              // Unique workload identifier
  repeated TaskInfo tasks = 2;         // List of tasks to schedule
}

message TaskInfo {
  string name = 1;                     // Task name (max 16 chars)
  int32 priority = 2;                  // RT priority (1-99)
  SchedPolicy policy = 3;              // NORMAL | FIFO | RR
  uint64 cpu_affinity = 4;             // CPU bitmask
  int32 period = 5;                    // Period in μs
  int32 release_time = 6;              // Release offset in μs
  int32 runtime = 7;                   // WCET in μs
  int32 deadline = 8;                  // Deadline in μs
  string node_id = 9;                  // Target node (empty = auto)
  int32 max_dmiss = 10;                // Max consecutive deadline misses
}
```

**Response:**
```protobuf
message Response {
  int32 status = 1;    // 0 = success, non-zero = error code
}
```

#### FaultService
Allows timpani-o to report faults back to the orchestrator.

**Methods:**
```protobuf
service FaultService {
  // Report a fault (e.g., deadline miss)
  rpc NotifyFault (FaultInfo) returns (Response) {}
}
```

**Request: FaultInfo**
```protobuf
message FaultInfo {
  string workload_id = 1;              // Workload where fault occurred
  string node_id = 2;                  // Node reporting the fault
  string task_name = 3;                // Task that faulted
  FaultType type = 4;                  // UNKNOWN | DMISS
}

enum FaultType {
  UNKNOWN = 0;
  DMISS = 1;                           // Deadline miss
}
```

---

### 2. NodeService (timpani-o ↔ timpani-n)

Defined in: `timpani_rust/timpani-n/proto/node_service.proto`

**Methods:**
```protobuf
service NodeService {
  // Retrieve schedule for this node
  rpc GetSchedInfo (NodeSchedRequest) returns (NodeSchedResponse) {}

  // Synchronize start time across all nodes (barrier)
  rpc SyncTimer (SyncRequest) returns (SyncResponse) {}

  // Report a deadline miss
  rpc ReportDMiss (DeadlineMissInfo) returns (NodeResponse) {}
}
```

#### GetSchedInfo
timpani-n calls this at startup to retrieve its task schedule.

**Request: NodeSchedRequest**
```protobuf
message NodeSchedRequest {
  string node_id = 1;    // Node identifier from config
}
```

**Response: NodeSchedResponse**
```protobuf
message NodeSchedResponse {
  string workload_id = 1;              // Active workload ID
  uint64 hyperperiod_us = 2;           // Hyperperiod (LCM of all periods)
  repeated ScheduledTask tasks = 3;    // Tasks assigned to this node
}

message ScheduledTask {
  string name = 1;                     // Task name
  int32  sched_priority = 2;           // RT priority (1-99)
  int32  sched_policy = 3;             // 0=NORMAL, 1=FIFO, 2=RR
  int32  period_us = 4;                // Period in μs
  int32  release_time_us = 5;          // Release offset in μs
  int32  runtime_us = 6;               // WCET in μs
  int32  deadline_us = 7;              // Relative deadline in μs
  uint64 cpu_affinity = 8;             // CPU bitmask
  int32  max_dmiss = 9;                // Max consecutive misses
  string assigned_node = 10;           // Assigned node ID
}
```

#### SyncTimer
Synchronization barrier. All active nodes call this; server responds when all have checked in.

**Request: SyncRequest**
```protobuf
message SyncRequest {
  string node_id = 1;    // Node declaring readiness
}
```

**Response: SyncResponse**
```protobuf
message SyncResponse {
  bool  ack = 1;                       // true = barrier released
  int64 start_time_sec = 2;            // Absolute start time (seconds)
  int64 start_time_nsec = 3;           // Nanoseconds component
}
```

**Behavior:**
- **Blocking:** Call blocks until all active nodes have called `SyncTimer`
- **Late joiner:** If barrier already fired, returns past `start_time` immediately
- **Workload change:** Returns `ABORTED` if workload replaced while waiting

#### ReportDMiss
timpani-n reports deadline misses via this non-blocking call.

**Request: DeadlineMissInfo**
```protobuf
message DeadlineMissInfo {
  string node_id = 1;                  // Reporting node
  string task_name = 2;                // Task that missed deadline
}
```

**Response: NodeResponse**
```protobuf
message NodeResponse {
  int32 status = 1;    // 0 = success
}
```

---

## timpani-o Public API

### GlobalScheduler

**Module:** `timpani_rust/timpani-o/src/scheduler/`

**Purpose:** Distributes real-time tasks across compute nodes.

#### Algorithms

| Algorithm | Description | Use Case |
|-----------|-------------|----------|
| `node_priority` | Assigns tasks to a specific target node first, then spreads overflow | Single-node preference |
| `task_priority` | Greedy scheduling by task priority | Mixed-criticality |
| `best_fit` | Assigns tasks to node with least remaining capacity | Load balancing |

#### Usage

```rust
use timpani_o::scheduler::GlobalScheduler;
use timpani_o::task::Task;
use std::sync::Arc;

// Initialize with node configuration
let scheduler = GlobalScheduler::new(Arc::new(node_config_mgr));

// Schedule tasks
let result = scheduler.schedule(tasks, "node_priority")?;
// Returns: NodeSchedMap (BTreeMap<String, Vec<SchedTask>>)
```

#### Error Types

```rust
pub enum SchedulerError {
    NoNodes,                          // No nodes available
    InsufficientCpus,                 // Not enough CPUs for task
    OverUtilization(String),          // CPU util > 90%
    FeasibilityWarning(String),       // Liu & Layland bound exceeded
}
```

### HyperperiodCalculator

**Module:** `timpani_rust/timpani-o/src/hyperperiod/`

**Purpose:** Computes LCM of task periods and handles GCD-based optimizations.

#### Usage

```rust
use timpani_o::hyperperiod::HyperperiodInfo;

let hp_info = HyperperiodInfo::calculate(&tasks)?;
println!("Hyperperiod: {} μs", hp_info.hyperperiod_us());
```

### Configuration Management

**Module:** `timpani_rust/timpani-o/src/config/`

**Purpose:** Loads `node_configurations.yaml`.

#### Example Config

```yaml
nodes:
  node1:
    cpus: 4
    cpu_ids: [0, 1, 2, 3]
  node2:
    cpus: 8
    cpu_ids: [0, 1, 2, 3, 4, 5, 6, 7]
```

#### Usage

```rust
use timpani_o::config::NodeConfigManager;

let mgr = NodeConfigManager::from_file("node_configurations.yaml")?;
let node_info = mgr.get_node("node1").unwrap();
println!("Node has {} CPUs", node_info.cpus);
```

---

## timpani-n Public API

### NodeClient (gRPC Client)

**Module:** `timpani_rust/timpani-n/src/grpc/`

**Purpose:** gRPC client for communicating with timpani-o.

#### Methods

```rust
impl NodeClient {
    // Connect to timpani-o (with retry)
    pub async fn connect(uri: &str, node_id: &str) -> TimpaniResult<Self>;

    // Fetch schedule at startup
    pub async fn get_sched_info(&self) -> TimpaniResult<NodeSchedResponse>;

    // Sync barrier (blocks until all nodes ready)
    pub async fn sync_timer(&self) -> TimpaniResult<SyncResponse>;

    // Report deadline miss (non-blocking, queued)
    pub fn report_dmiss(&self, task_name: &str) -> TimpaniResult<()>;
}
```

**Key Design Decisions:**

- **D-N-001:** Use `nix` crate over raw libc FFI for type-safe POSIX syscalls
  - Returns typed `Errno` instead of raw -1/errno pairs
  - Linux-specific constraints encoded in type system (e.g., `sched::Policy`, `Signal` enums)
  - Memory-safe with no raw pointer passing
  - Exception: `libc` kept for `SIGRTMIN()` (dynamic value not exposed by nix)

- **D-N-002:** Use `procfs` crate over manual /proc parsing
  - Handles TOCTOU races gracefully (process may disappear mid-scan)
  - Strongly-typed structs for `/proc/<PID>/stat` and `/proc/<PID>/status`
  - Lazy iterator for memory-efficient process table scanning

- **D-N-003:** Use `libbpf-rs` for eBPF integration
  - Official Rust binding maintained by kernel BPF maintainers
  - Type-safe Rust skeletons generated from `.bpf.c` at build time via `libbpf-cargo`
  - Bundles own libbpf via `libbpf-sys` (no version conflict with `/libbpf` git submodule)

- **D-N-004:** Connection retry count is runtime configurable (not compile-time constant)
  - Deployment flexibility: staging nodes may need different timeout than production
  - Configured via `Config::max_retries` field

- **D-N-005:** Shutdown signal handling with `CancellationToken`
  - Uses `tokio_util::sync::CancellationToken` for structured shutdown propagation
  - Signals all async worker tasks (timer loops, BPF poll thread, watchdog)
  - Handles SIGINT/SIGTERM gracefully without missed-signal windows

- **D-N-006:** Use raw libc for `sched_setscheduler` (not nix wrapper)
  - nix 0.29 does not wrap `sched_setscheduler`, `pidfd_open`, or `pidfd_send_signal`
  - Direct libc calls necessary until nix adds support
  - Still type-safe via internal `SchedPolicy` enum and priority validation (0-99)

- **D-N-007:** Single client instance for process lifetime
  - timpani-n is pure client (never hosts gRPC server)
  - Avoids connection overhead and resource leaks

- **D-N-008:** Auto-retry with 1s interval on connection failure
  - Handles transient network issues during startup
  - Prevents tight retry loops that waste CPU
  - `RETRY_INTERVAL_MS = 1000`

- **D-N-009:** `report_dmiss` uses 64-entry MPSC queue to avoid RT loop blocking
  - RT loop never blocks on network I/O (~10ns enqueue time)
  - Queue depth calculation: 5ms miss interval + 1ms round-trip = ~5 steady-state depth
  - 64 entries absorbs ~64ms worth of misses before backpressure
  - Background worker drains queue serially (prevents thundering herd on reconnect)
  - Backpressure: drops notification with warning log if queue full

### Scheduler

**Module:** `timpani_rust/timpani-n/src/sched/`

**Purpose:** Applies Linux scheduling policies via `sched_setscheduler` and `sched_setaffinity`.

#### Supported Policies

- `SCHED_NORMAL` (SCHED_OTHER)
- `SCHED_FIFO` (real-time, fixed priority)
- `SCHED_RR` (real-time, round-robin)
- `SCHED_DEADLINE` (EDF, requires runtime/deadline/period)

#### Usage

```rust
use timpani_n::sched::apply_sched_params;

apply_sched_params(
    pid,
    sched_policy,
    sched_priority,
    cpu_affinity,
    runtime_us,
    deadline_us,
    period_us
)?;
```

### BPF Integration

**Module:** `timpani_rust/timpani-n/src/bpf/`

**Feature Flag:** `bpf` (enabled by default)

**Purpose:** eBPF-based deadline miss detection via `sigwait.bpf.c`.

#### Build Flags

```bash
# Enable BPF (default)
cargo build

# Disable BPF
cargo build --no-default-features

# Enable plot generation (schedstat eBPF events)
cargo build --features plot
```

---

## Common Types

### Task Representation

**timpani-o:**
```rust
pub struct Task {
    pub name: String,
    pub priority: i32,
    pub policy: SchedPolicy,
    pub cpu_affinity: CpuAffinity,
    pub period_us: i32,
    pub release_time_us: i32,
    pub runtime_us: i32,
    pub deadline_us: i32,
    pub node_id: Option<String>,
    pub max_dmiss: i32,
}
```

**timpani-n:**
```rust
pub struct TaskConfig {
    pub name: String,
    pub sched_priority: i32,
    pub sched_policy: i32,
    pub period_us: i32,
    pub release_time_us: i32,
    pub runtime_us: i32,
    pub deadline_us: i32,
    pub cpu_affinity: u64,
    pub max_dmiss: i32,
}
```

### SchedPolicy Enum

```rust
pub enum SchedPolicy {
    Normal = 0,    // SCHED_NORMAL
    Fifo = 1,      // SCHED_FIFO
    Rr = 2,        // SCHED_RR
}
```

### CpuAffinity

```rust
pub enum CpuAffinity {
    Any,                // Run on any CPU
    Mask(u64),          // Bitmask: bit N = CPU N
}
```

---

## Error Handling

### timpani-o Error Types

```rust
// Scheduler errors
pub enum SchedulerError {
    NoNodes,
    InsufficientCpus,
    OverUtilization(String),
    FeasibilityWarning(String),
}

// Config errors
pub enum ConfigError {
    FileNotFound(PathBuf),
    ParseError(String),
    InvalidNodeConfig(String),
}
```

### timpani-n Error Types

```rust
pub enum TimpaniError {
    GrpcError(tonic::Status),
    SchedulerError(String),
    BpfError(String),
    ConfigError(String),
    IoError(std::io::Error),
}

pub type TimpaniResult<T> = Result<T, TimpaniError>;
```

### Error Propagation

Both timpani-o and timpani-n use `anyhow::Result` for application-level errors and `thiserror` for library error types:

```rust
use anyhow::{Context, Result};
use thiserror::Error;

#[derive(Error, Debug)]
#[error("Failed to load config from {path}: {source}")]
pub struct ConfigError {
    path: PathBuf,
    #[source]
    source: std::io::Error,
}
```

---

## Build and Test

### Building

```bash
cd timpani_rust

# Build all crates
cargo build --release

# Build specific crate
cargo build -p timpani-o --release
cargo build -p timpani-n --release

# Build with features
cargo build -p timpani-n --features plot
```

### Testing

```bash
# Run all tests
cargo test

# Run with logging
RUST_LOG=debug cargo test -- --nocapture

# Run specific test
cargo test -p timpani-o scheduler::tests::test_node_priority
```

### Running

```bash
# timpani-o
./target/release/timpani-o \
  --config examples/node_configurations.yaml \
  --listen 0.0.0.0:50051

# timpani-n
./target/release/timpani-n \
  --node-id node1 \
  --timpani-o-uri http://192.168.1.100:50051
```

---

## API Versioning

- **gRPC Package:** `schedinfo.v1`
- **Rust Crate Version:** `0.1.0` (Milestone 1/2)
- **Protobuf Files:** `proto/schedinfo.proto`, `proto/node_service.proto`

Breaking changes will increment the major version and require a new protobuf package (e.g., `schedinfo.v2`).

---

## References

- **Protobuf Definitions:** `timpani_rust/timpani-{o,n}/proto/`
- **Rust Documentation:** Run `cargo doc --open`
- **C++ Reference:** `timpani-o/src/`, `timpani-n/src/`
- **gRPC Guide:** [gRPC.io](https://grpc.io/)
