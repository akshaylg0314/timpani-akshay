<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: Resource Management

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-n-lld-09
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** Cleanup & State Management
**Responsibility:** Resource cleanup, global state, graceful shutdown
**Status:** ⏸️ Not Started in Rust

---

## AS-IS: C Implementation

**Files:** `timpani-n/src/cleanup.c`, `timpani-n/src/globals.c`

### Cleanup Function

```c
void cleanup_context(struct context *ctx) {
    // Stop BPF monitoring
    bpf_off();

    // Close timer file descriptors
    if (ctx->runtime.hyperperiod_timer_fd >= 0) {
        close(ctx->runtime.hyperperiod_timer_fd);
    }

    // Close D-Bus connection
    if (ctx->runtime.dbus) {
        sd_bus_unref(ctx->runtime.dbus);
    }

    // Free task list
    if (ctx->runtime.tt_list) {
        free(ctx->runtime.tt_list);
    }

    // Free schedule info
    destroy_task_info_list(ctx->sinfo.tasks);
}
```

### Global State

```c
static struct context *g_ctx = NULL;  // For signal handlers

void set_global_context(struct context *ctx) {
    g_ctx = ctx;
}
```

---

## WILL-BE: Rust Implementation (⏸️ Not Started)

**Planned:**
- RAII-style cleanup (Drop trait)
- No global mutable state
- Structured resource ownership

---

**Document Version:** 1.0
**Status:** C ✅, Rust ⏸️
