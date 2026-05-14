<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: eBPF Monitoring System

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-n-lld-07
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** Kernel Monitoring
**Responsibility:** Deadline miss detection, scheduler statistics via eBPF
**Status:** ⏸️ Not Started in Rust

---

## AS-IS: C Implementation

**Files:** `timpani-n/src/sigwait.bpf.c`, `timpani-n/src/schedstat.bpf.c`, `timpani-n/src/trace_bpf.c`

### sigwait.bpf.c - Deadline Monitoring

```c
SEC("tp/syscalls/sys_enter_rt_sigtimedwait")
int handle_sigwait_enter(struct trace_event_raw_sys_enter *ctx) {
    pid_t pid = bpf_get_current_pid_tgid() >> 32;

    // Check if PID is in filter map
    int *filtered = bpf_map_lookup_elem(&pid_filter_map, &pid);
    if (!filtered) return 0;

    // Record entry timestamp
    u64 ts = bpf_ktime_get_ns();
    struct sigwait_event event = {
        .pid = pid,
        .timestamp_ns = ts,
        .event_type = SIGWAIT_ENTER
    };

    bpf_ringbuf_output(&events, &event, sizeof(event), 0);
    return 0;
}

SEC("tp/syscalls/sys_exit_rt_sigtimedwait")
int handle_sigwait_exit(struct trace_event_raw_sys_exit *ctx) {
    // Similar logic for exit event
}
```

### Ring Buffer Handling (Userspace)

```c
int bpf_on(ring_buffer_sample_fn sigwait_cb,
          ring_buffer_sample_fn schedstat_cb,
          void *ctx) {
    struct sigwait_bpf *skel = sigwait_bpf__open_and_load();
    sigwait_bpf__attach(skel);

    struct ring_buffer *rb = ring_buffer__new(
        bpf_map__fd(skel->maps.events), sigwait_cb, ctx, NULL);

    return 0;
}

static int handle_sigwait_bpf_event(void *ctx, void *data, size_t size) {
    struct sigwait_event *event = data;
    struct context *timpani_ctx = ctx;

    // Find corresponding task
    struct time_trigger *tt = find_task_by_pid(timpani_ctx, event->pid);

    if (event->event_type == SIGWAIT_EXIT) {
        // Check if deadline was missed
        uint64_t elapsed_ns = event->timestamp_ns - tt->sigwait_ts;
        uint64_t deadline_ns = tt->deadline.tv_sec * 1000000000 + tt->deadline.tv_nsec;

        if (elapsed_ns > deadline_ns) {
            report_deadline_miss(timpani_ctx, tt->task.name);
        }
    }

    return 0;
}
```

---

## WILL-BE: Rust Implementation (⏸️ Not Started)

**Planned:**
- Use `aya` crate for eBPF in Rust
- Type-safe BPF program loading
- Async ring buffer polling

---

**Document Version:** 1.0
**Status:** C ✅, Rust ⏸️
