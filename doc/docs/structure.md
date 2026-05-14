
<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# Project Structure

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-doc-structure
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0c | 2026-05-14 | Added scope legends to timpani-o and timpani-n block diagrams | LGSI-KarumuriHari | - |
| 0.0b | 2026-05-13 | Added HLD section and features/requirements documentation | LGSI-KarumuriHari | - |
| 0.0a | 2026-05-13 | Initial structure documentation | Eclipse timpani Team | - |

---

This document describes the current structure of the timpani repository. All files and folders listed here are considered stable and will remain untouched in the future, except for the `timpani_rust` folder, which will be the sole focus of ongoing development.

---

## timpani System Block Diagrams

### timpani-o System Block Diagram

```mermaid
graph TB
    subgraph "External Systems"
        PICCOLO[Piccolo Orchestrator]
        ADMIN[System Administrator]
    end

    subgraph "Distributed Nodes"
        NODE1[timpani-n Node 1]
        NODE2[timpani-n Node 2]
        NODEN[timpani-n Node N]
    end

    subgraph "timpani-o Global"
        subgraph "Interface Layer"
            DBUS_SRV[D-Bus Server<br/>replaced by gRPC]
            GRPC_SRV[gRPC Server<br/>SchedInfoService]
            FAULT_CLI[Fault Client<br/>gRPC to Piccolo]
        end

        subgraph "Core Processing Layer"
            SCHEDINFO[SchedInfoServiceImpl]
            HYPER[HyperperiodManager]
            GLOBAL[GlobalScheduler]
            NODECONFIG[NodeConfigManager]
            CLI[CLI/Config]
        end

        subgraph "Data Management Layer"
            TASKCONV[Task Converter]
            SCHEDMAP[SchedInfoMap]
            SCHEDUTIL[Scheduler Utils]
        end

        subgraph "Storage Layer"
            SCHEDSTATE[Schedule State]
            HYPERINFO[Hyperperiod Info]
            NODEFILES[Node Config Files]
        end
    end

    subgraph Legend[" "]
        L1["timpani-o (Our Scope)"]
        L2["timpani-n Nodes (Our Scope)"]
        L3["gRPC Communication (Our Scope)"]
        L4["External Systems"]
    end

    PICCOLO -->|gRPC SchedInfo| GRPC_SRV
    ADMIN -->|CLI Config| CLI

    GRPC_SRV --> SCHEDINFO
    DBUS_SRV -.->|legacy| SCHEDINFO

    SCHEDINFO --> HYPER
    SCHEDINFO --> GLOBAL
    SCHEDINFO --> TASKCONV

    CLI --> NODECONFIG
    NODECONFIG --> NODEFILES

    HYPER --> HYPERINFO
    GLOBAL --> SCHEDUTIL
    GLOBAL --> SCHEDMAP

    TASKCONV --> SCHEDMAP
    SCHEDMAP --> SCHEDSTATE

    FAULT_CLI -->|gRPC FaultNotify| PICCOLO

    GRPC_SRV -->|Deadline Miss| NODE1
    GRPC_SRV -->|Deadline Miss| NODE2
    GRPC_SRV -->|Deadline Miss| NODEN

    NODE1 -->|libtrpc Schedule| GRPC_SRV
    NODE2 -->|libtrpc Schedule| GRPC_SRV
    NODEN -->|libtrpc Schedule| GRPC_SRV

    NODE1 -.->|Deadline Miss| FAULT_CLI
    NODE2 -.->|Deadline Miss| FAULT_CLI
    NODEN -.->|Deadline Miss| FAULT_CLI

    style PICCOLO fill:#f5f5f5,stroke:#757575,stroke-width:2px
    style ADMIN fill:#f5f5f5,stroke:#757575,stroke-width:2px
    style NODE1 fill:#e8f5e9,stroke:#388e3c,stroke-width:3px
    style NODE2 fill:#e8f5e9,stroke:#388e3c,stroke-width:3px
    style NODEN fill:#e8f5e9,stroke:#388e3c,stroke-width:3px
    style GRPC_SRV fill:#fff3e0,stroke:#f57c00,stroke-width:3px
    style DBUS_SRV fill:#d3d3d3,stroke:#757575,stroke-width:2px
    style FAULT_CLI fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style SCHEDINFO fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style HYPER fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style GLOBAL fill:#e3f2fd,stroke:#1976d2,stroke-width:3px
    style NODECONFIG fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style CLI fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style TASKCONV fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style SCHEDMAP fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style SCHEDUTIL fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style SCHEDSTATE fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style HYPERINFO fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style NODEFILES fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style L1 fill:#e3f2fd,stroke:#1976d2,stroke-width:3px
    style L2 fill:#e8f5e9,stroke:#388e3c,stroke-width:3px
    style L3 fill:#fff3e0,stroke:#f57c00,stroke-width:3px
    style L4 fill:#f5f5f5,stroke:#757575,stroke-width:2px
```

### timpani-n System Block Diagram

```mermaid
graph TB
    subgraph "Linux Kernel"
        SCHED[Scheduling Events<br/>tracepoints]
        SYSCALL[System Calls<br/>sigtimedwait]
    end

    subgraph "External Systems"
        SAMPLE[Sample Applications<br/>Execution Tasks]
        TIMPANIO[timpani-o<br/>Global Scheduler]
    end

    subgraph "timpani-n (time-trigger)"
        subgraph "BPF Monitoring"
            SCHEDSTAT[schedstat.bpf.c<br/>Scheduler Monitoring]
            SIGWAIT[sigwait.bpf.c<br/>Signal Monitoring]
            RINGBUF[BPF Ring Buffer]
        end

        subgraph "Core Layer"
            MAIN[main.c<br/>Main Controller]
            CONFIG[config.c<br/>Configuration Manager]
            CONTEXT[Context Structure<br/>internal.h]
        end

        subgraph "Execution Layer"
            TASK[task.c<br/>Task Manager]
            RTSCHED[sched.c<br/>RT Scheduler]
            TIMER[timer.c<br/>Timer Manager]
            SIGNAL[Signal Handler<br/>sigwait]
        end

        subgraph "System Interface"
            LSCHED[Linux Scheduler<br/>SCHED_DEADLINE]
            AFFINITY[CPU Affinity<br/>Control]
            POSIX[POSIX Timers]
        end

        subgraph "Communication Layer"
            TRPC[trpc.c<br/>libtrpc Client]
            DBUS[D-Bus Connection]
        end
    end

    subgraph Legend2[" "]
        L21["timpani-n (Our Scope)"]
        L22["timpani-o (Our Scope)"]
        L23["Communication (Our Scope)"]
        L24["External Systems"]
    end

    TIMPANIO --> TRPC
    TRPC --> DBUS
    SAMPLE --> TASK

    MAIN --> CONFIG
    MAIN --> CONTEXT
    MAIN --> TASK

    TASK --> RTSCHED
    TASK --> TIMER
    TASK --> SIGNAL

    RTSCHED --> LSCHED
    RTSCHED --> AFFINITY
    TIMER --> POSIX

    SIGNAL --> SYSCALL

    SCHEDSTAT --> RINGBUF
    SIGWAIT --> RINGBUF
    RINGBUF --> MAIN

    SCHED -.-> SCHEDSTAT
    SYSCALL -.-> SIGWAIT

    style TIMPANIO fill:#e3f2fd,stroke:#1976d2,stroke-width:3px
    style SAMPLE fill:#f5f5f5,stroke:#757575,stroke-width:2px
    style SCHED fill:#f5f5f5,stroke:#757575,stroke-width:2px
    style SYSCALL fill:#f5f5f5,stroke:#757575,stroke-width:2px
    style MAIN fill:#e8f5e9,stroke:#388e3c,stroke-width:3px
    style CONFIG fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style CONTEXT fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style TASK fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style RTSCHED fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style TIMER fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style SIGNAL fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style SCHEDSTAT fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style SIGWAIT fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style RINGBUF fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style LSCHED fill:#f5f5f5,stroke:#757575,stroke-width:2px
    style AFFINITY fill:#f5f5f5,stroke:#757575,stroke-width:2px
    style POSIX fill:#f5f5f5,stroke:#757575,stroke-width:2px
    style TRPC fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style DBUS fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style L21 fill:#e8f5e9,stroke:#388e3c,stroke-width:3px
    style L22 fill:#e3f2fd,stroke:#1976d2,stroke-width:3px
    style L23 fill:#fff3e0,stroke:#f57c00,stroke-width:3px
    style L24 fill:#f5f5f5,stroke:#757575,stroke-width:2px
```

---


## Current Repository Layout

```bash
timpani/
├── LICENSE
├── README.md
├── doc/
│   ├── README.md                    # Documentation guide
│   ├── architecture/
│   │   ├── HLD/                     # High-Level Design documents
│   │   │   ├── timpani_system_design_document.md
│   │   │   └── timpani_rust_grpc_architecture.md
│   │   └── LLD/                     # Low-Level Design documents
│   │       ├── timpani-o/           # timpani-o component LLDs (10 docs)
│   │       └── timpani-n/           # timpani-n component LLDs (10 docs)
│   ├── features/
│   │   ├── timpani_features.md      # Feature specification
│   │   └── requirements/
│   │       └── timpani_requirements.md  # FR/NFR requirements
│   ├── contribution/
│   │   ├── coding-rule.md
│   │   └── guidelines-en.md
│   ├── docs/
│   │   ├── api.md
│   │   ├── getting-started.md
│   │   ├── developments.md
│   │   ├── structure.md             # This file
│   │   └── release.md
│   └── images/
├── examples/
│   └── readme.md
├── libbpf/                          # eBPF library (submodule)
├── libtrpc/                         # Legacy D-Bus RPC library
│   ├── src/
│   ├── test/
│   ├── CMakeLists.txt
│   └── README.md
├── sample-apps/                     # Sample applications
│   ├── src/
│   ├── README.md
│   └── WORKLOAD_GUIDE.md
├── scripts/                         # Build and test scripts
│   ├── buildNparse.sh
│   ├── installdeps.sh
│   └── version.txt
├── timpani-n/                       # Legacy C node executor
│   ├── src/
│   ├── test/
│   ├── scripts/
│   ├── README.md
│   ├── README.CentOS.md
│   └── README.Ubuntu20.md
├── timpani-o/                       # Legacy C++ orchestrator
│   ├── src/
│   ├── proto/
│   ├── cmake/
│   ├── tests/
│   └── README.md
└── timpani_rust/                    # 🦀 Active development area
    ├── Cargo.toml                   # Workspace manifest
    ├── timpani-n/                   # Rust node executor
    │   ├── src/
    │   ├── Cargo.toml
    │   └── README.md
    ├── timpani-o/                   # Rust orchestrator
    │   ├── src/
    │   ├── proto/
    │   ├── Cargo.toml
    │   └── README.md
    └── test-tools/                  # Testing utilities
        ├── src/
        └── Cargo.toml
```

---

## Future Development: `timpani_rust/`

All future work is focused on the `timpani_rust` directory. The rest of the repository remains as a reference and for legacy support.

### Current Rust Structure

```bash
timpani_rust/
├── Cargo.toml                   # Workspace manifest
├── about.toml                   # License information
├── deny.toml                    # Dependency checks
├── Justfile                     # Task runner commands
├── timpani-n/                   # Rust node executor
│   ├── src/
│   │   ├── main.rs              # Entry point
│   │   ├── lib.rs               # Core library
│   │   ├── config/              # CLI & configuration (✅ Complete)
│   │   ├── context/             # Runtime context
│   │   └── error/               # Error types (✅ Complete)
│   ├── Cargo.toml
│   ├── build.rs                 # Build script
│   ├── proto/                   # gRPC definitions
│   └── README.md
├── timpani-o/                   # Rust orchestrator (✅ Complete)
│   ├── src/
│   │   ├── main.rs              # Entry point
│   │   ├── lib.rs               # Core library
│   │   ├── config/              # Configuration management
│   │   ├── context/             # Application context
│   │   ├── error/               # Error handling
│   │   ├── fault_client/        # Fault manager client
│   │   ├── hyperperiod/         # Hyperperiod calculation
│   │   ├── node_config/         # Node configuration
│   │   ├── scheduler/           # Global scheduler
│   │   ├── schedinfo_service/   # SchedInfo gRPC service
│   │   └── server/              # gRPC server
│   ├── proto/                   # Protobuf definitions
│   ├── examples/                # Configuration examples
│   ├── Cargo.toml
│   └── README.md
└── test-tools/                  # Testing utilities
    ├── src/
    │   ├── lib.rs
    │   └── bin/                 # Test binaries
    ├── workloads/               # Test workload configs
    └── Cargo.toml
```

#### Module Overview

- **timpani-n**: Rust implementation of the time-triggered node executor
  - **Status:** 🔄 In Progress (Config ✅, Runtime ⏸️)
  - **Communication:** Will use gRPC client (planned)
  - **Monitoring:** Will integrate aya for eBPF (planned)

- **timpani-o**: Rust implementation of the global orchestrator
  - **Status:** ✅ Complete
  - **Communication:** gRPC server (Tonic) on port 50054
  - **Services:** SchedInfo, SyncTimer, ReportDMiss

- **test-tools**: Integration testing and workload validation
  - **Status:** ✅ Active
  - **Purpose:** End-to-end testing, performance benchmarks

---

## Documentation Structure

The `doc/` directory contains all project documentation:

- **architecture/**: System architecture documentation
  - **HLD/**: High-Level Design documents
    - `timpani_system_design_document.md`: Overall system architecture, components, deployment
    - `timpani_rust_grpc_architecture.md`: D-Bus → gRPC migration, communication flow, performance
  - **LLD/**: Low-Level Design component documents
    - `timpani-o/`: 10 component LLD documents (AS-IS vs WILL-BE)
    - `timpani-n/`: 10 component LLD documents (AS-IS vs WILL-BE)

- **features/**: Feature specifications and requirements
  - `timpani_features.md`: Feature breakdown with mermaid diagrams, 3-level feature tables
  - `requirements/timpani_requirements.md`: Functional and non-functional requirements (FR/NFR)

- **docs/**: Implementation and developer guides
  - `api.md`: gRPC services and Rust APIs
  - `getting-started.md`: Build and run instructions
  - `developments.md`: Development workflows
  - `structure.md`: This file
  - `release.md`: Release procedures

- **contribution/**: Coding standards and contribution guidelines
  - `coding-rule.md`: Rust coding standards
  - `guidelines-en.md`: GitHub workflow guidelines

---

## Migration Status

| Component | Legacy | Rust | Status | Documentation |
|-----------|--------|------|--------|---------------|
| **timpani-o** | C++ | Rust | ✅ Complete | [HLD](../architecture/HLD/timpani_system_design_document.md), [LLD/timpani-o/](../architecture/LLD/timpani-o/) |
| **timpani-n** | C | Rust | 🔄 Partial | [HLD](../architecture/HLD/timpani_system_design_document.md), [LLD/timpani-n/](../architecture/LLD/timpani-n/) |
| **Communication** | D-Bus | gRPC | ✅ timpani-o, ⏸️ timpani-n | [gRPC Architecture](../architecture/HLD/timpani_rust_grpc_architecture.md) |

---

## Notes

- **Legacy code** (timpani-n/, timpani-o/, libtrpc/) remains for reference and backward compatibility
- **Active development** occurs exclusively in `timpani_rust/`
- **Documentation** follows architecture → LLD → implementation flow
- **Build system** uses Cargo workspace for Rust components, CMake for legacy C/C++
- **Testing** includes both unit tests (Rust) and integration tests (test-tools/)
