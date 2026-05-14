<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: Data Structures Component

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-o-lld-08
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
**Responsibility:** Define task representations, scheduling results, and type-safe enumerations
**Status:** ✅ Migrated (C++ → Rust)

## Component Overview

Data Structures component defines the core types used throughout timpani-o for representing tasks, scheduling policies, CPU affinity constraints, and final scheduling assignments.

---

## As-Is: C++ Implementation

### Key Structures

```cpp
struct Task {
    std::string name;
    std::string workload_id;
    std::string target_node;

    int policy;              // 0=Normal, 1=FIFO, 2=RR
    int priority;
    std::string affinity;    // String representation
    int cpu_affinity;        // Bitmask

    int period_ms;           // Milliseconds
    uint64_t period_us;      // Microseconds (duplicate)
    int runtime_ms;          // Milliseconds
    uint64_t runtime_us;     // Microseconds (duplicate)
    int deadline_ms;         // Milliseconds
    uint64_t deadline_us;    // Microseconds (duplicate)
    int release_time_us;
    int max_dmiss;

    std::string assigned_node;
    int assigned_cpu;        // -1 = unassigned

    // Dead fields (unused)
    std::vector<std::string> dependencies;
    std::string cluster_requirement;
};

struct sched_task_t {
    char name[16];           // Fixed-size buffer
    char assigned_node[16];  // Fixed-size buffer
    int assigned_cpu;
    int policy;
    int priority;
    uint64_t period_ns;      // Nanoseconds
    uint64_t runtime_ns;
    uint64_t deadline_ns;
    int release_time_us;
    int max_dmiss;
};

using NodeSchedMap = std::map<std::string, std::vector<sched_task_t>>;
```

### Issues (C++)

| Issue | Impact |
|-------|--------|
| Dual time units (ms + µs) | Redundant storage, sync issues |
| `int policy` | No type safety, invalid values possible |
| Dual affinity (`std::string` + `int`) | Confusing, requires manual parsing |
| `assigned_cpu = -1` sentinel | Ambiguous with actual CPU -1 |
| Fixed `char[16]` buffers | Silent truncation risk |
| Dead fields (`dependencies`, `cluster_requirement`) | Wasted memory |
| `std::map<std::string, std::vector<sched_task_t>>` | Copies entire task list |

---

## Will-Be: Rust Implementation

### Core Types

```rust
// File: timpani_rust/timpani-o/src/task.rs

/// Scheduling policy enum (replaces `int policy`)
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum SchedPolicy {
    #[default]
    Normal,      // SCHED_NORMAL
    Fifo,        // SCHED_FIFO
    RoundRobin,  // SCHED_RR
}

impl SchedPolicy {
    pub fn to_linux_int(self) -> i32 {
        match self {
            SchedPolicy::Normal => 0,
            SchedPolicy::Fifo => 1,
            SchedPolicy::RoundRobin => 2,
        }
    }

    pub fn from_proto_int(v: i32) -> Self {
        match v {
            1 => SchedPolicy::Fifo,
            2 => SchedPolicy::RoundRobin,
            _ => SchedPolicy::Normal,
        }
    }
}

/// CPU affinity constraint (replaces dual string/int representation)
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum CpuAffinity {
    #[default]
    Any,
    Pinned(u64),  // Bitmask
}

impl CpuAffinity {
    pub fn from_proto(v: u64) -> Self {
        if v == 0 || v == u64::MAX {
            CpuAffinity::Any
        } else {
            CpuAffinity::Pinned(v)
        }
    }

    pub fn allows_cpu(&self, cpu_id: u32) -> bool {
        match self {
            CpuAffinity::Any => true,
            CpuAffinity::Pinned(mask) => (mask >> cpu_id) & 1 == 1,
        }
    }

    pub fn lowest_cpu(&self) -> Option<u32> {
        match self {
            CpuAffinity::Any => None,
            CpuAffinity::Pinned(mask) => {
                if *mask == 0 {
                    None
                } else {
                    Some(mask.trailing_zeros())
                }
            }
        }
    }
}

/// Internal task (working copy during scheduling)
#[derive(Debug, Clone, Default)]
pub struct Task {
    // Identity
    pub name: String,
    pub workload_id: String,
    pub target_node: String,

    // Scheduling parameters
    pub policy: SchedPolicy,
    pub priority: i32,
    pub affinity: CpuAffinity,

    // Resource requirements
    pub memory_mb: u64,  // Dormant until proto extended

    // Timing (single unit: microseconds)
    pub period_us: u64,
    pub runtime_us: u64,
    pub deadline_us: u64,
    pub release_time_us: u32,
    pub max_dmiss: i32,

    // Assignment (filled by scheduler)
    pub assigned_node: String,
    pub assigned_cpu: Option<u32>,  // None = unassigned
}

impl Task {
    pub fn utilization(&self) -> f64 {
        if self.period_us == 0 {
            0.0
        } else {
            self.runtime_us as f64 / self.period_us as f64
        }
    }

    pub fn is_assigned(&self) -> bool {
        !self.assigned_node.is_empty() && self.assigned_cpu.is_some()
    }
}

/// Wire-ready task (sent to timpani-n)
#[derive(Debug, Clone)]
pub struct SchedTask {
    pub name: String,              // No length limit
    pub assigned_node: String,     // No length limit
    pub assigned_cpu: u32,
    pub policy: SchedPolicy,
    pub priority: i32,
    pub period_ns: u64,            // Nanoseconds
    pub runtime_ns: u64,
    pub deadline_ns: u64,
    pub release_time_us: i32,
    pub max_dmiss: i32,
}

impl SchedTask {
    pub fn from_task(task: &Task) -> Self {
        debug_assert!(task.is_assigned());

        SchedTask {
            name: task.name.clone(),
            assigned_node: task.assigned_node.clone(),
            assigned_cpu: task.assigned_cpu.unwrap_or(0),
            policy: task.policy,
            priority: task.priority,
            period_ns: task.period_us.saturating_mul(1_000),
            runtime_ns: task.runtime_us.saturating_mul(1_000),
            deadline_ns: task.deadline_us.saturating_mul(1_000),
            release_time_us: task.release_time_us as i32,
            max_dmiss: task.max_dmiss,
        }
    }
}

/// Final scheduling result (node_id → list of tasks)
pub type NodeSchedMap = HashMap<String, Vec<SchedTask>>;
```

---

## As-Is vs Will-Be Comparison

| Aspect | C++ (As-Is) | Rust (Will-Be) |
|--------|-------------|----------------|
| **Scheduling Policy** | `int policy` (0/1/2) | `enum SchedPolicy { Normal, Fifo, RoundRobin }` |
| **CPU Affinity** | Dual: `std::string` + `int` | `enum CpuAffinity { Any, Pinned(u64) }` |
| **Time Units** | ms + µs (duplicate storage) | Single unit: µs internally, ns for wire |
| **Unassigned CPU** | `assigned_cpu = -1` | `assigned_cpu: Option<u32>` |
| **Task Name Length** | `char[16]` (truncation risk) | `String` (unbounded) |
| **Memory Tracking** | Not present | `memory_mb: u64` (ready for future) |
| **Dead Fields** | `dependencies`, `cluster_requirement` | Removed |
| **Utilization** | No helper | `task.utilization()` method |
| **Assignment Check** | Manual field checks | `task.is_assigned()` method |
| **Type Safety** | Runtime validation | Compile-time via enums |

---

## Design Decisions

### D-DATA-001: Single Time Unit

**C++ Problem:**
```cpp
struct Task {
    int period_ms;        // Duplicated
    uint64_t period_us;   // Duplicated
    // Which one is source of truth?
};
```

**Rust Solution:**
```rust
pub struct Task {
    pub period_us: u64,    // Single source of truth
}

impl SchedTask {
    pub fn from_task(task: &Task) -> Self {
        SchedTask {
            period_ns: task.period_us.saturating_mul(1_000),  // Convert to ns
            // ...
        }
    }
}
```

**Rationale:**
- **Internal:** Use µs (microseconds) everywhere
- **Wire Protocol:** Convert to ns (nanoseconds) only when sending to timpani-n
- **No Duplication:** Single field eliminates sync issues

---

### D-DATA-002: Type-Safe Scheduling Policy

**C++ Problem:**
```cpp
int policy = 99;  // Compiles, but invalid!
```

**Rust Solution:**
```rust
pub enum SchedPolicy {
    Normal,
    Fifo,
    RoundRobin,
}

// Cannot create invalid value at compile time
let policy = SchedPolicy::Fifo;
```

**Benefits:**
- **Invalid States Impossible:** Compiler rejects invalid policies
- **Pattern Matching:** Exhaustive `match` ensures all cases handled
- **Self-Documenting:** `SchedPolicy::Fifo` clearer than `1`

---

### D-DATA-003: Option<u32> for Assignment

**C++ Sentinel:**
```cpp
int assigned_cpu = -1;  // Unassigned
if (task.assigned_cpu == -1) { /* not assigned */ }
```

**Rust Option:**
```rust
pub assigned_cpu: Option<u32>,

if task.assigned_cpu.is_none() { /* not assigned */ }
```

**Benefits:**
- **No Magic Number:** `-1` is not a valid `u32` value
- **Explicit Intent:** `Option::None` clearly means "not yet assigned"
- **Type Safety:** Cannot accidentally use `None` as a CPU ID

---

### D-DATA-004: CPU Affinity Enum

**C++ Dual Representation:**
```cpp
std::string affinity = "0x0C";  // String representation
int cpu_affinity = 12;          // Numeric representation
// Which is source of truth? Need manual parsing
```

**Rust Unified Type:**
```rust
pub enum CpuAffinity {
    Any,              // No constraint
    Pinned(u64),      // Bitmask
}

impl CpuAffinity {
    pub fn allows_cpu(&self, cpu_id: u32) -> bool {
        match self {
            CpuAffinity::Any => true,
            CpuAffinity::Pinned(mask) => (mask >> cpu_id) & 1 == 1,
        }
    }
}
```

**Usage:**
```rust
if task.affinity.allows_cpu(2) {
    // CPU 2 is allowed
}
```

**Benefits:**
- **Single Representation:** No string/int duality
- **Clear Semantics:** `Any` vs `Pinned` explicit
- **Helper Methods:** `allows_cpu()`, `lowest_cpu()`

---

### D-DATA-005: Unbounded Task Names

**C++ Fixed Buffer:**
```cpp
char name[16];  // "very_long_task_name" → "very_long_task_" (truncated)
strncpy(sched_task.name, task.name.c_str(), 15);
sched_task.name[15] = '\0';
```

**Rust String:**
```rust
pub name: String,  // No length limit
```

**Rationale:**
- **No Truncation:** Task names preserve full length
- **Safety:** Rust strings are UTF-8 validated
- **Flexibility:** Can use descriptive names

---

## Memory Layout Comparison

### C++ Task (Approximate)

```
sizeof(Task) ≈ 200+ bytes:
- std::string name (24 bytes)
- std::string workload_id (24 bytes)
- std::string target_node (24 bytes)
- int period_ms (4 bytes)
- uint64_t period_us (8 bytes)  ← Duplicate
- ... (more duplicates)
- std::vector<std::string> dependencies (24 bytes)  ← Unused
- std::string cluster_requirement (24 bytes)  ← Unused
```

### Rust Task (Approximate)

```
sizeof(Task) ≈ 140 bytes:
- String name (24 bytes)
- String workload_id (24 bytes)
- String target_node (24 bytes)
- SchedPolicy (1 byte + padding)
- CpuAffinity (16 bytes = enum tag + u64)
- period_us (8 bytes)  ← Single
- ... (no duplicates)
- No dead fields
```

**Savings:** ~60 bytes per task (~30% reduction)

---

## Utilization Calculation

### C++ Implementation

```cpp
double GetUtilization(const Task& task) {
    if (task.period_us == 0) return 0.0;
    return static_cast<double>(task.runtime_us) / task.period_us;
}
// Separate free function
```

### Rust Implementation

```rust
impl Task {
    pub fn utilization(&self) -> f64 {
        if self.period_us == 0 {
            0.0
        } else {
            self.runtime_us as f64 / self.period_us as f64
        }
    }
}

// Usage:
let u = task.utilization();
```

**Benefits:**
- Method attached to type (discoverability)
- Consistent interface (`task.utilization()`)
- No external helper function needed

---

## Proto Conversion

### TaskInfo → Task

```rust
fn task_from_proto(t: &TaskInfo, workload_id: &str) -> Task {
    Task {
        name: t.name.clone(),
        workload_id: workload_id.to_owned(),
        target_node: t.node_id.clone(),
        policy: SchedPolicy::from_proto_int(t.policy),
        priority: t.priority,
        affinity: CpuAffinity::from_proto(t.cpu_affinity),
        period_us: t.period.max(0) as u64,
        runtime_us: t.runtime.max(0) as u64,
        deadline_us: t.deadline.max(0) as u64,
        release_time_us: t.release_time.max(0) as u32,
        max_dmiss: t.max_dmiss,
        memory_mb: 0,  // Not in proto yet
        ..Task::default()
    }
}
```

### Task → ScheduledTask (Proto)

```rust
fn to_proto_task(t: &SchedTask) -> ScheduledTask {
    ScheduledTask {
        name: t.name.clone(),
        sched_priority: t.priority,
        sched_policy: t.policy.to_linux_int(),
        period_us: (t.period_ns / 1_000) as i32,
        release_time_us: t.release_time_us,
        runtime_us: (t.runtime_ns / 1_000) as i32,
        deadline_us: (t.deadline_ns / 1_000) as i32,
        cpu_affinity: 1u64 << t.assigned_cpu,  // Single-bit mask
        max_dmiss: t.max_dmiss,
        assigned_node: t.assigned_node.clone(),
    }
}
```

---

## Testing

### C++ Testing

```cpp
TEST(TaskTest, Utilization) {
    Task task;
    task.period_us = 10000;
    task.runtime_us = 2000;

    double util = GetUtilization(task);
    EXPECT_DOUBLE_EQ(util, 0.2);
}
```

### Rust Testing

```rust
#[test]
fn test_task_utilization() {
    let task = Task {
        period_us: 10_000,
        runtime_us: 2_000,
        ..Default::default()
    };

    assert_eq!(task.utilization(), 0.2);
}

#[test]
fn test_cpu_affinity_allows() {
    let affinity = CpuAffinity::Pinned(0x0C);  // CPUs 2 and 3

    assert!(!affinity.allows_cpu(0));
    assert!(!affinity.allows_cpu(1));
    assert!(affinity.allows_cpu(2));
    assert!(affinity.allows_cpu(3));
    assert!(!affinity.allows_cpu(4));
}

#[test]
fn test_policy_roundtrip() {
    let policy = SchedPolicy::Fifo;
    let proto_int = policy.to_linux_int();  // 1
    let parsed = SchedPolicy::from_proto_int(proto_int);

    assert_eq!(parsed, SchedPolicy::Fifo);
}

#[test]
fn test_task_assignment_check() {
    let mut task = Task::default();
    assert!(!task.is_assigned());

    task.assigned_node = "node01".to_string();
    task.assigned_cpu = Some(2);
    assert!(task.is_assigned());
}
```

---

## Migration Notes

### What Changed

1. **Policy:** `int` → `enum SchedPolicy`
2. **Affinity:** Dual representation → `enum CpuAffinity`
3. **Time Units:** ms + µs → µs only
4. **Assignment:** `int = -1` → `Option<u32>`
5. **Task Names:** `char[16]` → `String`
6. **Dead Fields:** Removed `dependencies`, `cluster_requirement`
7. **Helpers:** Added `utilization()`, `is_assigned()`, `allows_cpu()`

### What Stayed the Same

1. **Core Fields:** name, workload_id, priority, period, runtime, deadline
2. **Scheduling Semantics:** FIFO, RR, Normal policies
3. **Affinity Logic:** Bitmask-based CPU selection
4. **Wire Protocol:** Same proto messages (TaskInfo, ScheduledTask)

---

**Document Version:** 1.0
**Last Updated:** May 12, 2026
**Status:** ✅ Complete
**Verified Against:** `timpani_rust/timpani-o/src/task.rs` (actual implementation)
