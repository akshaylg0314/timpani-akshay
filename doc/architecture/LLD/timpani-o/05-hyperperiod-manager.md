<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: Hyperperiod Manager Component

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-o-lld-05
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** Mathematical Utility
**Responsibility:** Calculate Least Common Multiple (LCM) of task periods for hyperperiod determination
**Status:** ✅ Migrated (C++ → Rust)

## Component Overview

The Hyperperiod Manager calculates the hyperperiod for a set of periodic tasks. The hyperperiod is the Least Common Multiple (LCM) of all task periods, representing the smallest time window after which the entire task set repeats its execution pattern.

---

## As-Is: C++ Implementation

### Class Structure

```cpp
class HyperperiodManager {
public:
    HyperperiodManager();

    uint64_t CalculateHyperperiod(const std::string& workload_id,
                                  const std::vector<Task>& tasks);

    const HyperperiodInfo* GetHyperperiodInfo(const std::string& workload_id) const;

private:
    uint64_t CalculateLCM(uint64_t a, uint64_t b);
    uint64_t CalculateGCD(uint64_t a, uint64_t b);

    std::map<std::string, HyperperiodInfo> hyperperiod_map_;
};
```

### Responsibilities (C++)

1. **Calculate** LCM of all task periods
2. **Store** hyperperiod information per workload
3. **Validate** against sanity thresholds (1 hour warning)
4. **Track** unique periods and task counts

### Key Features (C++)

- **Algorithm:** Euclidean GCD + LCM formula `lcm(a,b) = (a × b) / gcd(a,b)`
- **Sanity Check:** Logs warning if hyperperiod > 1 hour (3,600,000,000 µs)
- **Storage:** Maintains internal map of workload → HyperperiodInfo
- **Return Value:** `0` for both "no tasks" and "overflow" (ambiguous)

### Design Issues (C++)

| Issue | Impact |
|-------|--------|
| `CalculateHyperperiod` returns `0` for "no tasks" and "overflow" | Caller cannot distinguish failures |
| `(a / gcd) * b` can overflow silently | Incorrect results without detection |
| Warning-only sanity check | Scheduler proceeds with multi-hour hyperperiod |
| Copies entire vector for filtering | Performance overhead |

---

## Will-Be: Rust Implementation

### Module Structure

```rust
// File: timpani_rust/timpani-o/src/hyperperiod/mod.rs

pub struct HyperperiodManager {
    limit_us: u64,
    history: HashMap<String, HyperperiodInfo>,
}

impl HyperperiodManager {
    pub fn new() -> Self {
        Self {
            limit_us: DEFAULT_HYPERPERIOD_LIMIT_US,
            history: HashMap::new(),
        }
    }

    pub fn with_limit(limit_us: u64) -> Self {
        Self {
            limit_us,
            history: HashMap::new(),
        }
    }

    pub fn calculate_hyperperiod(
        &mut self,
        workload_id: &str,
        tasks: &[Task],
    ) -> Result<&HyperperiodInfo, HyperperiodError> {
        // Extract unique non-zero periods
        let unique_periods = extract_unique_periods(tasks);

        if unique_periods.is_empty() {
            return Err(HyperperiodError::NoValidPeriods);
        }

        // Calculate LCM with overflow detection
        let hyperperiod_us = lcm_of_slice(&unique_periods)?;

        // Check limit
        if hyperperiod_us > self.limit_us {
            return Err(HyperperiodError::TooLarge {
                value_us: hyperperiod_us,
                limit_us: self.limit_us,
            });
        }

        // Store and return
        let info = HyperperiodInfo {
            workload_id: workload_id.to_owned(),
            hyperperiod_us,
            unique_periods: unique_periods.clone(),
            task_count: tasks.len(),
        };

        self.history.insert(workload_id.to_owned(), info);
        Ok(self.history.get(workload_id).unwrap())
    }
}
```

### Responsibilities (Rust)

1. **Extract** unique non-zero periods from `&[Task]` (zero-copy iterator)
2. **Calculate** LCM using checked multiplication (overflow detection)
3. **Validate** against configurable limit (default 1 hour)
4. **Return** `Result<&HyperperiodInfo, HyperperiodError>` with specific error variants
5. **Cache** results in internal `HashMap`

### Key Features (Rust)

- **Overflow Detection:** `checked_mul()` returns `Err(Overflow { a, b })`
- **Configurable Limit:** `with_limit()` constructor for custom thresholds
- **Zero-Copy:** `&[Task]` borrow + `filter` iterator (no vector copies)
- **Structured Errors:** Each failure case is a distinct enum variant
- **Type Safety:** Cannot misuse `0` as valid result

---

## As-Is vs Will-Be Comparison

| Aspect | C++ (As-Is) | Rust (Will-Be) |
|--------|-------------|----------------|
| **Error Handling** | `0` for both "no tasks" and "overflow" | `Result<T, HyperperiodError>` with distinct variants |
| **Overflow Detection** | Silent overflow in `(a / gcd) * b` | `checked_mul` → `Err(Overflow { a, b })` |
| **Sanity Check** | Warning only (proceeds anyway) | `Err(TooLarge)` - caller decides |
| **Period Extraction** | Copy vector + filter | Zero-copy `&[Task]` + iterator |
| **Limit Configuration** | Hard-coded 1 hour | Configurable via `with_limit()` |
| **Failure Context** | No information in return value | Error variants include operands/limits |
| **Return Type** | `uint64_t` (0 = error) | `Result<&HyperperiodInfo, E>` |
| **Memory Management** | `std::map` with copying | `HashMap` with owned values |

---

## Design Decisions

### D-HP-001: Result Type Instead of Sentinel Value

**C++ Approach:**
```cpp
uint64_t CalculateHyperperiod(...) {
    if (unique_periods.empty()) {
        return 0; // No tasks
    }
    uint64_t lcm = CalculateLCM(...);
    if (lcm == 0) {
        return 0; // Overflow occurred
    }
    if (lcm > LIMIT) {
        LOG_WARNING("Hyperperiod too large");
        // Return anyway - just a warning
    }
    return lcm;
}
```

**Issue:** Caller sees `0` and cannot distinguish:
- No valid periods?
- Overflow during LCM?
- Actual hyperperiod of 0 µs (impossible but type allows it)?

**Rust Approach:**
```rust
pub enum HyperperiodError {
    NoValidPeriods,
    Overflow { a: u64, b: u64 },
    TooLarge { value_us: u64, limit_us: u64 },
}

pub fn calculate_hyperperiod(...) -> Result<&HyperperiodInfo, HyperperiodError> {
    if unique_periods.is_empty() {
        return Err(HyperperiodError::NoValidPeriods);
    }

    let hp = lcm_of_slice(&unique_periods)?; // Propagates Overflow

    if hp > self.limit_us {
        return Err(HyperperiodError::TooLarge {
            value_us: hp,
            limit_us: self.limit_us,
        });
    }

    Ok(info)
}
```

**Benefits:**
- **Clear Failures:** Each error case has distinct variant
- **Actionable Context:** Error includes operands that overflowed, or actual/limit values
- **Type Safety:** Cannot accidentally treat error as valid hyperperiod

---

### D-HP-002: Checked Arithmetic for Overflow

**C++ LCM Calculation:**
```cpp
uint64_t CalculateLCM(uint64_t a, uint64_t b) {
    if (a == 0 || b == 0) return 0;

    uint64_t gcd = CalculateGCD(a, b);
    // This can overflow silently!
    return (a / gcd) * b;
}
```

**Problem:** If `(a / gcd) * b` exceeds `UINT64_MAX`, result wraps around silently.

**Rust LCM Calculation:**
```rust
pub fn lcm(a: u64, b: u64) -> Result<u64, HyperperiodError> {
    if a == 0 || b == 0 {
        return Ok(0);
    }

    let g = gcd(a, b);
    let quotient = a / g;

    // checked_mul returns None on overflow
    quotient.checked_mul(b).ok_or_else(|| {
        HyperperiodError::Overflow { a, b }
    })
}

pub fn lcm_of_slice(periods: &[u64]) -> Result<u64, HyperperiodError> {
    periods.iter().try_fold(1u64, |acc, &p| lcm(acc, p))
}
```

**Benefits:**
- **Explicit Detection:** `checked_mul()` returns `None` on overflow
- **Error Context:** Includes `a` and `b` that caused overflow
- **Safe Propagation:** `?` operator propagates errors up the call chain

---

### D-HP-003: Zero-Copy Period Extraction

**C++ Approach:**
```cpp
std::vector<uint64_t> unique_periods;
for (const auto& task : tasks) {
    if (task.period_us > 0 &&
        std::find(unique_periods.begin(), unique_periods.end(), task.period_us) == unique_periods.end()) {
        unique_periods.push_back(task.period_us);
    }
}
// Entire filtered vector is created - O(n) memory
```

**Rust Approach:**
```rust
fn extract_unique_periods(tasks: &[Task]) -> Vec<u64> {
    let mut periods: Vec<u64> = tasks
        .iter()  // Iterator - no copy
        .map(|t| t.period_us)
        .filter(|&p| p > 0)
        .collect();  // Only allocate final result

    periods.sort_unstable();
    periods.dedup();
    periods
}
```

**Benefits:**
- **Zero-Copy:** `tasks` is borrowed (`&[Task]`), not moved
- **Lazy Evaluation:** `iter().map().filter()` chains without intermediate allocations
- **Single Allocation:** Only `collect()` allocates memory for final result

---

## Error Handling

### Error Enum

```rust
#[derive(Debug, PartialEq, Eq)]
pub enum HyperperiodError {
    /// The task slice was empty (or all tasks had `period_us == 0`).
    NoValidPeriods,

    /// LCM calculation overflowed `u64`.
    Overflow { a: u64, b: u64 },

    /// The calculated hyperperiod exceeded the configured limit.
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

### Error Display Examples

```
no tasks with a valid (non-zero) period

LCM overflow computing lcm(18446744073709551615, 2)

hyperperiod 7200000000µs (7200.0s) exceeds limit 3600000000µs (3600.0s)
```

---

## HyperperiodInfo Structure

### C++ Structure

```cpp
struct HyperperiodInfo {
    std::string workload_id;
    uint64_t hyperperiod_us;
    std::vector<uint64_t> unique_periods;
    size_t task_count;
};
```

### Rust Structure

```rust
#[derive(Debug, Clone)]
pub struct HyperperiodInfo {
    pub workload_id: String,
    pub hyperperiod_us: u64,
    pub unique_periods: Vec<u64>,
    pub task_count: usize,
}
```

**Identical fields** - direct translation.

---

## Algorithm: Euclidean GCD

### Implementation (Rust)

```rust
/// Euclidean algorithm for Greatest Common Divisor.
pub fn gcd(mut a: u64, mut b: u64) -> u64 {
    while b != 0 {
        let temp = b;
        b = a % b;
        a = temp;
    }
    a
}
```

**Example:**
```
gcd(48, 18)
  → 48 % 18 = 12
  → 18 % 12 = 6
  → 12 % 6 = 0
  → gcd = 6
```

---

## Algorithm: LCM Formula

### Formula

$$\text{lcm}(a, b) = \frac{a \times b}{\gcd(a, b)} = \left(\frac{a}{\gcd(a, b)}\right) \times b$$

**Why divide first?**
- Reduces magnitude before multiplication
- Minimizes overflow risk
- `(a / gcd) < a` always

### Implementation (Rust)

```rust
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
```

### Multi-Value LCM

```rust
pub fn lcm_of_slice(periods: &[u64]) -> Result<u64, HyperperiodError> {
    periods.iter().try_fold(1u64, |acc, &p| lcm(acc, p))
}
```

**Explanation:**
- Start with `acc = 1`
- For each period `p`: compute `acc = lcm(acc, p)`
- `try_fold` short-circuits on first error
- Final `acc` is LCM of all periods

**Example:**
```rust
periods = [10, 20, 30]
  acc = 1
  acc = lcm(1, 10)  = 10
  acc = lcm(10, 20) = 20
  acc = lcm(20, 30) = 60  ← hyperperiod
```

---

## Limits and Thresholds

### Default Limit

```rust
pub const DEFAULT_HYPERPERIOD_LIMIT_US: u64 = 3_600_000_000; // 1 hour
```

### Configurable Limit

```rust
let mgr = HyperperiodManager::with_limit(7_200_000_000); // 2 hours
```

### Overflow Limit

Maximum possible `u64` value:
```
u64::MAX = 18,446,744,073,709,551,615 µs
         ≈ 18,446,744 seconds
         ≈ 213 days
```

Practically, hyperperiods > 1 hour are usually configuration errors.

---

## Usage Example

### C++ Usage

```cpp
HyperperiodManager hp_mgr;
uint64_t hyperperiod = hp_mgr.CalculateHyperperiod("wl_001", tasks);

if (hyperperiod == 0) {
    // Error - but what kind?
    LOG_ERROR("Hyperperiod calculation failed");
    return false;
}

const HyperperiodInfo* info = hp_mgr.GetHyperperiodInfo("wl_001");
```

### Rust Usage

```rust
let mut hp_mgr = HyperperiodManager::new();

match hp_mgr.calculate_hyperperiod("wl_001", &tasks) {
    Ok(info) => {
        info!(
            workload_id = %info.workload_id,
            hyperperiod_ms = info.hyperperiod_us / 1_000,
            task_count = info.task_count,
            "Hyperperiod calculated"
        );
    }
    Err(HyperperiodError::Overflow { a, b }) => {
        error!("LCM overflow: lcm({}, {})", a, b);
        return Err(Status::invalid_argument("hyperperiod overflow"));
    }
    Err(HyperperiodError::TooLarge { value_us, limit_us }) => {
        warn!("Hyperperiod {}s exceeds {}s - rejecting",
              value_us / 1_000_000, limit_us / 1_000_000);
        return Err(Status::invalid_argument("hyperperiod too large"));
    }
    Err(HyperperiodError::NoValidPeriods) => {
        error!("No tasks with valid periods");
        return Err(Status::invalid_argument("no valid periods"));
    }
}
```

---

## Testing

### C++ Testing

```cpp
TEST_F(HyperperiodManagerTest, CalculateHyperperiod) {
    HyperperiodManager mgr;

    std::vector<Task> tasks = { ... };
    uint64_t result = mgr.CalculateHyperperiod("wl_1", tasks);

    EXPECT_GT(result, 0); // Cannot distinguish errors
}
```

### Rust Testing

```rust
#[test]
fn test_lcm_overflow_detection() {
    let a = u64::MAX;
    let b = 2;

    let result = lcm(a, b);

    assert!(matches!(
        result,
        Err(HyperperiodError::Overflow { a: u64::MAX, b: 2 })
    ));
}

#[test]
fn test_hyperperiod_too_large() {
    let mut mgr = HyperperiodManager::with_limit(1_000_000); // 1 second

    let tasks = vec![
        Task { period_us: 500_000, ..Default::default() },
        Task { period_us: 700_000, ..Default::default() },
    ];
    // lcm(500000, 700000) = 3,500,000 > 1,000,000 limit

    let result = mgr.calculate_hyperperiod("wl_1", &tasks);

    assert!(matches!(
        result,
        Err(HyperperiodError::TooLarge { value_us: 3_500_000, .. })
    ));
}

#[test]
fn test_classic_periods() {
    let mut mgr = HyperperiodManager::new();

    let tasks = vec![
        Task { period_us: 10_000, ..Default::default() },
        Task { period_us: 20_000, ..Default::default() },
        Task { period_us: 30_000, ..Default::default() },
    ];
    // lcm(10000, 20000, 30000) = 60000

    let result = mgr.calculate_hyperperiod("wl_1", &tasks).unwrap();

    assert_eq!(result.hyperperiod_us, 60_000);
    assert_eq!(result.unique_periods, vec![10_000, 20_000, 30_000]);
    assert_eq!(result.task_count, 3);
}
```

---

## Migration Notes

### What Changed

1. **Return Type:** `uint64_t` → `Result<&HyperperiodInfo, HyperperiodError>`
2. **Overflow Handling:** Silent → Explicit `checked_mul()`
3. **Limit Enforcement:** Warning → Error (caller decides)
4. **Period Extraction:** Vector copy → Zero-copy iterator
5. **Error Clarity:** Sentinel `0` → Typed error variants

### What Stayed the Same

1. **Algorithm:** Euclidean GCD + LCM formula unchanged
2. **Data Structure:** `HyperperiodInfo` fields identical
3. **Default Limit:** 1 hour (3,600,000,000 µs)
4. **Business Logic:** Same calculation steps

---

**Document Version:** 1.0
**Last Updated:** May 12, 2026
**Status:** ✅ Complete
**Verified Against:** `timpani_rust/timpani-o/src/hyperperiod/mod.rs` (actual implementation)
