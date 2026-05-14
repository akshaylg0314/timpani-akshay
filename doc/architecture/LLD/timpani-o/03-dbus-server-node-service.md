<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: D-Bus Server / Node Service Component

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-o-lld-03
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** Communication Server
**Responsibility:** Serve scheduling information and coordinate synchronization with timpani-n nodes
**Status:** ✅ Migrated (C++ D-Bus → Rust gRPC)

## Component Overview

This component provides the communication interface between timpani-o (global orchestrator) and timpani-n nodes (local schedulers). It handles three primary operations: serving schedules, coordinating synchronized starts, and receiving deadline miss reports.

---

## As-Is: C++ Implementation (D-Bus Server)

### Class Structure

```cpp
class DBusServer {
public:
    explicit DBusServer(std::shared_ptr<SchedInfoServiceImpl> sched_info_service,
                       std::shared_ptr<NodeConfigManager> node_config_manager);

    bool Initialize(int port = 7777);
    void Run();
    void Stop();

    // Static callbacks for libtrpc
    static struct trpc_msg* GetSchedInfoCallback(const struct trpc_msg* req);
    static struct trpc_msg* SyncCallback(const struct trpc_msg* req);
    static void DMissCallback(const struct trpc_msg* req);
};
```

### Responsibilities (C++)

1. **Listen** for incoming connections on TCP port 7777
2. **Serve** scheduling information to timpani-n nodes (via `trpc_client_schedinfo`)
3. **Coordinate** synchronization barrier for all nodes (via `trpc_client_sync`)
4. **Receive** deadline miss reports (via `trpc_client_dmiss`)
5. **Serialize** messages using custom binary format (libtrpc)

### Key Features (C++)

- **Protocol:** D-Bus peer-to-peer over TCP (custom libtrpc implementation)
- **Port:** 7777 (default)
- **Serialization:** Custom binary serialization (`serialize.c`)
- **Callbacks:** C-style static callbacks due to libtrpc C API

### Data Flow (C++)

```
timpani-n (libtrpc client)
  ↓ TCP connection to port 7777
DBusServer::GetSchedInfoCallback()
  → sched_info_service_->GetSchedInfoMap()
  → Serialize schedinfo_t struct
  ↓
Return binary message to timpani-n
```

### Configuration (C++)

```cpp
class DBusServerConfig {
    int port = 7777;
    std::string bind_address = "0.0.0.0";
    int max_connections = 10;
};
```

---

## Will-Be: Rust Implementation (NodeService)

### Module Structure

```rust
// File: timpani_rust/timpani-o/src/grpc/node_service.rs

#[derive(Clone)]
pub struct NodeServiceImpl {
    workload_store: WorkloadStore,
    fault_notifier: Arc<dyn FaultNotifier>,
    sync_timeout: Duration,
}
```

### Responsibilities (Rust)

1. **GetSchedInfo:** timpani-n pulls its task list via gRPC
2. **SyncTimer:** Blocking barrier - all nodes synchronize start time
3. **ReportDMiss:** Deadline miss forwarded to Pullpiri
4. **Barrier Management:** Watch channel coordination for sync barrier

### Protocol Change: D-Bus → gRPC

| Operation | C++ (D-Bus/libtrpc) | Rust (gRPC) |
|-----------|---------------------|-------------|
| **Transport** | TCP with custom binary protocol | HTTP/2 with Protobuf |
| **Port** | 7777 | 50054 (configurable via `--nodeport`) |
| **API Contract** | C function pointers | Protobuf service definition |
| **Serialization** | `serialize.c` custom format | Protocol Buffers (auto-generated) |
| **Error Handling** | Return NULL or error codes | `Result<Response<T>, Status>` |

### Implementation (Rust)

```rust
#[tonic::async_trait]
impl NodeService for NodeServiceImpl {
    // ── GetSchedInfo ──────────────────────────────────────────────────────────
    async fn get_sched_info(
        &self,
        request: Request<NodeSchedRequest>,
    ) -> Result<Response<NodeSchedResponse>, Status> {
        let node_id = request.into_inner().node_id;

        let guard = self.workload_store.lock().await;
        let ws = guard.as_ref()
            .ok_or_else(|| Status::not_found("no workload has been scheduled yet"))?;

        // Return this node's task list (or empty vec if node has no tasks)
        let tasks: Vec<ScheduledTask> = ws.schedule
            .get(&node_id)
            .map(|v| v.iter().map(to_proto_task).collect())
            .unwrap_or_default();

        Ok(Response::new(NodeSchedResponse {
            workload_id: ws.workload_id.clone(),
            hyperperiod_us: ws.hyperperiod.hyperperiod_us,
            tasks,
        }))
    }

    // ── SyncTimer ─────────────────────────────────────────────────────────────
    async fn sync_timer(
        &self,
        request: Request<SyncRequest>,
    ) -> Result<Response<SyncResponse>, Status> {
        let node_id = request.into_inner().node_id;

        // Phase 1: Register node and subscribe to barrier (under lock)
        let mut barrier_rx = {
            let mut guard = self.workload_store.lock().await;
            let ws = guard.as_mut()
                .ok_or_else(|| Status::not_found("no workload"))?;

            // Subscribe before firing so we can't miss Released
            let rx = ws.barrier_tx.subscribe();
            ws.synced_nodes.insert(node_id.clone());

            // If this completes the set, fire the barrier
            if ws.active_nodes.iter().all(|n| ws.synced_nodes.contains(n)) {
                let (sec, nsec) = compute_start_time();
                let _ = ws.barrier_tx.send(BarrierStatus::Released {
                    start_time_sec: sec,
                    start_time_nsec: nsec,
                });
            }
            rx
        }; // Lock released

        // Phase 2: Wait for barrier or timeout (async, no lock)
        loop {
            match *barrier_rx.borrow_and_update() {
                BarrierStatus::Released { start_time_sec, start_time_nsec } => {
                    return Ok(Response::new(SyncResponse {
                        ack: true, start_time_sec, start_time_nsec
                    }));
                }
                BarrierStatus::Cancelled => {
                    return Err(Status::aborted("workload replaced"));
                }
                BarrierStatus::TimedOut => {
                    return Err(Status::deadline_exceeded("barrier timeout"));
                }
                BarrierStatus::Waiting => {}
            }

            tokio::select! {
                result = barrier_rx.changed() => { result?; }
                _ = &mut timeout_sleep => {
                    // Broadcast timeout to all waiters
                    {
                        let guard = self.workload_store.lock().await;
                        if let Some(ws) = guard.as_ref() {
                            let _ = ws.barrier_tx.send(BarrierStatus::TimedOut);
                        }
                    }
                    return Err(Status::deadline_exceeded("barrier timeout"));
                }
            }
        }
    }

    // ── ReportDMiss ───────────────────────────────────────────────────────────
    async fn report_d_miss(
        &self,
        request: Request<DeadlineMissInfo>,
    ) -> Result<Response<NodeResponse>, Status> {
        let info = request.into_inner();
        let node_id = info.node_id.clone();
        let task_name = info.task_name.clone();

        // Resolve workload_id from active schedule
        let workload_id = {
            let guard = self.workload_store.lock().await;
            guard.as_ref()
                .ok_or_else(|| Status::failed_precondition("no workload"))?
                .workload_id.clone()
        };

        // Forward to Pullpiri FaultService
        let fault_info = FaultNotification {
            workload_id,
            node_id,
            task_name,
            fault_type: FaultType::Dmiss,
        };

        self.fault_notifier.notify_fault(fault_info).await
            .map_err(|e| Status::internal(format!("fault notify failed: {}", e)))?;

        Ok(Response::new(NodeResponse { status: 0, error_message: String::new() }))
    }
}
```

---

## As-Is vs Will-Be Comparison

| Aspect | C++ (D-Bus) | Rust (gRPC) |
|--------|-------------|-------------|
| **Protocol** | D-Bus peer-to-peer over TCP | gRPC/HTTP2 |
| **Port** | 7777 | 50054 |
| **Serialization** | Custom binary (`serialize.c`) | Protocol Buffers (auto-generated) |
| **Message Format** | `schedinfo_t` C struct | `NodeSchedResponse` protobuf message |
| **API Style** | C callbacks with `struct trpc_msg*` | Rust async trait methods |
| **Concurrency** | Blocking I/O | Async/await with Tokio |
| **Error Handling** | NULL return or error codes | `Result<Response<T>, Status>` |
| **Barrier Sync** | Manual condition variable | Tokio watch channel |
| **Type Safety** | Manual serialization, type casts | Compile-time type checking via Tonic |
| **Dependencies** | `libtrpc` (custom C library) | `tonic` (official gRPC framework) |

---

## Design Decisions

### D-DBUS-001: Why Replace D-Bus with gRPC?

**C++ Limitations:**
- **Custom Protocol:** `libtrpc` is project-specific binary protocol
- **Limited Tooling:** No standard debugging tools (Wireshark, grpcurl)
- **Manual Serialization:** Hand-written `serialize.c` code
- **C API Constraints:** Static callbacks, no type safety

**Rust Benefits:**
- **Standard Protocol:** gRPC is industry-standard
- **Auto-Generated Code:** Tonic generates client/server from `.proto`
- **Better Debugging:** grpcurl, gRPC reflection, Wireshark dissectors
- **Type Safety:** Protobuf types checked at compile time

**Rationale:** gRPC provides better interoperability, tooling, and safety with no performance loss.

---

### D-DBUS-002: Barrier Synchronization Design

**C++ Approach:**
```cpp
// SyncCallback blocks all nodes until all check in
static struct trpc_msg* SyncCallback(const struct trpc_msg* req) {
    std::unique_lock<std::mutex> lock(barrier_mutex_);
    synced_nodes_.insert(node_id);

    if (synced_nodes_.size() == active_nodes_.size()) {
        barrier_cv_.notify_all(); // Wake all waiting nodes
    } else {
        barrier_cv_.wait(lock); // Block this thread
    }

    return CreateSyncResponse();
}
```

**Rust Approach:**
```rust
// SyncTimer uses Tokio watch channel for coordination
// Phase 1: Register (under lock)
let mut barrier_rx = {
    let mut guard = self.workload_store.lock().await;
    let ws = guard.as_mut()?;

    let rx = ws.barrier_tx.subscribe(); // Subscribe BEFORE firing
    ws.synced_nodes.insert(node_id);

    if all_nodes_ready() {
        ws.barrier_tx.send(Released { start_time... });
    }
    rx
}; // Lock released here

// Phase 2: Wait (NO lock held - async)
loop {
    match *barrier_rx.borrow_and_update() {
        Released { start_time } => return Ok(...),
        Cancelled => return Err(Status::aborted(...)),
        TimedOut => return Err(Status::deadline_exceeded(...)),
        Waiting => {}
    }

    tokio::select! {
        _ = barrier_rx.changed() => {},
        _ = timeout_sleep => { broadcast_timeout(); ... }
    }
}
```

**Key Differences:**
- **Lock Duration:** C++ holds mutex during wait; Rust releases before async wait
- **Broadcast:** C++ uses condition variable; Rust uses watch channel
- **Timeout:** C++ per-thread timer; Rust first-to-timeout broadcasts to all
- **Cancellation:** Rust supports workload cancellation (new feature)

**Benefits:**
- ✅ No lock contention during wait (Rust releases lock before async wait)
- ✅ Handles workload replacement during sync (Cancelled state)
- ✅ Configurable timeout (via `--sync-timeout-secs`)
- ✅ All handlers wake simultaneously (watch channel broadcast)

---

### D-DBUS-003: C Callbacks vs Rust Async Traits

**C++ Constraint:**
```cpp
// libtrpc requires static C-linkage callbacks
extern "C" struct trpc_msg* GetSchedInfoCallback(const struct trpc_msg* req) {
    // Cannot capture 'this' - must use global/singleton
    auto* instance = DBusServer::GetInstance();
    return instance->HandleGetSchedInfo(req);
}
```

**Rust Solution:**
```rust
// Tonic generates async trait implementation
#[tonic::async_trait]
impl NodeService for NodeServiceImpl {
    async fn get_sched_info(&self, request: Request<...>) -> Result<...> {
        // 'self' is available, no global state needed
        let guard = self.workload_store.lock().await;
        // ...
    }
}
```

**Rationale:** Rust's trait system eliminates need for C callbacks and global state. Dependency injection (`self.workload_store`) provides testability.

---

## Proto Message Definitions

### Service Definition

```protobuf
service NodeService {
    rpc GetSchedInfo (NodeSchedRequest) returns (NodeSchedResponse);
    rpc SyncTimer (SyncRequest) returns (SyncResponse);
    rpc ReportDMiss (DeadlineMissInfo) returns (NodeResponse);
}

message NodeSchedRequest {
    string node_id = 1;
}

message NodeSchedResponse {
    string workload_id = 1;
    uint64 hyperperiod_us = 2;
    repeated ScheduledTask tasks = 3;
}

message SyncRequest {
    string node_id = 1;
}

message SyncResponse {
    bool ack = 1;
    int64 start_time_sec = 2;
    int32 start_time_nsec = 3;
}

message DeadlineMissInfo {
    string workload_id = 1;
    string node_id = 2;
    string task_name = 3;
}

message NodeResponse {
    int32 status = 1;
    string error_message = 2;
}
```

---

## Barrier State Machine

### Rust BarrierStatus Enum

```rust
#[derive(Debug, Clone)]
pub enum BarrierStatus {
    Waiting,
    Released { start_time_sec: i64, start_time_nsec: i32 },
    Cancelled,
    TimedOut,
}
```

### State Transitions

```
Initial: Waiting
  ↓
  ├─→ All nodes check in → Released (success)
  ├─→ New workload arrives → Cancelled (abort)
  └─→ Timeout expires → TimedOut (failure)
```

### Timeout Handling

```rust
// First handler to timeout broadcasts to all others
tokio::select! {
    _ = barrier_rx.changed() => { /* Another handler fired */ }
    _ = &mut timeout_sleep => {
        // This handler timed out first - wake all others
        let guard = self.workload_store.lock().await;
        if let Some(ws) = guard.as_ref() {
            let _ = ws.barrier_tx.send(BarrierStatus::TimedOut);
        }
        return Err(Status::deadline_exceeded("barrier timeout"));
    }
}
```

**Default Timeout:** 30 seconds (configurable via `--sync-timeout-secs`)

---

## Migration Notes

### Breaking Changes

1. **Protocol Change:** D-Bus → gRPC (timpani-n must use gRPC client)
2. **Port Change:** 7777 → 50054
3. **Message Format:** Binary struct → Protobuf

### Backwards Compatibility

**None** - this is a breaking change. Requires:
- timpani-n migration to gRPC client (Milestone 2)
- Both components must be upgraded together

### Migration Path

1. Implement Rust timpani-o with gRPC NodeService
2. Migrate timpani-n from libtrpc to Tonic gRPC client
3. Deploy both simultaneously
4. Decommission D-Bus server and libtrpc

---

## Testing

### C++ Testing Challenges

- Requires running D-Bus server and libtrpc client
- Hard to mock C callbacks
- Manual message serialization testing

### Rust Testing Advantages

```rust
#[tokio::test]
async fn test_get_sched_info_success() {
    let store = new_workload_store();
    let notifier = Arc::new(MockFaultNotifier::new());
    let service = NodeServiceImpl::new(store.clone(), notifier, Duration::from_secs(30));

    // Populate workload
    {
        let mut guard = store.lock().await;
        *guard = Some(WorkloadState { ... });
    }

    // Call gRPC method
    let request = Request::new(NodeSchedRequest {
        node_id: "node01".to_string(),
    });

    let response = service.get_sched_info(request).await.unwrap();
    assert_eq!(response.into_inner().tasks.len(), 3);
}

#[tokio::test]
async fn test_sync_timer_barrier() {
    // Spawn two concurrent SyncTimer calls
    let (resp1, resp2) = tokio::join!(
        service.sync_timer(node1_req),
        service.sync_timer(node2_req),
    );

    // Both should succeed with same start time
    assert_eq!(resp1.start_time_sec, resp2.start_time_sec);
}
```

**Benefits:**
- No external server required
- Concurrent barrier tests using `tokio::join!`
- Mock fault notifier for isolation

---

**Document Version:** 1.0
**Last Updated:** May 12, 2026
**Status:** ✅ Complete
**Verified Against:** `timpani_rust/timpani-o/src/grpc/node_service.rs` (actual implementation)
