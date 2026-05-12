<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# HLD: Real-Time Scheduling

**Component Type:** RT Scheduling Control  
**Responsibility:** CPU affinity, RT priority, sched_setattr() syscalls  
**Status:** ⏸️ Not Started in Rust

---

## AS-IS: C Implementation

**File:** `timpani-n/src/sched.c`

### CPU Affinity

```c
ttsched_error_t set_affinity(pid_t pid, int cpu) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu, &cpuset);
    
    return sched_setaffinity(pid, sizeof(cpu_set_t), &cpuset) == 0
        ? TTSCHED_SUCCESS : TTSCHED_ERROR_SYSTEM;
}

ttsched_error_t set_affinity_cpumask(pid_t pid, uint64_t cpumask) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    
    for (int i = 0; i < 64; i++) {
        if (cpumask & (1ULL << i)) {
            CPU_SET(i, &cpuset);
        }
    }
    
    return sched_setaffinity(pid, sizeof(cpu_set_t), &cpuset) == 0
        ? TTSCHED_SUCCESS : TTSCHED_ERROR_SYSTEM;
}
```

### RT Priority

```c
ttsched_error_t set_schedattr(pid_t pid, unsigned int priority, unsigned int policy) {
    struct sched_param param;
    param.sched_priority = priority;
    
    return sched_setscheduler(pid, policy, &param) == 0
        ? TTSCHED_SUCCESS : TTSCHED_ERROR_PERMISSION;
}
```

---

## WILL-BE: Rust Implementation (⏸️ Not Started)

**Planned:**
- Use `nix` crate for `sched_setaffinity()`
- Rust-safe CPU set management
- RT priority via syscalls

---

**Document Version:** 1.0  
**Status:** C ✅, Rust ⏸️
