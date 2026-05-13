<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: FaultService Client Component

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-o-lld-02
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** gRPC Client
**Responsibility:** Report fault events (deadline misses) to Pullpiri orchestrator
**Status:** ✅ Migrated (C++ → Rust)

## Component Overview

The FaultService Client component is responsible for forwarding fault notifications (primarily deadline misses) from timpani-n nodes back to the Pullpiri orchestrator. It maintains a persistent gRPC connection and handles failures gracefully.

---

## As-Is: C++ Implementation

### Class Structure

```cpp
class FaultServiceClient {
public:
    static FaultServiceClient& GetInstance();

    bool Initialize(const std::string& server_address);
    bool IsInitialized() const;
    bool NotifyFault(const std::string& workload_id,
                    const std::string& node_id,
                    const std::string& task_name,
                    FaultType fault_type);
private:
    // Singleton - private constructor
    FaultServiceClient() = default;
};
```

### Responsibilities (C++)

1. Maintain persistent gRPC connection to Piccolo
2. Send fault notifications for deadline misses
3. Handle connection failures and retries
4. Aggregate fault information from multiple sources

### Key Features (C++)

- **Singleton Pattern:** Single instance per process (`GetInstance()`)
- **Connection Management:** Automatic reconnection on failures
- **Fault Types:** Support for various fault types (DMISS, etc.)
- **Asynchronous Operation:** Non-blocking fault reporting

### Configuration (C++)

- **Target:** Piccolo FaultService (default: localhost:50053)
- **Protocol:** gRPC over HTTP/2
- **Retry Policy:** Exponential backoff with maximum attempts

### Design Limitation (C++)

The singleton pattern exists **only** to work around C-style static callbacks in `DBusServer::DMissCallback`, which cannot capture `this` pointer.

---

## Will-Be: Rust Implementation

### Module Structure

```rust
// File: timpani_rust/timpani-o/src/fault/mod.rs

/// Production gRPC client for Pullpiri's `FaultService`.
pub struct FaultClient {
    stub: ProtoFaultClient<Channel>,
}

/// Async interface for sending fault notifications.
#[tonic::async_trait]
pub trait FaultNotifier: Send + Sync {
    async fn notify_fault(&self, info: FaultNotification) -> Result<(), FaultError>;
}
```

### Responsibilities (Rust)

1. **Connect** lazily to Pullpiri FaultService
2. **Send** fault notifications asynchronously
3. **Handle** RPC errors with structured error types
4. **Support** dependency injection via trait abstraction

### Implementation (Rust)

```rust
impl FaultClient {
    /// Create a fault client that connects lazily to `addr`.
    ///
    /// The TCP connection is not established until the first RPC call.
    pub fn connect_lazy(addr: String) -> anyhow::Result<Arc<dyn FaultNotifier>> {
        let channel = tonic::transport::Endpoint::from_shared(addr)?
            .connect_lazy();
        let stub = ProtoFaultClient::new(channel);
        Ok(Arc::new(Self { stub }))
    }
}

#[tonic::async_trait]
impl FaultNotifier for FaultClient {
    async fn notify_fault(&self, info: FaultNotification) -> Result<(), FaultError> {
        let request = FaultInfo {
            workload_id: info.workload_id.clone(),
            node_id: info.node_id.clone(),
            task_name: info.task_name.clone(),
            fault_type: info.fault_type as i32,
        };

        let mut stub = self.stub.clone();
        let response = stub.notify_fault(request).await?;

        if response.into_inner().status != 0 {
            return Err(FaultError::RemoteError(response.status));
        }

        Ok(())
    }
}
```

### Key Features (Rust)

- **Lazy Connection:** TCP connection established on first RPC call
- **Trait Abstraction:** `FaultNotifier` trait enables testing with mocks
- **No Singleton:** Injected as `Arc<dyn FaultNotifier>`
- **Structured Errors:** `FaultError` enum with specific error variants
- **Clone-able Stub:** Tonic clients are cheap to clone (shared channel)

---

## As-Is vs Will-Be Comparison

| Aspect | C++ (As-Is) | Rust (Will-Be) |
|--------|-------------|----------------|
| **Lifetime Management** | Singleton (global state) | Arc<dyn FaultNotifier> (injected) |
| **Connection Strategy** | Eager (on Initialize) | Lazy (on first RPC) |
| **Error Handling** | bool return + logging | Result<(), FaultError> with typed errors |
| **Testing** | Hard to mock singleton | Easy - inject MockFaultNotifier |
| **Thread Safety** | Mutex-protected singleton | Arc + Send + Sync trait bounds |
| **Async Support** | Synchronous (blocking) | Fully async with Tokio |
| **Dependency Injection** | Global instance | Constructor injection via Arc |
| **Reason for Singleton** | C-style callbacks limitation | No callbacks - async closures |

---

## Design Decisions

### D-FAULT-001: No Singleton Pattern

**C++ Limitation:**
```cpp
// DBusServer has static C-style callback
static void DMissCallback(const char* name, const char* task) {
    // Cannot capture 'this' → must use singleton
    FaultServiceClient::GetInstance().NotifyFault(...);
}
```

**Rust Solution:**
```rust
// Async closure can capture state
let fault_notifier = Arc::clone(&self.fault_notifier);
tokio::spawn(async move {
    fault_notifier.notify_fault(info).await.ok();
});
```

**Rationale:** Rust async closures can capture `Arc<dyn FaultNotifier>` directly, eliminating the need for global singletons. This improves testability and reduces coupling.

---

### D-FAULT-002: Lazy vs Eager Connection

**C++ Approach:**
```cpp
bool Initialize(const std::string& server_address) {
    // Connect immediately - fails if Pullpiri not running
    channel_ = grpc::CreateChannel(server_address, ...);
    if (!channel_->WaitForConnected(...)) {
        return false; // timpani-o won't start
    }
    return true;
}
```

**Rust Approach:**
```rust
pub fn connect_lazy(addr: String) -> anyhow::Result<Arc<dyn FaultNotifier>> {
    // Connection established on first RPC call
    let channel = Endpoint::from_shared(addr)?.connect_lazy();
    // timpani-o can start even if Pullpiri is down
    Ok(Arc::new(FaultClient { stub: ProtoFaultClient::new(channel) }))
}
```

**Rationale:** Lazy connection avoids hard startup ordering dependency. timpani-o can start before Pullpiri is running. The first fault notification will trigger connection establishment.

---

### D-FAULT-003: Trait-Based Abstraction

**Interface:**
```rust
#[tonic::async_trait]
pub trait FaultNotifier: Send + Sync {
    async fn notify_fault(&self, info: FaultNotification) -> Result<(), FaultError>;
}
```

**Benefits:**
1. **Testing:** Inject `MockFaultNotifier` in unit tests
2. **Flexibility:** Can swap implementations without changing consumers
3. **Decoupling:** Consumers depend on trait, not concrete type

**Example Mock:**
```rust
#[cfg(test)]
mod test_support {
    pub struct MockFaultNotifier {
        calls: Arc<Mutex<Vec<FaultNotification>>>,
    }

    #[tonic::async_trait]
    impl FaultNotifier for MockFaultNotifier {
        async fn notify_fault(&self, info: FaultNotification) -> Result<(), FaultError> {
            self.calls.lock().unwrap().push(info);
            Ok(())
        }
    }
}
```

---

## Error Handling

### C++ Error Handling

```cpp
bool NotifyFault(...) {
    try {
        auto response = stub_->NotifyFault(context, request);
        if (!response.ok()) {
            LOG_ERROR("RPC failed: " << response.error_message());
            return false;
        }
        if (response.value().status() != 0) {
            LOG_ERROR("Pullpiri rejected fault");
            return false;
        }
        return true;
    } catch (const std::exception& e) {
        LOG_ERROR("Exception: " << e.what());
        return false;
    }
}
```

### Rust Error Handling

```rust
#[derive(Debug, Error)]
pub enum FaultError {
    #[error("transport error: {0}")]
    Transport(#[from] tonic::transport::Error),

    #[error("RPC status: {0}")]
    Rpc(#[from] tonic::Status),

    #[error("Pullpiri returned non-zero status {0}")]
    RemoteError(i32),
}

async fn notify_fault(&self, info: FaultNotification) -> Result<(), FaultError> {
    let mut stub = self.stub.clone();
    let response = stub.notify_fault(request).await?; // ? propagates errors

    if response.into_inner().status != 0 {
        return Err(FaultError::RemoteError(response.status));
    }

    Ok(())
}
```

**Improvements:**
- **Typed Errors:** Each error case has a distinct variant
- **No Exceptions:** All errors are Result<> - no unwinding
- **Error Context:** `#[from]` provides automatic conversion
- **Propagation:** `?` operator for clean error propagation

---

## Data Structures

### FaultNotification

```rust
#[derive(Debug, Clone)]
pub struct FaultNotification {
    pub workload_id: String,
    pub node_id: String,
    pub task_name: String,
    pub fault_type: FaultType,
}
```

### FaultType (from Proto)

```protobuf
enum FaultType {
    UNKNOWN = 0;
    DMISS = 1;  // Deadline miss
}
```

---

## Usage Example

### C++ Usage

```cpp
// Singleton initialization at startup
FaultServiceClient::GetInstance().Initialize("localhost:50053");

// Later, in DBusServer callback:
FaultServiceClient::GetInstance().NotifyFault(
    workload_id, node_id, task_name, FaultType::DMISS
);
```

### Rust Usage

```rust
// At startup - inject into services
let fault_notifier = FaultClient::connect_lazy(
    "http://localhost:50053".to_string()
)?;

// In NodeService::report_dmiss
let info = FaultNotification {
    workload_id,
    node_id,
    task_name,
    fault_type: FaultType::Dmiss,
};

self.fault_notifier.notify_fault(info).await?;
```

---

## Testing

### C++ Testing Challenges

- Singleton makes unit testing difficult
- Requires mock server or actual Pullpiri instance
- Cannot inject test doubles

### Rust Testing Advantages

```rust
#[tokio::test]
async fn test_fault_notification() {
    let mock = Arc::new(MockFaultNotifier::new());

    // Inject mock into service
    let service = NodeServiceImpl::new(store, mock.clone(), timeout);

    // Trigger fault
    service.report_dmiss(request).await.unwrap();

    // Verify mock received call
    assert_eq!(mock.calls().len(), 1);
    assert_eq!(mock.calls()[0].task_name, "task_0");
}
```

---

## Migration Notes

### Breaking Changes

**None** - gRPC API contract remains identical:
```protobuf
service FaultService {
    rpc NotifyFault (FaultInfo) returns (Response);
}
```

### Implementation Changes

1. **Singleton removed** → Arc<dyn FaultNotifier> injection
2. **Eager connection** → Lazy connection
3. **Blocking RPC** → Async RPC
4. **bool return** → Result<(), FaultError>
5. **Global state** → Dependency injection

### Benefits

- ✅ Unit testable without mock server
- ✅ No hard startup ordering dependency
- ✅ Better error reporting (typed errors)
- ✅ No global mutable state
- ✅ Async-friendly (no blocking)

---

**Document Version:** 1.0
**Last Updated:** May 12, 2026
**Status:** ✅ Complete
**Verified Against:** `timpani_rust/timpani-o/src/fault/mod.rs` (actual implementation)
