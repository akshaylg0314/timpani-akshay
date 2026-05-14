<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: Scheduler Utilities Component

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-o-lld-07
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** Helper Functions & Utilities
**Responsibility:** Provide reusable scheduling utilities, feasibility checks, and mathematical functions
**Status:** ✅ Migrated (C++ → Rust)

## Component Overview

Scheduler Utilities component provides helper functions used by the GlobalScheduler and HyperperiodManager, including feasibility analysis (Liu & Layland bounds), mathematical utilities (GCD/LCM), and CPU utilization calculations.

---

## As-Is: C++ Implementation

### Utility Functions (C++)

```cpp
// Namespace or free functions
namespace timpani {

// GCD calculation (Euclidean algorithm)
uint64_t CalculateGCD(uint64_t a, uint64_t b);

// LCM calculation
uint64_t CalculateLCM(uint64_t a, uint64_t b);

// CPU utilization
double CalculateCpuUtilization(const std::vector<Task>& tasks_on_cpu);

// Total utilization for a node
double CalculateNodeUtilization(const std::map<int, std::vector<Task>>& cpu_map);

// Helper: Find minimum element
template<typename T>
typename T::const_iterator FindMin(const T& container);

}
```

---

## Will-Be: Rust Implementation

### 1. Feasibility Analysis

**File:** `timpani_rust/timpani-o/src/scheduler/feasibility.rs`

```rust
/// Compute Liu & Layland utilisation upper bound for `n` tasks.
///
/// U_bound(n) = n × (2^(1/n) − 1)
pub fn liu_layland_bound(n: usize) -> f64 {
    if n == 0 {
        return 0.0;
    }
    let nf = n as f64;
    nf * (2.0_f64.powf(1.0 / nf) - 1.0)
}

/// Check whether tasks satisfy Liu & Layland schedulability bound.
///
/// Returns `None` if provably schedulable (U ≤ bound).
/// Returns `Some(total_u)` if bound exceeded (warning).
pub fn check_liu_layland(tasks_on_node: &[&Task]) -> Option<f64> {
    let feasible: Vec<&Task> = tasks_on_node
        .iter()
        .copied()
        .filter(|t| t.period_us > 0)
        .collect();

    if feasible.is_empty() {
        return None;
    }

    let total_u: f64 = feasible
        .iter()
        .map(|t| t.runtime_us as f64 / t.period_us as f64)
        .sum();

    let bound = liu_layland_bound(feasible.len());

    if total_u > bound {
        Some(total_u)
    } else {
        None
    }
}
```

**Usage:**
```rust
if let Some(total_u) = check_liu_layland(&tasks_on_node) {
    warn!(
        node_id = %node_id,
        utilization = %total_u,
        bound = %liu_layland_bound(tasks_on_node.len()),
        "Liu & Layland bound exceeded — RTA recommended"
    );
}
```

---

### 2. Mathematical Utilities

**File:** `timpani_rust/timpani-o/src/hyperperiod/math.rs`

```rust
/// Greatest Common Divisor (Euclidean algorithm)
pub fn gcd(mut a: u64, mut b: u64) -> u64 {
    while b != 0 {
        let temp = b;
        b = a % b;
        a = temp;
    }
    a
}

/// Least Common Multiple with overflow detection
pub fn lcm(a: u64, b: u64) -> Result<u64, HyperperiodError> {
    if a == 0 || b == 0 {
        return Ok(0);
    }

    let g = gcd(a, b);
    let quotient = a / g;

    quotient.checked_mul(b).ok_or_else(|| {
        HyperperiodError::Overflow { a, b }
    })
}

/// LCM of multiple values
pub fn lcm_of_slice(periods: &[u64]) -> Result<u64, HyperperiodError> {
    periods.iter().try_fold(1u64, |acc, &p| lcm(acc, p))
}
```

---

### 3. CPU Utilization Helpers

**Integrated in `scheduler/mod.rs`:**

```rust
// Build CPU utilization map
fn build_cpu_utilization(avail: &AvailCpus) -> CpuUtil {
    let mut util = BTreeMap::new();
    for (node_id, cpus) in avail {
        let mut cpu_map = BTreeMap::new();
        for &cpu in cpus {
            cpu_map.insert(cpu, 0.0);
        }
        util.insert(node_id.clone(), cpu_map);
    }
    util
}

// Update utilization after assignment
fn update_cpu_utilization(
    node_id: &str,
    cpu: u32,
    task: &Task,
    util: &mut CpuUtil,
) {
    let task_util = task.runtime_us as f64 / task.period_us as f64;
    *util.get_mut(node_id).unwrap().get_mut(&cpu).unwrap() += task_util;
}

// Find least loaded node
fn find_least_loaded_node(util: &CpuUtil) -> Option<String> {
    util.iter()
        .map(|(node_id, cpu_map)| {
            let total_u: f64 = cpu_map.values().sum();
            (node_id, total_u)
        })
        .min_by(|(_, u1), (_, u2)| u1.partial_cmp(u2).unwrap())
        .map(|(node_id, _)| node_id.clone())
}
```

---

## As-Is vs Will-Be Comparison

| Utility | C++ (As-Is) | Rust (Will-Be) |
|---------|-------------|----------------|
| **GCD** | `uint64_t CalculateGCD(a, b)` | `pub fn gcd(a: u64, b: u64) -> u64` |
| **LCM** | `uint64_t CalculateLCM(a, b)` (silent overflow) | `pub fn lcm(a, b) -> Result<u64, E>` (checked) |
| **Liu & Layland** | Not implemented | `pub fn liu_layland_bound(n) -> f64` |
| **Feasibility Check** | Not implemented | `pub fn check_liu_layland(&[&Task]) -> Option<f64>` |
| **CPU Utilization** | `double CalculateCpuUtilization(...)` | Integrated in scheduler as methods |
| **Organization** | Free functions in namespace | Modules (`feasibility.rs`, `math.rs`) |

---

## Design Decisions

### D-UTIL-001: Module Organization

**C++ (Scattered):**
```cpp
// Some in scheduler.cpp
// Some in hyperperiod.cpp
// Some in utils.cpp
namespace timpani {
    uint64_t CalculateGCD(...);
    double CalculateCpuUtilization(...);
}
```

**Rust (Organized by Domain):**
```
src/
  scheduler/
    mod.rs         ← Main scheduler logic
    feasibility.rs ← Liu & Layland utilities
    error.rs       ← Error types
  hyperperiod/
    mod.rs         ← Hyperperiod manager
    math.rs        ← GCD/LCM utilities
```

**Rationale:** Group utilities by domain for better discoverability and testing.

---

### D-UTIL-002: Liu & Layland Implementation

**Formula:**
$$U_{\text{bound}}(n) = n \left(2^{1/n} - 1\right)$$

**Implementation:**
```rust
pub fn liu_layland_bound(n: usize) -> f64 {
    if n == 0 {
        return 0.0;
    }
    let nf = n as f64;
    nf * (2.0_f64.powf(1.0 / nf) - 1.0)
}
```

**Test Cases:**
```rust
#[test]
fn bound_one_task_is_one() {
    assert_eq!(liu_layland_bound(1), 1.0);
}

#[test]
fn bound_two_tasks_is_approximately_0_828() {
    let b = liu_layland_bound(2);
    assert!((b - 0.8284).abs() < 1e-3);
}

#[test]
fn bound_converges_toward_ln2() {
    let b = liu_layland_bound(1000);
    assert!((b - 2.0_f64.ln()).abs() < 1e-3);  // ln(2) ≈ 0.6931
}
```

---

### D-UTIL-003: Checked Arithmetic

**C++ (Unchecked):**
```cpp
uint64_t CalculateLCM(uint64_t a, uint64_t b) {
    uint64_t gcd = CalculateGCD(a, b);
    return (a / gcd) * b;  // Can overflow silently!
}
```

**Rust (Checked):**
```rust
pub fn lcm(a: u64, b: u64) -> Result<u64, HyperperiodError> {
    let g = gcd(a, b);
    let quotient = a / g;

    quotient.checked_mul(b).ok_or_else(|| {
        HyperperiodError::Overflow { a, b }
    })
}
```

**Benefits:**
- **Explicit:** Caller must handle `Err(Overflow)`
- **Context:** Error includes operands that caused overflow
- **Safe:** Cannot silently wrap around

---

## Testing

### C++ Testing

```cpp
TEST(UtilsTest, GCD) {
    EXPECT_EQ(CalculateGCD(48, 18), 6);
}

TEST(UtilsTest, LCM) {
    EXPECT_EQ(CalculateLCM(4, 6), 12);
    // Cannot test overflow easily
}
```

### Rust Testing

```rust
#[test]
fn test_gcd() {
    assert_eq!(gcd(48, 18), 6);
    assert_eq!(gcd(0, 5), 5);
    assert_eq!(gcd(5, 0), 5);
}

#[test]
fn test_lcm_success() {
    assert_eq!(lcm(4, 6).unwrap(), 12);
    assert_eq!(lcm(10, 15).unwrap(), 30);
}

#[test]
fn test_lcm_overflow_detection() {
    let result = lcm(u64::MAX, 2);
    assert!(matches!(result, Err(HyperperiodError::Overflow { .. })));
}

#[test]
fn test_liu_layland_classic_example() {
    // From Liu & Layland's 1973 paper:
    //   Task A: T=10ms, C=3ms  → U=0.30
    //   Task B: T=20ms, C=5ms  → U=0.25
    //   Task C: T=50ms, C=8ms  → U=0.16
    //   Total U = 0.71, bound(3) ≈ 0.780 → FEASIBLE
    let a = Task { period_us: 10_000, runtime_us: 3_000, ..Default::default() };
    let b = Task { period_us: 20_000, runtime_us: 5_000, ..Default::default() };
    let c = Task { period_us: 50_000, runtime_us: 8_000, ..Default::default() };

    let result = check_liu_layland(&[&a, &b, &c]);

    assert!(result.is_none(), "Should be feasible");
}
```

---

## Usage Examples

### 1. Feasibility Check in Scheduler

```rust
impl GlobalScheduler {
    fn run_liu_layland_check(&self, tasks: &[Task]) {
        // Group tasks by node
        let mut node_tasks: HashMap<&str, Vec<&Task>> = HashMap::new();
        for task in tasks {
            node_tasks.entry(&task.assigned_node).or_default().push(task);
        }

        // Check each node
        for (node_id, tasks_on_node) in node_tasks {
            if let Some(total_u) = check_liu_layland(&tasks_on_node) {
                warn!(
                    node_id = %node_id,
                    utilization = %total_u,
                    bound = %liu_layland_bound(tasks_on_node.len()),
                    task_count = tasks_on_node.len(),
                    "Liu & Layland bound exceeded — Response Time Analysis recommended"
                );
            }
        }
    }
}
```

---

### 2. Hyperperiod Calculation

```rust
let unique_periods = vec![10_000, 20_000, 30_000];

match lcm_of_slice(&unique_periods) {
    Ok(hp) => info!("Hyperperiod: {}µs", hp),  // 60,000
    Err(HyperperiodError::Overflow { a, b }) => {
        error!("LCM overflow: lcm({}, {})", a, b);
    }
}
```

---

### 3. CPU Assignment

```rust
fn find_best_cpu_for_task(
    task: &Task,
    node_id: &str,
    avail: &AvailCpus,
    util: &CpuUtil,
) -> Result<u32, SchedulerError> {
    let node_cpus = avail.get(node_id).ok_or(...)?;

    // Filter by affinity
    let allowed: Vec<u32> = node_cpus.iter()
        .filter(|&&cpu| task.affinity.allows_cpu(cpu))
        .copied()
        .collect();

    // Find CPU with lowest utilization
    let best_cpu = allowed.iter()
        .min_by(|a, b| {
            let u_a = util[node_id].get(a).unwrap_or(&0.0);
            let u_b = util[node_id].get(b).unwrap_or(&0.0);
            u_a.partial_cmp(u_b).unwrap()
        })
        .copied()
        .ok_or(SchedulerError::NoAvailableCpu)?;

    Ok(best_cpu)
}
```

---

## Migration Notes

### What Changed

1. **Organization:** Scattered functions → Domain-specific modules
2. **Overflow Handling:** Silent → Checked arithmetic with `Result`
3. **Feasibility:** Not implemented → Liu & Layland bounds
4. **Type Safety:** Free functions → Module-scoped public functions
5. **Testing:** Limited → Comprehensive unit tests

### What Stayed the Same

1. **Algorithms:** GCD (Euclidean), LCM formula unchanged
2. **Utilization Calculation:** `runtime / period` logic identical
3. **Semantics:** Same mathematical operations

---

**Document Version:** 1.0
**Last Updated:** May 12, 2026
**Status:** ✅ Complete
**Verified Against:** `timpani_rust/timpani-o/src/scheduler/feasibility.rs` and `src/hyperperiod/math.rs`
