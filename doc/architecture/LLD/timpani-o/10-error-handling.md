<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: Error Handling and Fault Tolerance Component

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-o-lld-10
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** Error Management System
**Responsibility:** Define error types, propagation strategies, and fault recovery mechanisms
**Status:** ✅ Migrated (C++ → Rust)

## Component Overview

Error Handling component provides structured error types, propagation mechanisms, and recovery strategies for all failure scenarios in timpani-o, including scheduling failures, resource exhaustion, RPC errors, and configuration problems.

---

## As-Is: C++ Implementation

### Error Handling Patterns (C++)

**1. Boolean Returns:**
```cpp
bool LoadFromFile(const std::string& path) {
    try {
        // ... load config
        return true;
    } catch (const std::exception& e) {
        LOG_ERROR("Load failed: " << e.what());
        return false;  // Caller doesn't know why
    }
}
```

**2. Sentinel Values:**
```cpp
uint64_t CalculateHyperperiod(...) {
    if (tasks.empty()) {
        return 0;  // Error: no tasks
    }
    uint64_t lcm = ...;
    if (overflow) {
        return 0;  // Error: overflow
    }
    return lcm;  // Success: actual value (could also be 0!)
}
```

**3. Exceptions:**
```cpp
Status AddSchedInfo(...) {
    try {
        ProcessSchedule();
        return Status::OK;
    } catch (const std::exception& e) {
        return Status(StatusCode::INTERNAL, e.what());
    }
}
```

**4. NULL Pointers:**
```cpp
const NodeConfig* GetNodeConfig(const std::string& node_id) {
    auto it = nodes_.find(node_id);
    if (it == nodes_.end()) {
        return nullptr;  // Not found
    }
    return &it->second;
}
```

### Issues (C++)

| Pattern | Problem |
|---------|---------|
| `bool` return | No error context, cannot distinguish failure types |
| Sentinel `0` or `-1` | Ambiguous with valid values |
| Exceptions | Expensive, not automotive-safe (unwinding) |
| NULL pointers | Requires manual null checks, easy to forget |
| Log-only errors | Caller cannot programmatically handle errors |

---

## Will-Be: Rust Implementation

### Error Handling Philosophy (Rust)

**Core Principle:** All errors are explicit, typed, and propagate via `Result<T, E>`

**Three Error Patterns:**

1. **Domain-Specific Errors:** Custom enums with context
2. **Generic Errors:** `anyhow::Error` for quick prototyping
3. **RPC Errors:** `tonic::Status` for gRPC boundaries

---

## Error Types

### 1. Scheduler Errors

**File:** `timpani_rust/timpani-o/src/scheduler/error.rs`

```rust
/// Top-level scheduler failure
#[derive(Debug, Error)]
pub enum SchedulerError {
    #[error("no tasks provided — task list is empty")]
    NoTasks,

    #[error("node configuration is not loaded")]
    ConfigNotLoaded,

    #[error("unknown scheduling algorithm: '{0}'")]
    UnknownAlgorithm(String),

    #[error("task '{task}' has no workload_id")]
    MissingWorkloadId { task: String },

    #[error("task '{task}' has no target_node")]
    MissingTargetNode { task: String },

    #[error("task '{task}' rejected on node '{node}': {reason}")]
    AdmissionRejected {
        task: String,
        node: String,
        reason: AdmissionReason,
    },

    #[error("no schedulable node found for task '{0}'")]
    NoSchedulableNode(String),
}

/// Detailed reason for admission failure
#[derive(Debug, Clone, PartialEq)]
pub enum AdmissionReason {
    NodeNotFound { node: String },

    InsufficientMemory { required_mb: u64, available_mb: u64 },

    CpuAffinityUnavailable { requested_cpu: u32 },

    CpuUtilizationExceeded {
        cpu: u32,
        current: f64,
        added: f64,
        threshold: f64,
    },

    NoAvailableCpu,
}

impl std::fmt::Display for AdmissionReason {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            AdmissionReason::NodeNotFound { node } => {
                write!(f, "node '{}' not found in configuration", node)
            }
            AdmissionReason::CpuUtilizationExceeded { cpu, current, added, threshold } => {
                write!(
                    f,
                    "CPU {} utilization would be {:.1}% + {:.1}% = {:.1}% (threshold {:.0}%)",
                    cpu,
                    current * 100.0,
                    added * 100.0,
                    (current + added) * 100.0,
                    threshold * 100.0,
                )
            }
            // ... other variants
        }
    }
}
```

**Benefits:**
- **Specific Variants:** Each failure mode has a distinct type
- **Context:** Carries exact values (CPU ID, utilization, task name)
- **Actionable:** Caller can pattern match and handle differently
- **Display:** Automatic human-readable error messages

---

### 2. Hyperperiod Errors

**File:** `timpani_rust/timpani-o/src/hyperperiod/mod.rs`

```rust
#[derive(Debug, PartialEq, Eq)]
pub enum HyperperiodError {
    /// No tasks with valid periods
    NoValidPeriods,

    /// LCM calculation overflowed u64
    Overflow { a: u64, b: u64 },

    /// Hyperperiod exceeded configured limit
    TooLarge { value_us: u64, limit_us: u64 },
}

impl std::fmt::Display for HyperperiodError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            HyperperiodError::NoValidPeriods => {
                write!(f, "no tasks with a valid (non-zero) period")
            }
            HyperperiodError::Overflow { a, b } => {
                write!(f, "LCM overflow computing lcm({a}, {b})")
            }
            HyperperiodError::TooLarge { value_us, limit_us } => write!(
                f,
                "hyperperiod {value_us}µs ({:.1}s) exceeds limit {limit_us}µs ({:.1}s)",
                *value_us as f64 / 1_000_000.0,
                *limit_us as f64 / 1_000_000.0
            ),
        }
    }
}
```

**Error Display Examples:**
```
no tasks with a valid (non-zero) period

LCM overflow computing lcm(18446744073709551615, 2)

hyperperiod 7200000000µs (7200.0s) exceeds limit 3600000000µs (3600.0s)
```

---

### 3. Fault Service Errors

**File:** `timpani_rust/timpani-o/src/fault/mod.rs`

```rust
#[derive(Debug, Error)]
pub enum FaultError {
    /// tonic channel construction failure
    #[error("transport error: {0}")]
    Transport(#[from] tonic::transport::Error),

    /// gRPC call failed (network, server unavailable)
    #[error("RPC status: {0}")]
    Rpc(#[from] tonic::Status),

    /// Pullpiri returned non-zero status
    #[error("Pullpiri returned non-zero status {0}")]
    RemoteError(i32),
}
```

**Error Conversion:**
```rust
// Automatic conversion via #[from]
async fn notify_fault(...) -> Result<(), FaultError> {
    let channel = Endpoint::from_shared(addr)?; // transport::Error → FaultError::Transport
    let response = stub.notify_fault(request).await?; // Status → FaultError::Rpc

    if response.status != 0 {
        return Err(FaultError::RemoteError(response.status));
    }

    Ok(())
}
```

---

### 4. Configuration Errors

**File:** `timpani_rust/timpani-o/src/config/mod.rs`

```rust
// Uses anyhow::Error with context
pub fn load_from_file(&mut self, path: &Path) -> Result<()> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("Cannot open configuration file: {}", path.display()))?;

    let file: NodeConfigFile = serde_yaml::from_str(&content)
        .with_context(|| format!("Failed to parse YAML file: {}", path.display()))?;

    // ...
    Ok(())
}
```

**Error Context Chain:**
```
Failed to parse YAML file: /etc/timpani/nodes.yaml
Caused by:
    missing field `available_cpus` at line 3 column 5
```

---

## As-Is vs Will-Be Comparison

| Aspect | C++ (As-Is) | Rust (Will-Be) |
|--------|-------------|----------------|
| **Return Types** | `bool`, sentinel values, NULL | `Result<T, E>` |
| **Error Context** | Logged separately | Carried in error variant |
| **Exceptions** | Used for unexpected failures | Not used (zero-cost abstractions) |
| **Error Propagation** | Manual checks, early returns | `?` operator (automatic) |
| **Type Safety** | Runtime distinction | Compile-time via enum variants |
| **Null Checks** | Manual `if (ptr == nullptr)` | `Option<T>` (compile-enforced) |
| **Error Messages** | Format strings in code | `Display` trait implementation |
| **Testability** | Hard to test error paths | Easy with `assert!(matches!(err, E::Variant))` |

---

## Design Decisions

### D-ERR-001: Result<T, E> vs Exceptions

**C++ Exceptions:**
```cpp
void ProcessSchedule() {
    if (error) {
        throw std::runtime_error("Scheduling failed");
    }
}

try {
    ProcessSchedule();
} catch (const std::exception& e) {
    LOG_ERROR(e.what());
    return false;
}
```

**Rust Result:**
```rust
fn process_schedule(...) -> Result<NodeSchedMap, SchedulerError> {
    if error {
        return Err(SchedulerError::AdmissionRejected { ... });
    }
    Ok(map)
}

match process_schedule(...) {
    Ok(map) => { /* success */ }
    Err(SchedulerError::AdmissionRejected { task, reason }) => {
        error!("Task '{}' rejected: {}", task, reason);
        return Err(Status::resource_exhausted(...));
    }
}
```

**Why Result?**
- **Explicit:** Compiler enforces error handling
- **Zero-Cost:** No stack unwinding overhead
- **Automotive-Safe:** No hidden control flow
- **Pattern Matching:** Structured error handling

---

### D-ERR-002: Custom Errors vs anyhow::Error

**Custom Errors (Production):**
```rust
pub enum SchedulerError {
    NoTasks,
    AdmissionRejected { task: String, reason: AdmissionReason },
    // ... specific variants
}
```

**anyhow::Error (Prototyping/Config):**
```rust
pub fn load_from_file(&mut self, path: &Path) -> Result<()> {
    // Result<()> is shorthand for Result<(), anyhow::Error>
    let content = std::fs::read_to_string(path)?;
    Ok(())
}
```

**When to Use Each:**

| Use Case | Error Type | Rationale |
|----------|------------|-----------|
| **Scheduler logic** | `SchedulerError` enum | Need specific handling per variant |
| **Fault reporting** | `FaultError` enum | Different recovery strategies |
| **Config loading** | `anyhow::Error` | Generic I/O errors, context is enough |
| **Hyperperiod** | `HyperperiodError` enum | Caller needs to know overflow vs too large |

---

### D-ERR-003: Error Propagation with `?` Operator

**C++ Manual Propagation:**
```cpp
bool Outer() {
    bool result = Inner();
    if (!result) {
        LOG_ERROR("Inner failed");
        return false;
    }
    // ... continue
    return true;
}
```

**Rust `?` Operator:**
```rust
fn outer(...) -> Result<T, E> {
    let value = inner()?; // If Err, return immediately
    // ... continue with value
    Ok(result)
}
```

**How `?` Works:**
```rust
// This:
let value = inner()?;

// Desugars to:
let value = match inner() {
    Ok(v) => v,
    Err(e) => return Err(e.into()), // Auto-convert via From trait
};
```

**Benefits:**
- **Concise:** One character instead of 3-5 lines
- **Automatic Conversion:** `E1` → `E2` if `From<E1> for E2` exists
- **Early Return:** Exits immediately on error
- **Type-Checked:** Compiler verifies error types match

---

### D-ERR-004: Option<T> vs NULL Pointers

**C++ NULL Pointer:**
```cpp
const NodeConfig* GetNodeConfig(const std::string& node_id) {
    auto it = nodes_.find(node_id);
    if (it == nodes_.end()) {
        return nullptr;
    }
    return &it->second;
}

// Caller must remember to check
const NodeConfig* config = mgr->GetNodeConfig("node01");
if (config == nullptr) {  // Easy to forget!
    // handle error
}
```

**Rust Option:**
```rust
pub fn get_node_config(&self, name: &str) -> Option<&NodeConfig> {
    self.nodes.get(name)
}

// Compiler forces handling
match mgr.get_node_config("node01") {
    Some(config) => { /* use config */ }
    None => { /* handle missing node */ }
}

// Or use ? operator
let config = mgr.get_node_config("node01")
    .ok_or_else(|| SchedulerError::NodeNotFound { node: "node01".to_string() })?;
```

**Benefits:**
- **Cannot Forget:** Compiler error if `Option` not handled
- **No Null Dereference:** Cannot access value without matching `Some`
- **Chaining:** `.map()`, `.and_then()`, `.unwrap_or()` combinators

---

## Error Propagation Examples

### Scheduler Error Flow

```rust
// Bottom layer: Admission control
fn assign_task_to_node(...) -> Result<(), AdmissionReason> {
    if utilization > threshold {
        return Err(AdmissionReason::CpuUtilizationExceeded { cpu, current, added, threshold });
    }
    Ok(())
}

// Middle layer: Algorithm
fn schedule_target_node_priority(...) -> Result<(), SchedulerError> {
    for task in tasks {
        assign_task_to_node(task, node)?  // Propagates AdmissionReason
            .map_err(|reason| SchedulerError::AdmissionRejected {
                task: task.name.clone(),
                node: node.clone(),
                reason,
            })?;
    }
    Ok(())
}

// Top layer: gRPC handler
async fn add_sched_info(...) -> Result<Response<ProtoResponse>, Status> {
    let map = scheduler.schedule(tasks, algorithm)
        .map_err(|e| match e {
            SchedulerError::NoTasks => Status::invalid_argument("no tasks"),
            SchedulerError::AdmissionRejected { task, reason } => {
                Status::resource_exhausted(format!("task '{}' rejected: {}", task, reason))
            }
            // ... map other variants
        })?;

    Ok(Response::new(ProtoResponse { status: 0 }))
}
```

**Error Flow:**
```
AdmissionReason::CpuUtilizationExceeded
  ↓ (wrapped)
SchedulerError::AdmissionRejected { task, node, reason }
  ↓ (mapped)
tonic::Status::resource_exhausted("task 'task_0' rejected: CPU 2 utilization would be ...")
  ↓ (sent over gRPC)
Pullpiri receives StatusCode 8 (RESOURCE_EXHAUSTED)
```

---

## Fault Recovery Strategies

### 1. Retry Logic (Fault Client)

**Pattern:** Exponential backoff for transient RPC failures

```rust
impl FaultNotifier for FaultClient {
    async fn notify_fault(&self, info: FaultNotification) -> Result<(), FaultError> {
        let mut retries = 0;
        const MAX_RETRIES: u32 = 3;

        loop {
            match self.stub.clone().notify_fault(request.clone()).await {
                Ok(response) => {
                    if response.into_inner().status != 0 {
                        return Err(FaultError::RemoteError(response.status));
                    }
                    return Ok(());
                }
                Err(e) if retries < MAX_RETRIES => {
                    retries += 1;
                    let delay = Duration::from_millis(100 * 2u64.pow(retries));
                    warn!("Fault notification failed (retry {}/{}), retrying in {:?}",
                          retries, MAX_RETRIES, delay);
                    tokio::time::sleep(delay).await;
                }
                Err(e) => return Err(FaultError::Rpc(e)),
            }
        }
    }
}
```

---

### 2. Graceful Degradation (Config Loading)

**Pattern:** Use default config if file loading fails

```rust
let node_config_mgr = Arc::new({
    let mut mgr = NodeConfigManager::new();
    match mgr.load_from_file(Path::new(&args.node_config)) {
        Ok(_) => info!("Node configuration loaded successfully"),
        Err(e) => {
            warn!("Failed to load config: {}. Using default configuration.", e);
            // mgr falls back to default_node internally
        }
    }
    mgr
});
```

---

### 3. Barrier Cancellation (SyncTimer)

**Pattern:** Cancel pending sync when new workload arrives

```rust
// SchedInfoService: Cancel old barrier
{
    let mut guard = self.workload_store.lock().await;
    if let Some(old_ws) = guard.as_ref() {
        let _ = old_ws.barrier_tx.send(BarrierStatus::Cancelled);
    }
    *guard = Some(new_workload_state);
}

// NodeService: Handle cancellation
loop {
    match *barrier_rx.borrow_and_update() {
        BarrierStatus::Cancelled => {
            return Err(Status::aborted("workload was replaced"));
        }
        // ... other cases
    }
}
```

---

## Logging Strategy

### Rust Structured Logging (`tracing` crate)

**Levels:**
- **ERROR:** Unrecoverable failures requiring intervention
- **WARN:** Degraded operation, retries, fallbacks
- **INFO:** Normal operation milestones
- **DEBUG:** Detailed state for troubleshooting

**Examples:**
```rust
// Error with context
error!(
    task = %task_name,
    node = %node_id,
    reason = %admission_reason,
    "Task admission rejected"
);

// Warning with values
warn!(
    hyperperiod_us = %hp,
    limit_us = %limit,
    "Hyperperiod exceeds recommended limit"
);

// Info with structured fields
info!(
    workload_id = %workload_id,
    task_count = tasks.len(),
    hyperperiod_ms = hyperperiod_us / 1_000,
    "Hyperperiod calculated"
);

// Debug with detailed state
debug!(
    node_id = %node_id,
    cpu = cpu_id,
    current_util = %util,
    added_util = %task_util,
    "Assigning task to CPU"
);
```

**Benefits:**
- **Structured:** Key-value pairs (JSON export possible)
- **Filterable:** Can filter by field values
- **Contextual:** Automatically includes span context

---

## Testing Error Paths

### C++ (Difficult)

```cpp
TEST_F(SchedulerTest, TaskRejection) {
    // Hard to trigger specific error without mocking
    GlobalScheduler scheduler(node_config);
    NodeSchedMap result;
    bool success = scheduler.ProcessScheduleInfo(bad_sched_info, result);

    EXPECT_FALSE(success);  // Which error? Unknown!
}
```

### Rust (Easy)

```rust
#[test]
fn test_task_rejection_cpu_utilization() {
    let config = Arc::new(NodeConfigManager::default());
    let scheduler = GlobalScheduler::new(config);

    let tasks = vec![
        Task {
            name: "overload".into(),
            target_node: "node01".into(),
            period_us: 10_000,
            runtime_us: 9_500,  // 95% utilization (exceeds 90% threshold)
            ..Default::default()
        },
    ];

    let result = scheduler.schedule(tasks, "target_node_priority");

    // Pattern match exact error
    assert!(matches!(
        result,
        Err(SchedulerError::AdmissionRejected {
            reason: AdmissionReason::CpuUtilizationExceeded { .. },
            ..
        })
    ));
}

#[test]
fn test_hyperperiod_overflow() {
    let mut mgr = HyperperiodManager::new();

    let tasks = vec![
        Task { period_us: u64::MAX, ..Default::default() },
        Task { period_us: 2, ..Default::default() },
    ];

    let result = mgr.calculate_hyperperiod("wl_1", &tasks);

    assert!(matches!(
        result,
        Err(HyperperiodError::Overflow { a: u64::MAX, b: 2 })
    ));
}
```

---

## Migration Notes

### What Changed

1. **Error Returns:** `bool` → `Result<T, E>`
2. **Sentinel Values:** `0`, `-1`, `NULL` → `Option<T>`
3. **Exceptions:** Removed → Result-based propagation
4. **Error Context:** Logs only → Structured error types
5. **Propagation:** Manual checks → `?` operator
6. **Type Safety:** Runtime → Compile-time

### What Stayed the Same

1. **Logging Philosophy:** Still log errors at appropriate levels
2. **Recovery Strategies:** Retry, fallback, graceful degradation
3. **Error Codes:** gRPC status codes map to same HTTP/2 codes

---

**Document Version:** 1.0
**Last Updated:** May 12, 2026
**Status:** ✅ Complete
**Verified Against:** `timpani_rust/timpani-o/src/scheduler/error.rs` and `src/fault/mod.rs`
