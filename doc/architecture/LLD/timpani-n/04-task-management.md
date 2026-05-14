<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: Task Management

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-n-lld-04
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** Task Lifecycle Management
**Responsibility:** Task list management, activation scheduling, state tracking
**Status:** ⏸️ Not Started in Rust

---

## AS-IS: C Implementation

**File:** `timpani-n/src/task.c`

### Task Structure

```c
struct time_trigger {
    struct task_info task;      // Task metadata
    struct timespec period;     // Execution period
    struct timespec deadline;   // Deadline
    uint64_t sigwait_ts;        // Last signal timestamp
    bool sigwait_enter;         // Signal entry flag
    struct context *ctx;        // Back-pointer to context
};
```

### Task List Initialization

```c
tt_error_t init_task_list(struct context *ctx) {
    int task_count = ctx->sinfo.task_count;

    ctx->runtime.tt_list = calloc(task_count, sizeof(struct time_trigger));

    for (int i = 0; i < task_count; i++) {
        struct task_info *task = &ctx->sinfo.tasks[i];
        struct time_trigger *tt = &ctx->runtime.tt_list[i];

        tt->task = *task;
        tt->period.tv_sec = task->period_us / 1000000;
        tt->period.tv_nsec = (task->period_us % 1000000) * 1000;
        tt->ctx = ctx;

        // Add PID to BPF filter
        bpf_add_pid(task->pid);
    }

    return TT_SUCCESS;
}
```

### Task Activation

```c
static void activate_task(struct time_trigger *tt) {
    int pidfd = tt->task.pidfd;
    send_signal_pidfd(pidfd, SIGNO_TT);  // Send trigger signal
}
```

---

## WILL-BE: Rust Implementation (⏸️ Not Started)

**Planned:**
- Task list as `Vec<TimeTrigger>`
- Async task activation
- Safe PID handling

---

**Document Version:** 1.0
**Status:** C ✅, Rust ⏸️
