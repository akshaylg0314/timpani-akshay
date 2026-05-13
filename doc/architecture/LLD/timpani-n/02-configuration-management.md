<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: Configuration Management

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-n-lld-02
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** Configuration System
**Responsibility:** CLI parsing, configuration validation, defaults management
**Status:** ✅ Complete in Rust

---

## Component Overview

Configuration Management handles command-line argument parsing, configuration validation, and default value management for all timpani-n runtime parameters.

---

## AS-IS: C Implementation

**File:** `timpani-n/src/config.c`

### CLI Arguments

```c
static struct option long_options[] = {
    {"help",    no_argument,       0, 'h'},
    {"cpu",     required_argument, 0, 'c'},
    {"prio",    required_argument, 0, 'p'},
    {"port",    required_argument, 0, 'P'},
    {"address", required_argument, 0, 'a'},
    {"node-id", required_argument, 0, 'n'},
    {"log",     required_argument, 0, 'l'},
    {"retry",   required_argument, 0, 'r'},
    {"enable-apex", no_argument,   0, 'e'},
    {0, 0, 0, 0}
};
```

### Configuration Structure

```c
struct config {
    int cpu;                  // CPU affinity (-1 = no affinity)
    int prio;                 // RT priority (1-99, -1 = default)
    int port;                 // Server port (default: 7777)
    char address[256];        // Server address
    char node_id[256];        // Node identifier
    int log_level;            // Log verbosity (0-5)
    int max_retries;          // Connection retry limit
    bool enable_apex;         // Apex.OS integration mode
};
```

### Defaults

```c
#define TT_DEFAULT_CPU_AFFINITY -1
#define TT_DEFAULT_PRIORITY -1
#define TT_DEFAULT_PORT 7777
#define TT_DEFAULT_ADDRESS "127.0.0.1"
#define TT_DEFAULT_NODE_ID "1"
#define TT_DEFAULT_LOG_LEVEL 3  // INFO
#define TT_MAX_CONNECTION_RETRIES 300
```

---

## WILL-BE: Rust Implementation (✅ Complete)

**File:** `timpani_rust/timpani-n/src/config/mod.rs`

### Configuration Structure

```rust
#[derive(Debug, Clone, Parser)]
#[command(
    name = "timpani-n",
    about = "timpani-n Node Executor - Time-Triggered Real-Time Task Scheduler",
    version
)]
pub struct Config {
    /// CPU affinity (-1 for no affinity, 0-1023 for specific CPU)
    #[arg(short, long, default_value_t = defaults::CPU_NO_AFFINITY,
          value_parser = clap::value_parser!(i32).range(validation::CPU_MIN..=validation::CPU_MAX))]
    pub cpu: i32,

    /// Real-time priority (1-99 for SCHED_FIFO, -1 for default)
    #[arg(short, long, default_value_t = defaults::PRIORITY_DEFAULT,
          value_parser = clap::value_parser!(i32).range(validation::PRIORITY_MIN..=validation::PRIORITY_MAX))]
    pub priority: i32,

    /// Server port number
    #[arg(short = 'P', long, default_value_t = defaults::PORT,
          value_parser = clap::value_parser!(u16).range(validation::PORT_MIN..=validation::PORT_MAX))]
    pub port: u16,

    /// Server address
    #[arg(short, long, default_value = defaults::ADDRESS)]
    pub address: String,

    /// Node identifier
    #[arg(short, long, default_value = defaults::NODE_ID)]
    pub node_id: String,

    /// Log level (0=Silent, 1=Error, 2=Warn, 3=Info, 4=Debug, 5=Verbose)
    #[arg(short, long, default_value_t = defaults::LOG_LEVEL,
          value_parser = clap::value_parser!(u8).range(0..=5))]
    pub log_level: u8,

    /// Maximum connection retry attempts
    #[arg(short, long, default_value_t = defaults::MAX_RETRIES)]
    pub max_retries: u32,

    /// Enable Apex.OS integration mode
    #[arg(short, long, default_value_t = false)]
    pub enable_apex: bool,
}
```

### Parsing

```rust
impl Config {
    pub fn from_args() -> TimpaniResult<Self> {
        let config = Config::parse();
        config.validate()?;
        Ok(config)
    }

    pub fn validate(&self) -> TimpaniResult<()> {
        // CPU validation
        if self.cpu < -1 || self.cpu > 1023 {
            return Err(TimpaniError::InvalidCpuAffinity(self.cpu));
        }

        // Priority validation
        if self.priority != -1 && (self.priority < 1 || self.priority > 99) {
            return Err(TimpaniError::InvalidPriority(self.priority));
        }

        // Port validation
        if self.port == 0 {
            return Err(TimpaniError::InvalidPort(self.port));
        }

        Ok(())
    }
}
```

---

## AS-IS vs WILL-BE Comparison

| Aspect | C (AS-IS) | Rust (WILL-BE) |
|--------|-----------|----------------|
| **Parsing** | `getopt_long()` | `clap::Parser` derive macro ✅ |
| **Validation** | Manual checks | Clap validators + custom `validate()` ✅ |
| **Defaults** | #define constants | `defaults::*` module ✅ |
| **Help Text** | Manual fprintf | Clap auto-generated ✅ |
| **Error Messages** | Custom format strings | Structured TimpaniError ✅ |
| **Type Safety** | `int` for everything | Typed (i32, u16, u8, bool) ✅ |

---

## Migration Notes

### What Changed
1. ✅ **Clap Derive** instead of getopt: Auto-generated parsing
2. ✅ **Range Validators**: Compile-time + runtime validation
3. ✅ **Structured Types**: u16 for port, u8 for log level
4. ✅ **Auto Help**: `--help` generated automatically

### What Stayed the Same
1. Same CLI argument names (`-c`, `-p`, `-P`, etc.)
2. Same default values (port 7777, max_retries 300)
3. Same validation ranges (CPU 0-1023, priority 1-99)

---

**Document Version:** 1.0
**Status:** ✅ Complete
**Verified Against:** `timpani_rust/timpani-n/src/config/mod.rs`
