<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: Data Structures

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-n-lld-10
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** Core Data Models
**Responsibility:** Context, task info, runtime state structures
**Status:** 🔄 Partial (structures defined in Rust, not used yet)

---

## AS-IS: C Implementation

**File:** `timpani-n/src/internal.h`

### Main Context

```c
struct context {
    struct config config;                    // Configuration
    struct runtime runtime;                  // Runtime state
    struct sched_info sinfo;                 // Schedule from timpani-o
    struct hyperperiod_manager hp_manager;   // Hyperperiod info
    bool shutdown_requested;                 // Shutdown flag
};
```

### Task Info

```c
struct task_info {
    char name[256];          // Task name
    pid_t pid;               // Process ID
    int pidfd;               // PID file descriptor
    int priority;            // RT priority
    int policy;              // Scheduling policy
    uint64_t cpu_affinity;   // CPU affinity mask
    int period_us;           // Period in microseconds
    int release_time_us;     // Release time offset
    int runtime_us;          // WCET
    int deadline_us;         // Relative deadline
    int max_dmiss;           // Max deadline misses allowed
};
```

### Time Trigger

```c
struct time_trigger {
    struct task_info task;    // Task metadata
    struct timespec period;   // Period as timespec
    struct timespec deadline; // Deadline as timespec
    uint64_t sigwait_ts;      // Last signal timestamp
    bool sigwait_enter;       // Signal entry flag
    struct context *ctx;      // Back-pointer
};
```

### Runtime State

```c
struct runtime {
    struct time_trigger *tt_list;  // Task list
    int hyperperiod_timer_fd;      // Timer FD
    int bpf_ringbuf_fd;            // BPF ring buffer FD
    sd_bus *dbus;                  // D-Bus connection
    struct ring_buffer *rb;        // BPF ring buffer
    struct timespec sync_start_time;  // Synchronized start
};
```

---

## WILL-BE: Rust Implementation (🔄 Defined, Not Used)

**Files:** `timpani_rust/timpani-n/src/context/mod.rs`

```rust
pub struct Context {
    pub config: Config,
    pub runtime: RuntimeState,
    pub sched_info: Option<SchedInfo>,
    pub hyperperiod: Option<HyperperiodInfo>,
    pub shutdown_requested: Arc<AtomicBool>,
}

pub struct SchedInfo {
    pub workload_id: String,
    pub hyperperiod_us: u64,
    pub tasks: Vec<TaskInfo>,
}

pub struct TaskInfo {
    pub name: String,
    pub pid: i32,
    pub priority: i32,
    pub policy: SchedPolicy,
    pub cpu_affinity: u64,
    pub period_us: u64,
    pub runtime_us: u64,
    pub deadline_us: u64,
    pub max_dmiss: i32,
}
```

**Status:** Structures defined ✅, initialization logic ⏸️

---

**Document Version:** 1.0
**Status:** C ✅, Rust 🔄 (structures only)
