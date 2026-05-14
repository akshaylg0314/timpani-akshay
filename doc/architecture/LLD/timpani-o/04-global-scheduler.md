<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: Global Scheduler Component

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-o-lld-04
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** Core Scheduling Logic
**Responsibility:** Allocate tasks to nodes and CPUs using real-time scheduling algorithms
**Status:** ✅ Migrated (C++ → Rust)

## Component Overview

The Global Scheduler component implements the core task allocation logic for timpani-o. It receives a set of real-time tasks and distributes them across available compute nodes and CPUs, ensuring schedulability constraints are met.

---

## As-Is: C++ Implementation

### Class Structure

```cpp
class GlobalScheduler {
public:
    explicit GlobalScheduler(std::shared_ptr<NodeConfigManager> node_config_manager);

    bool ProcessScheduleInfo(const SchedInfo& sched_info, NodeSchedMap& result);
    bool SetAlgorithm(const std::string& algorithm_name);
    void Clear();

private:
    bool ScheduleTargetNodePriority();
    bool ScheduleLeastLoaded();
    bool ScheduleBestFitDecreasing();

    bool FindBestCPUForTask(Task& task, const std::string& node_id);

    std::vector<Task> tasks_;
    std::map<std::string, std::vector<int>> available_cpus_;
    std::map<std::string, std::map<int, double>> cpu_utilization_;
};
```

### Responsibilities (C++)

1. **Parse** and validate scheduling information
2. **Allocate** tasks to nodes based on selected algorithm
3. **Assign** CPUs to tasks on each node
4. **Track** CPU utilization to prevent oversubscription
5. **Validate** schedules against feasibility constraints

### Scheduling Algorithms (C++)

1. **Target Node Priority**
   - Each task specifies a `target_node`
   - Scheduler assigns to the requested node only
   - Finds best available CPU on that node

2. **Least Loaded**
   - Assigns each task to the node with lowest total utilization
   - Balances load across all nodes

3. **Best Fit Decreasing**
   - Sorts tasks by WCET (descending)
   - Assigns each to the node with tightest fit

### Key Features (C++)

- **Utilization Threshold:** 90% max CPU utilization (hard-coded)
- **State Management:** Mutable internal state cleared via `Clear()`
- **Iteration Order:** `std::map` (sorted by key)
- **Error Handling:** `bool` return values

---

## Will-Be: Rust Implementation

### Module Structure

```rust
// File: timpani_rust/timpani-o/src/scheduler/mod.rs

pub struct GlobalScheduler {
    node_config_manager: Arc<NodeConfigManager>,
}

impl GlobalScheduler {
    pub fn new(node_config_manager: Arc<NodeConfigManager>) -> Self {
        Self { node_config_manager }
    }

    pub fn schedule(
        &self,
        mut tasks: Vec<Task>,
        algorithm: &str,
    ) -> Result<NodeSchedMap, SchedulerError> {
        // Per-call local state
        let avail = self.build_available_cpus();
        let mut util = Self::build_cpu_utilization(&avail);

        // Algorithm dispatch
        match algorithm {
            "target_node_priority" => {
                self.schedule_target_node_priority(&mut tasks, &avail, &mut util)?
            }
            "least_loaded" => {
                self.schedule_least_loaded(&mut tasks, &avail, &mut util)?
            }
            "best_fit_decreasing" => {
                self.schedule_best_fit_decreasing(&mut tasks, &avail, &mut util)?
            }
            other => return Err(SchedulerError::UnknownAlgorithm(other.to_string())),
        }

        // Post-schedule Liu & Layland check
        self.run_liu_layland_check(&tasks);

        // Build final schedule map
        Ok(self.build_sched_map(tasks))
    }
}
```

### Responsibilities (Rust)

1. **Distribute** `Vec<Task>` across nodes using selected algorithm
2. **Assign** specific CPU to each task (populate `assigned_cpu`)
3. **Track** per-CPU utilization with `BTreeMap<String, BTreeMap<u32, f64>>`
4. **Validate** against 90% threshold during assignment
5. **Check** Liu & Layland bound post-scheduling (warning only)

### Scheduling Algorithms (Rust)

Same three algorithms as C++, with identical logic:

```rust
fn schedule_target_node_priority(...) -> Result<(), SchedulerError> {
    for task in tasks {
        let node = &task.target_node;
        let cpu = find_best_cpu_for_task(task, node, avail, util)?;
        task.assigned_node = node.clone();
        task.assigned_cpu = Some(cpu);
        update_utilization(node, cpu, task, util);
    }
    Ok(())
}
```

### Key Features (Rust)

- **Stateless Design:** All per-run state (`avail`, `util`) is local to `schedule()` call
- **Type Safety:** `Result<NodeSchedMap, SchedulerError>` with structured errors
- **Deterministic Order:** `BTreeMap` ensures alphabetical node iteration (automotive requirement)
- **Liu & Layland Validation:** Computes theoretical schedulability bound, logs warning if exceeded
- **No Mutable State:** `&self` is immutable, all mutation happens on local variables

---

## As-Is vs Will-Be Comparison

| Aspect | C++ (As-Is) | Rust (Will-Be) |
|--------|-------------|----------------|
| **State Management** | Mutable fields, explicit `Clear()` | Stateless - all state local to `schedule()` |
| **Map Type** | `std::map<>` (sorted) | `BTreeMap<>` (sorted + deterministic) |
| **Error Handling** | `bool` return + silent `continue` | `Result<T, SchedulerError>` with typed variants |
| **CPU Model (Alg 2&3)** | Dequeue CPUs from list | Utilization tracking for all algorithms |
| **Feasibility Check** | 90% hard-coded threshold | 90% threshold + Liu & Layland bound warning |
| **Thread Safety** | Mutable shared state | `Send + Sync` - no interior mutability |
| **Function Signature** | `bool ProcessScheduleInfo(const SchedInfo&, NodeSchedMap&)` | `fn schedule(&self, Vec<Task>, &str) -> Result<NodeSchedMap, E>` |
| **Iteration Order** | Sorted but platform-dependent | Always deterministic (BTreeMap) |

---

## Design Decisions

### D-SCHED-001: Stateless vs Stateful

**C++ Approach:**
```cpp
class GlobalScheduler {
    std::vector<Task> tasks_;          // Mutable state
    std::map<...> available_cpus_;     // Mutable state
    std::map<...> cpu_utilization_;    // Mutable state

public:
    bool ProcessScheduleInfo(...) {
        Clear(); // Must clear previous state
        // Use instance fields
    }
    void Clear() {
        tasks_.clear();
        available_cpus_.clear();
        cpu_utilization_.clear();
    }
};
```

**Rust Approach:**
```rust
pub struct GlobalScheduler {
    node_config_manager: Arc<NodeConfigManager>, // Read-only
}

impl GlobalScheduler {
    pub fn schedule(&self, mut tasks: Vec<Task>, algorithm: &str)
        -> Result<NodeSchedMap, SchedulerError>
    {
        // All state is local - allocated and dropped per call
        let avail = self.build_available_cpus();
        let mut util = Self::build_cpu_utilization(&avail);

        // ...

        Ok(self.build_sched_map(tasks))
    } // avail, util dropped here
}
```

**Rationale:**
- **Thread Safety:** Rust `&self` is immutable, no risk of concurrent modification
- **No Clear() Needed:** State automatically dropped at end of call
- **Testability:** Multiple concurrent `schedule()` calls don't interfere
- **Memory Safety:** Compiler guarantees no dangling references

---

### D-SCHED-002: BTreeMap vs HashMap

**C++ Implementation:**
```cpp
std::map<std::string, ...> available_cpus_;  // Sorted by key
```

**Rust Implementation:**
```rust
type AvailCpus = BTreeMap<String, Vec<u32>>;  // Sorted by key
type CpuUtil = BTreeMap<String, BTreeMap<u32, f64>>;  // Two-level sorted
```

**Why Not HashMap?**
- **Determinism:** For automotive systems, same input must always produce same output
- **BTreeMap guarantees:** Alphabetical iteration order (node names)
- **Debugging:** Consistent order in logs/traces

**Quote from Code:**
```rust
/// `BTreeMap` (not `HashMap`) so iteration order is always alphabetical by
/// node name — required for deterministic scheduling.
```

---

### D-SCHED-003: Liu & Layland Feasibility Check

**Theory:**
Under Rate Monotonic scheduling, a task set of `n` tasks is **guaranteed** schedulable if:

$$U = \sum_{i=1}^{n} \frac{C_i}{T_i} \leq n \left(2^{1/n} - 1\right)$$

**Bound Values:**
| n | Bound |
|---|-------|
| 1 | 1.000 |
| 2 | 0.828 |
| 3 | 0.780 |
| 5 | 0.743 |
| ∞ | ln(2) ≈ 0.693 |

**C++ Implementation:**
- 90% threshold hard-coded
- No Liu & Layland check

**Rust Implementation:**
```rust
pub fn liu_layland_bound(n: usize) -> f64 {
    if n == 0 { return 0.0; }
    let nf = n as f64;
    nf * (2.0_f64.powf(1.0 / nf) - 1.0)
}

pub fn check_liu_layland(tasks_on_node: &[&Task]) -> Option<f64> {
    let total_u: f64 = tasks.iter()
        .map(|t| t.runtime_us as f64 / t.period_us as f64)
        .sum();

    let bound = liu_layland_bound(tasks.len());

    if total_u > bound {
        Some(total_u)  // Warning - may not be schedulable
    } else {
        None  // Provably schedulable
    }
}
```

**Current Status:**
- Liu & Layland check is **implemented and logged**
- Schedule is **not rejected** if bound exceeded (warning only)
- 90% threshold remains the hard gate during assignment

**Future Intent:**
Use L&L bound to set `CPU_UTILIZATION_THRESHOLD` dynamically per node based on task count, instead of fixed 90%.

---

## Error Handling

### C++ Error Handling

```cpp
bool ProcessScheduleInfo(...) {
    if (tasks_.empty()) {
        LOG_ERROR("No tasks to schedule");
        return false;
    }
    if (!config_->IsLoaded()) {
        return false;
    }
    for (auto& task : tasks_) {
        if (!FindBestCPUForTask(task, task.target_node)) {
            continue; // Silent failure - skip task
        }
    }
    return true;
}
```

**Issues:**
- `bool` return doesn't explain what failed
- `continue` silently skips unschedulable tasks
- Caller cannot distinguish "no tasks" vs "config not loaded" vs "task rejected"

### Rust Error Handling

```rust
#[derive(Debug, Error)]
pub enum SchedulerError {
    #[error("no tasks to schedule")]
    NoTasks,

    #[error("node configuration is not loaded")]
    ConfigNotLoaded,

    #[error("unknown scheduling algorithm: {0}")]
    UnknownAlgorithm(String),

    #[error("task {task} rejected: {reason}")]
    TaskRejected {
        task: String,
        reason: AdmissionReason,
    },
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum AdmissionReason {
    NoTargetNode,
    NodeNotFound(String),
    NoCpuAvailable,
    CpuUtilizationExceeded { cpu_id: u32, utilization: u64 },
}
```

**Benefits:**
- **Specific Errors:** Each failure case has a distinct variant
- **Context:** `TaskRejected` includes task name and reason
- **Fail-Fast:** First rejected task aborts the entire schedule (no silent continues)
- **Testability:** Error variants can be pattern-matched in tests

---

## Scheduling Algorithm Details

### 1. Target Node Priority

**Use Case:** Tasks have strict node placement requirements (e.g., sensor tasks must run on sensor node)

**C++ Logic:**
```cpp
for (auto& task : tasks_) {
    if (!FindBestCPUForTask(task, task.target_node)) {
        continue; // Skip
    }
}
```

**Rust Logic:**
```rust
for task in tasks.iter_mut() {
    if task.target_node.is_empty() {
        return Err(SchedulerError::TaskRejected {
            task: task.name.clone(),
            reason: AdmissionReason::NoTargetNode,
        });
    }
    let cpu = find_best_cpu_for_task(task, &task.target_node, avail, util)?;
    task.assigned_node = task.target_node.clone();
    task.assigned_cpu = Some(cpu);
}
```

**Key Difference:** Rust fails immediately if task has no target node; C++ silently skips it.

---

### 2. Least Loaded

**Use Case:** Maximize resource availability by balancing load

**Algorithm:**
1. For each task, calculate current total utilization of each node
2. Assign task to node with lowest utilization
3. Find best CPU on that node

**Implementation:**
```rust
fn schedule_least_loaded(...) -> Result<(), SchedulerError> {
    for task in tasks.iter_mut() {
        // Find node with lowest total utilization
        let node = find_least_loaded_node(&util)?;
        let cpu = find_best_cpu_for_task(task, &node, avail, util)?;

        task.assigned_node = node.clone();
        task.assigned_cpu = Some(cpu);
        update_utilization(&node, cpu, task, util);
    }
    Ok(())
}
```

---

### 3. Best Fit Decreasing

**Use Case:** Bin-packing optimization for maximum utilization

**Algorithm:**
1. Sort tasks by WCET (descending)
2. For each task, find node that will have highest utilization **after** assignment, without exceeding 1.0
3. This creates tightest packing, leaving other nodes with more headroom

**Implementation:**
```rust
fn schedule_best_fit_decreasing(...) -> Result<(), SchedulerError> {
    // Sort by runtime (descending)
    tasks.sort_by(|a, b| b.runtime_us.cmp(&a.runtime_us));

    for task in tasks.iter_mut() {
        let node = find_best_fit_node_for_task(task, avail, util)?;
        let cpu = find_best_cpu_for_task(task, &node, avail, util)?;

        task.assigned_node = node.clone();
        task.assigned_cpu = Some(cpu);
        update_utilization(&node, cpu, task, util);
    }
    Ok(())
}
```

---

## CPU Assignment Logic

### find_best_cpu_for_task()

**Input:**
- `task`: Task to assign
- `node_id`: Target node
- `avail`: Available CPUs per node
- `util`: Current utilization per CPU

**Logic:**
```rust
fn find_best_cpu_for_task(...) -> Result<u32, SchedulerError> {
    let task_util = task.runtime_us as f64 / task.period_us as f64;

    // Get CPUs available on this node
    let node_cpus = avail.get(node_id)
        .ok_or_else(|| SchedulerError::NodeNotFound(node_id.clone()))?;

    // Filter by affinity constraint
    let allowed: Vec<u32> = node_cpus.iter()
        .filter(|&&cpu| task.affinity.allows_cpu(cpu))
        .copied()
        .collect();

    if allowed.is_empty() {
        return Err(SchedulerError::NoCpuAvailable);
    }

    // Find CPU with lowest current utilization
    let best_cpu = allowed.iter()
        .min_by(|a, b| {
            let u_a = util[node_id].get(a).unwrap_or(&0.0);
            let u_b = util[node_id].get(b).unwrap_or(&0.0);
            u_a.partial_cmp(u_b).unwrap()
        })
        .copied()
        .unwrap();

    // Check 90% threshold
    let new_util = util[node_id].get(&best_cpu).unwrap_or(&0.0) + task_util;
    if new_util > CPU_UTILIZATION_THRESHOLD {
        return Err(SchedulerError::CpuUtilizationExceeded {
            cpu_id: best_cpu,
            utilization: (new_util * 100.0) as u64,
        });
    }

    Ok(best_cpu)
}
```

**Constant:**
```rust
const CPU_UTILIZATION_THRESHOLD: f64 = 0.90; // 90%
```

---

## Data Structures

### NodeSchedMap

**Type Alias:**
```rust
pub type NodeSchedMap = HashMap<String, Vec<SchedTask>>;
```

**Purpose:** Final output of scheduler - maps `node_id` → list of tasks assigned to that node

**Example:**
```rust
{
    "node01": [
        SchedTask { name: "sensor_fusion", assigned_cpu: 2, ... },
        SchedTask { name: "lidar_proc", assigned_cpu: 3, ... },
    ],
    "node02": [
        SchedTask { name: "path_planning", assigned_cpu: 1, ... },
    ],
}
```

---

## Testing

### C++ Testing

```cpp
TEST_F(GlobalSchedulerTest, TargetNodePriority) {
    GlobalScheduler scheduler(node_config);

    SchedInfo info;
    // ... populate info

    NodeSchedMap result;
    bool success = scheduler.ProcessScheduleInfo(info, result);

    EXPECT_TRUE(success);
    EXPECT_EQ(result.size(), 2);
}
```

**Limitations:**
- `bool` return doesn't explain failures
- Hard to test error cases
- Requires clearing state between tests

### Rust Testing

```rust
#[test]
fn test_target_node_priority_success() {
    let config = Arc::new(NodeConfigManager::default());
    let scheduler = GlobalScheduler::new(config);

    let tasks = vec![
        Task {
            name: "task_a".into(),
            target_node: "node01".into(),
            period_us: 10_000,
            runtime_us: 2_000,
            ..Default::default()
        },
    ];

    let result = scheduler.schedule(tasks, "target_node_priority");

    assert!(result.is_ok());
    let map = result.unwrap();
    assert_eq!(map.len(), 1);
    assert!(map.contains_key("node01"));
}

#[test]
fn test_task_rejection_no_target_node() {
    let config = Arc::new(NodeConfigManager::default());
    let scheduler = GlobalScheduler::new(config);

    let tasks = vec![
        Task {
            name: "task_missing_target".into(),
            target_node: String::new(), // Missing!
            ..Default::default()
        },
    ];

    let result = scheduler.schedule(tasks, "target_node_priority");

    assert!(matches!(
        result,
        Err(SchedulerError::TaskRejected {
            reason: AdmissionReason::NoTargetNode,
            ..
        })
    ));
}
```

**Benefits:**
- Pattern matching on error types
- No state cleanup needed (stateless)
- Can test concurrently (no shared state)

---

## Migration Notes

### What Changed

1. **State Management:** Stateful → Stateless
2. **Error Handling:** `bool` → `Result<T, SchedulerError>`
3. **Feasibility:** Added Liu & Layland theoretical bound check
4. **Determinism:** `std::map` → `BTreeMap` for guaranteed order
5. **Error Propagation:** Silent `continue` → Fail-fast with context

### What Stayed the Same

1. **Algorithm Logic:** All three algorithms identical
2. **90% Threshold:** Still the hard gate
3. **CPU Assignment:** Same "find lowest utilization" logic
4. **Affinity Handling:** Same mask-based logic

---

**Document Version:** 1.0
**Last Updated:** May 12, 2026
**Status:** ✅ Complete
**Verified Against:** `timpani_rust/timpani-o/src/scheduler/mod.rs` (actual implementation)
