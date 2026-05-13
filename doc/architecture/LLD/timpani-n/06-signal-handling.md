<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: Signal Handling

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-n-lld-06
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** Signal Management
**Responsibility:** SIGALRM handlers, task signal delivery, shutdown signals
**Status:** ⏸️ Not Started in Rust

---

## AS-IS: C Implementation

**File:** `timpani-n/src/signal.c`

### Signal Setup

```c
tt_error_t setup_signal_handlers(struct context *ctx) {
    struct sigaction sa;

    // SIGINT/SIGTERM: Graceful shutdown
    sa.sa_handler = signal_handler_shutdown;
    sa.sa_flags = 0;
    sigemptyset(&sa.sa_mask);
    sigaction(SIGINT, &sa, NULL);
    sigaction(SIGTERM, &sa, NULL);

    // SIGALRM: Task activation timer
    sa.sa_handler = signal_handler_alarm;
    sa.sa_flags = SA_RESTART;
    sigaction(SIGALRM, &sa, NULL);

    return TT_SUCCESS;
}

static void signal_handler_shutdown(int sig) {
    g_ctx->shutdown_requested = true;
}

static void signal_handler_alarm(int sig) {
    // Timer tick - handled in epoll loop
}
```

### Task Signal Delivery

```c
tt_error_t send_signal_pidfd(int pidfd, int signal) {
    struct siginfo info = {0};
    info.si_signo = signal;
    info.si_code = SI_QUEUE;

    return syscall(__NR_pidfd_send_signal, pidfd, signal, &info, 0) == 0
        ? TT_SUCCESS : TT_ERROR_SIGNAL;
}
```

---

## WILL-BE: Rust Implementation (⏸️ Not Started)

**Planned:**
- Use `tokio::signal` for async signal handling
- Safe signal delivery via `pidfd`

---

**Document Version:** 1.0
**Status:** C ✅, Rust ⏸️
