<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: Time Trigger Core

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-n-lld-03
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** Core Runtime Engine
**Responsibility:** Event loop, hyperperiod management, timer coordination
**Status:** ⏸️ Not Started in Rust (C implementation documented)

---

## AS-IS: C Implementation

**Files:** `timpani-n/src/core.c`, `timpani-n/src/hyperperiod.c`

### Hyperperiod Calculation

```c
tt_error_t init_hyperperiod(struct context *ctx,
                            const char *workload_id,
                            uint64_t hyperperiod_us,
                            struct hyperperiod_manager *hp_mgr) {
    hp_mgr->hyperperiod_us = hyperperiod_us;
    hp_mgr->hp_count = 0;
    strncpy(hp_mgr->workload_id, workload_id, sizeof(hp_mgr->workload_id) - 1);

    clock_gettime(CLOCK_MONOTONIC, &hp_mgr->hp_timer_start);
    return TT_SUCCESS;
}
```

### Event Loop (epoll-based)

```c
tt_error_t epoll_loop(struct context *ctx) {
    int epfd = epoll_create1(0);

    while (!ctx->shutdown_requested) {
        int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);

        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == ctx->runtime.hyperperiod_timer_fd) {
                handle_hyperperiod_tick(ctx);
            } else if (events[i].data.fd == ctx->runtime.bpf_ringbuf_fd) {
                ring_buffer__poll(ctx->runtime.rb, 0);
            }
        }
    }

    return TT_SUCCESS;
}
```

### Timer Management

```c
tt_error_t start_hyperperiod_timer(struct context *ctx) {
    struct itimerspec its;
    its.it_interval.tv_sec = 0;
    its.it_interval.tv_nsec = ctx->hp_manager.hyperperiod_us * 1000;
    its.it_value = its.it_interval;

    return timerfd_settime(ctx->runtime.hyperperiod_timer_fd, 0, &its, NULL) == 0
        ? TT_SUCCESS : TT_ERROR_TIMER;
}
```

---

## WILL-BE: Rust Implementation (⏸️ Not Started)

**Planned Design:**
- Use `tokio::time::interval()` for periodic timers
- Async event loop instead of epoll
- Hyperperiod calculation using checked arithmetic

**Status:** Architecture defined, no code yet

---

**Document Version:** 1.0
**Status:** C ✅, Rust ⏸️
**Verified Against:** `timpani-n/src/core.c`, `timpani-n/src/hyperperiod.c`
