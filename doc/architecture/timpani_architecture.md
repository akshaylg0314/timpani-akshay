<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# TIMPANI System Architecture

**Document Version:** 1.0
**Last Updated:** May 12, 2026
**Status:** Living Document

---

## System Overview

TIMPANI is a **distributed real-time task orchestration framework** designed for time-triggered systems. It consists of two primary components:

- **Timpani-O (Orchestrator):** Global scheduler that manages workloads across multiple nodes
- **Timpani-N (Node):** Local executor that runs time-triggered tasks with real-time guarantees

---

## Component Architecture

```mermaid
graph TB
    subgraph "Timpani-O (Global Orchestrator)"
        O1[Global Scheduler]
        O2[Hyperperiod Manager]
        O3[Node Configuration Manager]
        O4[SchedInfo Service]
        O5[Fault Service Client]
        O6[gRPC Server]
    end

    subgraph "Timpani-N (Node Executor)"
        N1[Time Trigger Core]
        N2[Task Management]
        N3[Real-Time Scheduler]
        N4[eBPF Monitoring]
        N5[Signal Handlers]
        N6[gRPC Client]
    end

    subgraph "External Systems"
        E1[Sample Applications]
        E2[Fault Manager]
    end

    O1 --> O2
    O1 --> O3
    O4 --> O1
    O5 --> E2

    O6 <-->|gRPC| N6

    N1 --> N2
    N1 --> N3
    N1 --> N5
    N4 --> N1

    N2 --> E1
    N4 --> O6

    style O1 fill:#e3f2fd
    style N1 fill:#e8f5e9
    style O6 fill:#fff3e0
    style N6 fill:#fff3e0
```

---

## Timpani-O Components

| Component | Responsibility | Implementation |
|-----------|---------------|----------------|
| **Global Scheduler** | Workload scheduling, feasibility analysis | C++ → Rust ✅ |
| **Hyperperiod Manager** | LCM calculation, cycle management | C++ → Rust ✅ |
| **Node Configuration Manager** | Multi-node configuration | C++ → Rust ✅ |
| **SchedInfo Service** | Schedule distribution via gRPC | C++ → Rust ✅ |
| **Fault Service Client** | Deadline miss reporting | C++ → Rust ✅ |
| **gRPC Server** | Node communication (port 50054) | D-Bus → gRPC ✅ |

**Detailed Documentation:** [HLD/timpani-o/](HLD/timpani-o/)

---

## Timpani-N Components

| Component | Responsibility | Implementation |
|-----------|---------------|----------------|
| **Time Trigger Core** | Event loop, hyperperiod coordination | C → Rust 🔄 |
| **Task Management** | Task lifecycle, activation scheduling | C → Rust ⏸️ |
| **Real-Time Scheduler** | CPU affinity, SCHED_FIFO priority | C → Rust ⏸️ |
| **eBPF Monitoring** | Deadline miss detection (kernel) | C → Rust ⏸️ |
| **Signal Handlers** | SIGALRM, task activation signals | C → Rust ⏸️ |
| **Configuration** | CLI parsing, validation | C → Rust ✅ |
| **gRPC Client** | Communication with Timpani-O | libtrpc → gRPC 🔄 |

**Detailed Documentation:** [HLD/timpani-n/](HLD/timpani-n/)

**Legend:** ✅ Complete | 🔄 In Progress | ⏸️ Not Started

---

## Communication Flow

```mermaid
sequenceDiagram
    participant App as Sample Apps
    participant TN as Timpani-N
    participant TO as Timpani-O
    participant FM as Fault Manager

    Note over TO: Startup Phase
    TO->>TO: Load node configurations
    TO->>TO: Calculate global schedule

    Note over TN: Initialization Phase
    TN->>TO: GetSchedInfo(node_id)
    TO-->>TN: SchedInfo (tasks, hyperperiod)
    TN->>TN: Initialize task list
    TN->>TN: Load eBPF programs

    Note over TN,TO: Synchronization Phase
    TN->>TO: SyncTimer(node_id)
    TO-->>TN: Sync start time
    TN->>TN: Start timers

    Note over TN,App: Runtime Phase
    loop Every Hyperperiod
        TN->>TN: Hyperperiod tick
        TN->>App: Activate tasks (SIGALRM)
        App->>App: Execute task logic
        TN->>TN: eBPF: Monitor deadlines
    end

    Note over TN,FM: Fault Handling
    TN->>TO: ReportDeadlineMiss(task_name)
    TO->>FM: Forward fault event
```

---

## Technology Stack

### Legacy (C/C++)
- **Communication:** D-Bus + libtrpc (custom serialization)
- **Build System:** CMake
- **Monitoring:** libbpf (eBPF)
- **Concurrency:** epoll event loop

### Rust Migration
- **Communication:** gRPC (Tonic) + Protobuf
- **Build System:** Cargo
- **Async Runtime:** Tokio
- **Monitoring:** aya (eBPF in Rust, planned)
- **CLI:** Clap
- **Logging:** tracing

---

## Deployment Architecture

```mermaid
graph LR
    subgraph "Node 1"
        N1[Timpani-N]
        A1[App Tasks]
        N1 -.->|monitors| A1
    end

    subgraph "Node 2"
        N2[Timpani-N]
        A2[App Tasks]
        N2 -.->|monitors| A2
    end

    subgraph "Orchestration Node"
        TO[Timpani-O]
        FM[Fault Manager]
    end

    N1 <-->|gRPC<br/>:50054| TO
    N2 <-->|gRPC<br/>:50054| TO
    TO <-->|gRPC| FM

    style TO fill:#e3f2fd
    style N1 fill:#e8f5e9
    style N2 fill:#e8f5e9
```

---

## Key Design Patterns

### 1. Time-Triggered Architecture
- **Hyperperiod:** LCM of all task periods
- **Cyclic Scheduling:** Tasks activated at fixed intervals
- **Deadline Monitoring:** eBPF tracks rt_sigtimedwait syscalls

### 2. Distributed Coordination
- **Centralized Scheduling:** Timpani-O computes global schedule
- **Decentralized Execution:** Timpani-N executes local schedule
- **Synchronization:** Coordinated start time across nodes

### 3. Fault Tolerance
- **Deadline Miss Detection:** eBPF monitors at kernel level
- **Fault Reporting:** gRPC streaming from nodes to orchestrator
- **Fault Management:** Integration with external fault manager

---

## Migration Status

| Milestone | Component | Status | Documentation |
|-----------|-----------|--------|---------------|
| **M1** | Timpani-O | ✅ Complete | [HLD/timpani-o/](HLD/timpani-o/) |
| **M2** | Timpani-N | 🔄 Partial | [HLD/timpani-n/](HLD/timpani-n/) |
| **M3** | gRPC Integration | 🔄 In Progress | [grpc_architecture.md](grpc_architecture.md) |

---

## References

- **Component HLD:** [HLD/timpani-o/](HLD/timpani-o/), [HLD/timpani-n/](HLD/timpani-n/)
- **gRPC Architecture:** [grpc_architecture.md](grpc_architecture.md)
- **API Documentation:** [../docs/api.md](../docs/api.md)
- **Getting Started:** [../docs/getting-started.md](../docs/getting-started.md)

---

**Document Version:** 1.0
**Verified Against:** Component HLD documents, source code (timpani_rust/, timpani-n/, timpani-o/)

