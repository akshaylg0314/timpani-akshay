<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: Communication Protocols Component

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-o-lld-09
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** Protocol Definitions & Wire Format
**Responsibility:** Define gRPC services, message formats, and protocol buffers for all communication
**Status:** ✅ Migrated (C++ → Rust, D-Bus → gRPC)

## Component Overview

Communication Protocols component defines all inter-process communication between:
1. **Pullpiri ↔ timpani-o** (gRPC): Workload submission and fault reporting
2. **timpani-o ↔ timpani-n** (C++: D-Bus | Rust: gRPC): Schedule distribution and synchronization

---

## As-Is: C++ Implementation

### Protocol Summary (C++)

| Connection | Protocol | Port | Serialization |
|------------|----------|------|---------------|
| Pullpiri → timpani-o (SchedInfo) | gRPC | 50052 | Protobuf |
| timpani-o → Pullpiri (Fault) | gRPC | 50053 | Protobuf |
| timpani-n ↔ timpani-o | **D-Bus over TCP** | **7777** | **Custom binary (libtrpc)** |

### D-Bus Protocol (C++ Only)

```cpp
// libtrpc custom serialization
struct trpc_msg {
    uint32_t msg_type;
    uint32_t payload_size;
    char payload[];
};

// Three RPC operations (C callbacks)
extern "C" {
    struct trpc_msg* GetSchedInfoCallback(const struct trpc_msg* req);
    struct trpc_msg* SyncCallback(const struct trpc_msg* req);
    void DMissCallback(const struct trpc_msg* req);
}
```

### gRPC Protocol (C++)

**File:** `proto/schedinfo.proto`

```protobuf
service SchedInfoService {
    rpc AddSchedInfo (SchedInfo) returns (Response);
}

service FaultService {
    rpc NotifyFault (FaultInfo) returns (Response);
}

message SchedInfo {
    string workload_id = 1;
    repeated TaskInfo tasks = 2;
}

message TaskInfo {
    string name = 1;
    int32 priority = 2;
    int32 policy = 3;
    uint64 cpu_affinity = 4;
    int32 period = 5;
    int32 release_time = 6;
    int32 runtime = 7;
    int32 deadline = 8;
    string node_id = 9;
    int32 max_dmiss = 10;
}
```

---

## Will-Be: Rust Implementation

### Protocol Summary (Rust)

| Connection | Protocol | Port | Serialization |
|------------|----------|------|---------------|
| Pullpiri → timpani-o (SchedInfo) | gRPC | 50052 | Protobuf |
| timpani-o → Pullpiri (Fault) | gRPC | 50053 | Protobuf |
| timpani-n ↔ timpani-o | **gRPC/HTTP2** | **50054** | **Protobuf** |

### **BREAKING CHANGE: D-Bus → gRPC**

**What Changed:**
- **Protocol:** D-Bus peer-to-peer → gRPC/HTTP2
- **Port:** 7777 → 50054
- **Serialization:** Custom binary (`serialize.c`) → Protocol Buffers
- **API:** C callbacks → Rust async trait methods

**Why:**
1. **Standard Protocol:** gRPC is industry-standard, better tooling
2. **Type Safety:** Protobuf schema enforced at compile time
3. **Debugging:** grpcurl, gRPC reflection, Wireshark dissectors
4. **No Custom Code:** libtrpc removed - Tonic auto-generates everything

---

## Service Definitions

### 1. SchedInfoService (Pullpiri → timpani-o)

**Proto Definition:**
```protobuf
service SchedInfoService {
    rpc AddSchedInfo (SchedInfo) returns (Response);
}

message SchedInfo {
    string workload_id = 1;
    repeated TaskInfo tasks = 2;
}

message TaskInfo {
    string name = 1;
    int32 priority = 2;
    int32 policy = 3;
    uint64 cpu_affinity = 4;
    int32 period = 5;
    int32 release_time = 6;
    int32 runtime = 7;
    int32 deadline = 8;
    string node_id = 9;
    int32 max_dmiss = 10;
}

message Response {
    int32 status = 1;
}
```

**Rust Implementation:**
```rust
#[tonic::async_trait]
impl SchedInfoService for SchedInfoServiceImpl {
    async fn add_sched_info(
        &self,
        request: Request<SchedInfo>,
    ) -> Result<Response<ProtoResponse>, Status> {
        let req = request.into_inner();
        // ... process scheduling
        Ok(Response::new(ProtoResponse { status: 0 }))
    }
}
```

---

### 2. FaultService (timpani-o → Pullpiri)

**Proto Definition:**
```protobuf
service FaultService {
    rpc NotifyFault (FaultInfo) returns (Response);
}

message FaultInfo {
    string workload_id = 1;
    string node_id = 2;
    string task_name = 3;
    FaultType fault_type = 4;
}

enum FaultType {
    UNKNOWN = 0;
    DMISS = 1;  // Deadline miss
}
```

**Rust Implementation:**
```rust
#[tonic::async_trait]
impl FaultNotifier for FaultClient {
    async fn notify_fault(&self, info: FaultNotification) -> Result<(), FaultError> {
        let request = FaultInfo {
            workload_id: info.workload_id,
            node_id: info.node_id,
            task_name: info.task_name,
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

---

### 3. NodeService (timpani-o ↔ timpani-n)

**Proto Definition:**
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

message ScheduledTask {
    string name = 1;
    int32 sched_priority = 2;
    int32 sched_policy = 3;
    int32 period_us = 4;
    int32 release_time_us = 5;
    int32 runtime_us = 6;
    int32 deadline_us = 7;
    uint64 cpu_affinity = 8;
    int32 max_dmiss = 9;
    string assigned_node = 10;
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

**Rust Implementation:**
```rust
#[tonic::async_trait]
impl NodeService for NodeServiceImpl {
    async fn get_sched_info(...) -> Result<Response<NodeSchedResponse>, Status> {
        let guard = self.workload_store.lock().await;
        let ws = guard.as_ref().ok_or_else(|| Status::not_found("no workload"))?;

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

    async fn sync_timer(...) -> Result<Response<SyncResponse>, Status> {
        // Barrier synchronization logic
        // ...
        Ok(Response::new(SyncResponse { ack: true, start_time_sec, start_time_nsec }))
    }

    async fn report_d_miss(...) -> Result<Response<NodeResponse>, Status> {
        // Forward to FaultService
        self.fault_notifier.notify_fault(fault_info).await?;
        Ok(Response::new(NodeResponse { status: 0, error_message: String::new() }))
    }
}
```

---

## Protocol Comparison

### C++ D-Bus vs Rust gRPC

| Aspect | C++ D-Bus (Legacy) | Rust gRPC (New) |
|--------|-------------------|-----------------|
| **Transport** | TCP sockets + custom framing | HTTP/2 |
| **Serialization** | `serialize.c` (manual) | Protocol Buffers (auto-generated) |
| **Port** | 7777 | 50054 |
| **API Style** | C callbacks (`extern "C"`) | Rust async trait methods |
| **Type Safety** | Runtime (manual casts) | Compile-time (Tonic + prost) |
| **Error Handling** | NULL return / error codes | `Result<Response<T>, Status>` |
| **Debugging Tools** | None (custom protocol) | grpcurl, gRPC reflection, Wireshark |
| **Client Code** | libtrpc (custom C library) | Tonic (official Rust framework) |
| **Wire Format** | Binary struct layout | Protobuf encoding |

---

## Design Decisions

### D-PROTO-001: Why Replace D-Bus with gRPC?

**Technical Reasons:**

1. **Standard Protocol:** gRPC is widely adopted, well-documented
2. **Tooling:** grpcurl for CLI testing, gRPC reflection for introspection
3. **Type Safety:** Tonic generates types from `.proto` at compile time
4. **Async Native:** Tonic built on Tokio async runtime (better scalability)
5. **Debugging:** Wireshark has gRPC dissectors (D-Bus was opaque binary)

**Migration Cost:**
- ❌ **Breaking:** timpani-n must migrate from libtrpc to gRPC client
- ✅ **Benefit:** Removes ~2000 lines of custom serialization code
- ✅ **Benefit:** libtrpc dependency eliminated

---

### D-PROTO-002: Port Allocation

| Service | C++ Port | Rust Port | Rationale |
|---------|----------|-----------|-----------|
| SchedInfoService | 50052 | 50052 | Unchanged (Pullpiri compatibility) |
| FaultService | 50053 | 50053 | Unchanged (Pullpiri compatibility) |
| DBusServer | 7777 | — | Removed |
| NodeService | — | 50054 | New gRPC service |

**Why 50054?**
- Sequential from 50052, 50053
- Configurable via `--nodeport` CLI argument
- No conflict with legacy port 7777

---

### D-PROTO-003: Message Encoding

**C++ D-Bus (Custom Binary):**
```cpp
void serialize_schedinfo_t(const schedinfo_t* info, uint8_t* buffer) {
    memcpy(buffer, &info->hyperperiod_us, sizeof(uint64_t));
    buffer += 8;
    memcpy(buffer, &info->task_count, sizeof(uint32_t));
    buffer += 4;
    // ... manual layout
}
```

**Rust gRPC (Protobuf):**
```protobuf
message NodeSchedResponse {
    string workload_id = 1;
    uint64 hyperperiod_us = 2;
    repeated ScheduledTask tasks = 3;
}
```

```rust
// Tonic auto-generates this code:
impl prost::Message for NodeSchedResponse {
    fn encode(&self, buf: &mut impl prost::bytes::BufMut) {
        // Protobuf encoding (auto-generated)
    }
    fn decode(buf: impl prost::bytes::Buf) -> Result<Self, prost::DecodeError> {
        // Protobuf decoding (auto-generated)
    }
}

// Usage is transparent:
let response = NodeSchedResponse {
    workload_id: "wl_001".to_string(),
    hyperperiod_us: 60_000,
    tasks: vec![...],
};
// Tonic handles serialization automatically
```

**Benefits:**
- **No Manual Code:** Protobuf compiler generates encoding/decoding
- **Schema Evolution:** Can add optional fields without breaking compatibility
- **Language Agnostic:** Same `.proto` file works for C++, Rust, Python, etc.

---

## Wire Format Examples

### AddSchedInfo Request

**Protobuf Text Format:**
```protobuf
workload_id: "wl_automotive_001"
tasks {
  name: "sensor_fusion"
  priority: 95
  policy: 1  # FIFO
  cpu_affinity: 12  # CPUs 2,3
  period: 10000  # µs
  runtime: 2000
  deadline: 10000
  node_id: "node01"
  max_dmiss: 3
}
tasks {
  name: "lidar_processing"
  priority: 90
  policy: 1
  cpu_affinity: 15  # CPUs 0,1,2,3
  period: 20000
  runtime: 5000
  deadline: 20000
  node_id: "node01"
  max_dmiss: 2
}
```

**Binary Wire (Hex Dump - example):**
```
0a 13 77 6c 5f 61 75 74 6f 6d 6f 74 69 76 65 5f  ..wl_automotive_
30 30 31 12 3e 0a 0d 73 65 6e 73 6f 72 5f 66 75  001.>..sensor_fu
73 69 6f 6e 10 5f 18 01 20 0c 28 90 4e 30 d0 0f  sion._.. .(.N0..
... (Protobuf binary encoding)
```

---

## gRPC Error Mapping

### Rust → gRPC Status Codes

```rust
match scheduler.schedule(tasks, algorithm) {
    Ok(map) => Ok(Response::new(ProtoResponse { status: 0 })),

    Err(SchedulerError::NoTasks) => {
        Err(Status::invalid_argument("no tasks provided"))
    }

    Err(SchedulerError::ConfigNotLoaded) => {
        Err(Status::failed_precondition("node config not loaded"))
    }

    Err(SchedulerError::UnknownAlgorithm(algo)) => {
        Err(Status::invalid_argument(format!("unknown algorithm: {}", algo)))
    }

    Err(SchedulerError::TaskRejected { task, reason }) => {
        Err(Status::resource_exhausted(format!(
            "task '{}' rejected: {}", task, reason
        )))
    }
}
```

| SchedulerError Variant | gRPC Status Code | HTTP/2 Equivalent |
|------------------------|------------------|-------------------|
| `NoTasks` | `INVALID_ARGUMENT` | 400 Bad Request |
| `ConfigNotLoaded` | `FAILED_PRECONDITION` | 400 Bad Request |
| `UnknownAlgorithm` | `INVALID_ARGUMENT` | 400 Bad Request |
| `TaskRejected` | `RESOURCE_EXHAUSTED` | 429 Too Many Requests |
| `AdmissionRejected` | `RESOURCE_EXHAUSTED` | 429 Too Many Requests |

---

## Testing Tools

### C++ D-Bus (Limited)

```bash
# No standard tools - must write custom client
./test_dbus_client --port 7777
```

### Rust gRPC (Rich Tooling)

**grpcurl (CLI testing):**
```bash
# List services
grpcurl -plaintext localhost:50052 list

# Call RPC
grpcurl -plaintext -d '{
  "workload_id": "wl_test",
  "tasks": [
    {"name": "task_0", "period": 10000, "runtime": 2000, ...}
  ]
}' localhost:50052 SchedInfoService/AddSchedInfo
```

**gRPC Reflection:**
```rust
// Enable in server
tonic::transport::Server::builder()
    .add_service(tonic_reflection::server::Builder::configure()
        .register_encoded_file_descriptor_set(FILE_DESCRIPTOR_SET)
        .build()?)
    .add_service(SchedInfoServiceServer::new(sched_info_service))
    .serve(addr)
    .await?;
```

**Wireshark:**
- Filter: `http2.streamid && protobuf`
- Dissects gRPC frames automatically

---

## Migration Notes

### Breaking Changes

**timpani-n Side:**
```cpp
// OLD (C++ libtrpc client)
#include "peer_dbus.h"
schedinfo_t* info = trpc_client_schedinfo(node_id);

// NEW (Rust gRPC client)
// timpani-n will need Tonic client or C++ gRPC client
auto channel = grpc::CreateChannel("localhost:50054", ...);
auto stub = NodeService::NewStub(channel);
NodeSchedRequest request;
request.set_node_id(node_id);
NodeSchedResponse response;
stub->GetSchedInfo(&context, request, &response);
```

**Must Migrate Together:**
- Rust timpani-o (NodeService server) deployed with gRPC support
- timpani-n updated to use gRPC client (libtrpc removed)
- Cannot mix old/new protocols

---

### What Stayed the Same

1. **Proto Messages:** SchedInfo, TaskInfo, FaultInfo unchanged
2. **Ports:** 50052 (SchedInfo), 50053 (Fault) unchanged
3. **Business Logic:** Same scheduling algorithms, barrier sync
4. **Pullpiri API:** No changes to Pullpiri's client code

---

**Document Version:** 1.0
**Last Updated:** May 12, 2026
**Status:** ✅ Complete
**Verified Against:** `timpani_rust/timpani-o/proto/schedinfo.proto` and `src/grpc/*.rs`
