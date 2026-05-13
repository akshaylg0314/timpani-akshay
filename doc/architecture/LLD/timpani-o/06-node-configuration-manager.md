<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# LLD: Node Configuration Manager Component

**Document Information:**
- **Issuing Author:** Eclipse timpani Team
- **Configuration ID:** timpani-o-lld-06
- **Document Status:** Draft
- **Last Updated:** 2026-05-13

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0b | 2026-05-13 | Updated documentation metadata and standards compliance | LGSI-KarumuriHari | - |
| 0.0a | 2026-02-24 | Initial LLD document creation | Eclipse timpani Team | - |

---

**Component Type:** Configuration Loader
**Responsibility:** Load and manage node hardware specifications from YAML configuration files
**Status:** ✅ Migrated (C++ → Rust)

## Component Overview

The Node Configuration Manager loads node hardware specifications (CPU counts, memory limits, architecture details) from YAML files and provides read-only access to this information for the scheduler and other components.

---

## As-Is: C++ Implementation

### Class Structure

```cpp
class NodeConfigManager {
public:
    NodeConfigManager();

    bool LoadFromFile(const std::string& file_path);
    const NodeConfig* GetNodeConfig(const std::string& node_id) const;
    const NodeConfig* GetDefaultNodeConfig() const;
    const std::map<std::string, NodeConfig>& GetAllNodes() const;
    bool IsLoaded() const;

private:
    std::map<std::string, NodeConfig> nodes_;
    bool loaded_;
};

struct NodeConfig {
    std::string name;
    std::vector<int> available_cpus;
    uint64_t max_memory_mb;
    std::string architecture;
    std::string location;
    std::string description;
};
```

### Responsibilities (C++)

1. **Parse** YAML configuration files
2. **Store** node hardware specifications
3. **Provide** read-only access to node configurations
4. **Validate** configuration structure
5. **Fallback** to default configuration if file is empty

### YAML Format (C++)

```yaml
nodes:
  node01:
    available_cpus: [2, 3]
    max_memory_mb: 4096
    architecture: "aarch64"
    location: "front_sensor_unit"
    description: "Perception and sensor fusion node"
```

---

## Will-Be: Rust Implementation

### Module Structure

```rust
// File: timpani_rust/timpani-o/src/config/mod.rs

#[derive(Debug, Default)]
pub struct NodeConfigManager {
    nodes: HashMap<String, NodeConfig>,
    loaded: bool,
}

#[derive(Debug, Clone)]
pub struct NodeConfig {
    pub name: String,
    pub available_cpus: Vec<u32>,
    pub max_memory_mb: u64,
    pub architecture: String,
    pub location: String,
    pub description: String,
}
```

### Responsibilities (Rust)

1. **Load** YAML using `serde_yaml` (type-safe deserialization)
2. **Validate** structure at parse time (compile-time schema)
3. **Provide** immutable access via `&NodeConfig` references
4. **Fallback** to default config if no nodes parsed
5. **Support** config reload (clears and re-parses)

### Implementation (Rust)

```rust
impl NodeConfigManager {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn load_from_file(&mut self, path: &Path) -> Result<()> {
        info!("Loading node configuration from: {}", path.display());

        // Reset state before (re-)loading
        self.nodes.clear();
        self.loaded = false;

        // Read file
        let content = std::fs::read_to_string(path)
            .with_context(|| format!("Cannot open configuration file: {}", path.display()))?;

        // Parse YAML with Serde
        let file: NodeConfigFile = serde_yaml::from_str(&content)
            .with_context(|| format!("Failed to parse YAML file: {}", path.display()))?;

        // Convert to NodeConfig
        for (name, entry) in file.nodes {
            let node = NodeConfig {
                name: name.clone(),
                available_cpus: entry.available_cpus,
                max_memory_mb: entry.max_memory_mb,
                architecture: entry.architecture.unwrap_or_default(),
                location: entry.location.unwrap_or_default(),
                description: entry.description.unwrap_or_default(),
            };
            self.nodes.insert(name, node);
        }

        // Fallback: if no nodes, insert default
        if self.nodes.is_empty() {
            warn!("No nodes found in configuration file, using default configuration");
            let default = NodeConfig::default_config("default_node");
            self.nodes.insert("default_node".to_string(), default);
        }

        self.loaded = true;
        Ok(())
    }

    pub fn get_node_config(&self, name: &str) -> Option<&NodeConfig> {
        self.nodes.get(name)
    }

    pub fn get_all_nodes(&self) -> &HashMap<String, NodeConfig> {
        &self.nodes
    }

    pub fn is_loaded(&self) -> bool {
        self.loaded
    }

    pub fn node_count(&self) -> usize {
        self.nodes.len()
    }
}
```

### Default Configuration (Rust)

```rust
impl NodeConfig {
    pub fn default_config(name: impl Into<String>) -> Self {
        Self {
            name: name.into(),
            available_cpus: vec![0, 1, 2, 3],
            max_memory_mb: 4096,
            architecture: String::from("aarch64"),
            location: String::from("default_location"),
            description: String::from("Default node configuration"),
        }
    }

    pub fn cpu_count(&self) -> usize {
        self.available_cpus.len()
    }
}
```

---

## As-Is vs Will-Be Comparison

| Aspect | C++ (As-Is) | Rust (Will-Be) |
|--------|-------------|----------------|
| **YAML Parser** | Custom/yaml-cpp (manual parsing) | `serde_yaml` (automatic deserialization) |
| **Error Handling** | `bool` return + logging | `Result<(), anyhow::Error>` with context |
| **Type Safety** | Runtime validation | Compile-time schema via Serde `Deserialize` |
| **CPU Type** | `std::vector<int>` | `Vec<u32>` (unsigned) |
| **Optional Fields** | Manual presence checks | `Option<String>` + `unwrap_or_default()` |
| **Memory Limit** | `uint64_t` | `u64` (with sentinel `u64::MAX` = unconstrained) |
| **Default Fallback** | `GetDefaultNodeConfig()` returns pointer | `NodeConfig::default_config()` returns value |
| **Reload Support** | Implicit | Explicit `clear()` before re-parse |
| **Access Pattern** | `const NodeConfig*` (pointer) | `Option<&NodeConfig>` (reference) |

---

## Design Decisions

### D-CFG-001: Serde Deserialization vs Manual Parsing

**C++ Approach:**
```cpp
bool LoadFromFile(const std::string& path) {
    YAML::Node root = YAML::LoadFile(path);
    YAML::Node nodes = root["nodes"];

    for (auto it = nodes.begin(); it != nodes.end(); ++it) {
        std::string name = it->first.as<std::string>();
        YAML::Node node = it->second;

        NodeConfig config;
        config.name = name;
        config.available_cpus = node["available_cpus"].as<std::vector<int>>();
        config.max_memory_mb = node["max_memory_mb"].as<uint64_t>(4096);
        // ... manual field extraction

        nodes_[name] = config;
    }
    return true;
}
```

**Rust Approach:**
```rust
// Define YAML structure with Serde
#[derive(Debug, Deserialize)]
struct NodeConfigFile {
    nodes: HashMap<String, NodeConfigEntry>,
}

#[derive(Debug, Deserialize)]
struct NodeConfigEntry {
    #[serde(default)]
    available_cpus: Vec<u32>,

    #[serde(default = "default_max_memory_mb")]
    max_memory_mb: u64,

    architecture: Option<String>,
    location: Option<String>,
    description: Option<String>,
}

fn default_max_memory_mb() -> u64 {
    u64::MAX // Unconstrained
}

// Deserialization is automatic
let file: NodeConfigFile = serde_yaml::from_str(&content)?;
```

**Benefits:**
- **Type Safety:** Serde validates types at parse time
- **Default Values:** `#[serde(default)]` attribute handles missing fields
- **Error Messages:** Serde provides detailed parse errors with line numbers
- **No Manual Extraction:** Automatic conversion from YAML to Rust struct

---

### D-CFG-002: Optional Fields Handling

**C++ Approach:**
```cpp
// All fields are required - crash if missing
config.architecture = node["architecture"].as<std::string>();
```

**Rust Approach:**
```rust
// YAML field type
architecture: Option<String>,

// Conversion to NodeConfig
architecture: entry.architecture.unwrap_or_default(),
// If missing in YAML → Some(None) → unwrap_or_default() → ""
```

**Validation Levels:**
1. **Required:** `available_cpus: Vec<u32>` - parse fails if missing
2. **Optional with default:** `max_memory_mb` - uses `default_max_memory_mb()` if missing
3. **Optional:** `architecture: Option<String>` - becomes `""` if missing

**Example YAML (minimal valid):**
```yaml
nodes:
  node01:
    available_cpus: [0, 1]
    # max_memory_mb → defaults to u64::MAX
    # architecture → defaults to ""
```

---

### D-CFG-003: Memory Limit Semantics

**C++ Implementation:**
```cpp
uint64_t max_memory_mb; // 0 = unconstrained?
```

**Rust Implementation:**
```rust
#[serde(default = "default_max_memory_mb")]
max_memory_mb: u64,

fn default_max_memory_mb() -> u64 {
    u64::MAX // Explicitly means "no constraint"
}
```

**Rationale:**
- `0` is ambiguous (zero memory allowed? or unconstrained?)
- `u64::MAX` is explicit sentinel value for "no limit"
- Scheduler checks: `if node.max_memory_mb == u64::MAX { /* skip memory check */ }`

**Future Extension:**
When proto adds `memory_mb` field for tasks (currently dormant), scheduler will:
```rust
if node.max_memory_mb != u64::MAX {
    let total_memory: u64 = tasks_on_node.iter().map(|t| t.memory_mb).sum();
    if total_memory > node.max_memory_mb {
        return Err(SchedulerError::MemoryExceeded);
    }
}
```

---

## YAML Schema

### Full Example

```yaml
nodes:
  node01:
    available_cpus: [2, 3]
    max_memory_mb: 4096
    architecture: "aarch64"
    location: "front_sensor_unit"
    description: "Perception and sensor fusion node"

  node02:
    available_cpus: [0, 1, 2, 3]
    max_memory_mb: 8192
    architecture: "x86_64"
    location: "compute_unit"
    description: "High-performance compute node"

  node03:
    available_cpus: [4, 5, 6, 7, 8, 9, 10, 11]
    # max_memory_mb omitted → defaults to u64::MAX (unconstrained)
    architecture: "aarch64"
    location: "rear_compute_cluster"
```

### Field Descriptions

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `available_cpus` | `Vec<u32>` | ✅ Yes | N/A | List of CPU IDs available on this node |
| `max_memory_mb` | `u64` | ❌ No | `u64::MAX` | Maximum memory in MB (u64::MAX = unconstrained) |
| `architecture` | `String` | ❌ No | `""` | CPU architecture (aarch64, x86_64, etc.) |
| `location` | `String` | ❌ No | `""` | Physical location (documentation only) |
| `description` | `String` | ❌ No | `""` | Node purpose (documentation only) |

---

## Error Handling

### C++ Error Handling

```cpp
bool LoadFromFile(const std::string& path) {
    try {
        YAML::Node root = YAML::LoadFile(path);
        // ... parse
        return true;
    } catch (const YAML::Exception& e) {
        LOG_ERROR("YAML parse error: " << e.what());
        return false;
    }
}
```

**Issues:**
- `bool` return doesn't explain what failed
- No file I/O error details
- Caller doesn't know if file missing vs. invalid YAML

### Rust Error Handling

```rust
pub fn load_from_file(&mut self, path: &Path) -> Result<()> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("Cannot open configuration file: {}", path.display()))?;

    let file: NodeConfigFile = serde_yaml::from_str(&content)
        .with_context(|| format!("Failed to parse YAML file: {}", path.display()))?;

    // ...
    Ok(())
}
```

**Error Messages:**
```
Cannot open configuration file: /path/to/nodes.yaml: No such file or directory

Failed to parse YAML file: /path/to/nodes.yaml: missing field `available_cpus` at line 3 column 5
```

**Benefits:**
- **Context Chain:** `with_context()` adds file path to underlying I/O error
- **Serde Errors:** Include line/column numbers for parse errors
- **Propagation:** `?` operator propagates errors with full context

---

## Usage Example

### C++ Usage

```cpp
auto node_config_mgr = std::make_shared<NodeConfigManager>();

if (!node_config_mgr->LoadFromFile("/etc/timpani/nodes.yaml")) {
    LOG_ERROR("Failed to load configuration");
    return -1;
}

const NodeConfig* node = node_config_mgr->GetNodeConfig("node01");
if (node == nullptr) {
    LOG_ERROR("Node not found");
    return -1;
}

std::cout << "Node: " << node->name
          << ", CPUs: " << node->available_cpus.size() << std::endl;
```

### Rust Usage

```rust
let mut node_config_mgr = NodeConfigManager::new();

node_config_mgr.load_from_file(Path::new("/etc/timpani/nodes.yaml"))?;

let node = node_config_mgr.get_node_config("node01")
    .ok_or_else(|| anyhow!("Node 'node01' not found"))?;

info!(
    "Node: {}, CPUs: {}, Memory: {}MB",
    node.name,
    node.cpu_count(),
    node.max_memory_mb
);

// Iterate all nodes
for (name, config) in node_config_mgr.get_all_nodes() {
    info!("  {} → {} CPUs", name, config.cpu_count());
}
```

---

## Injection Pattern

### C++ (Constructor Injection)

```cpp
class GlobalScheduler {
    std::shared_ptr<NodeConfigManager> node_config_mgr_;

public:
    explicit GlobalScheduler(std::shared_ptr<NodeConfigManager> mgr)
        : node_config_mgr_(mgr) {}
};

// Usage
auto node_mgr = std::make_shared<NodeConfigManager>();
auto scheduler = std::make_shared<GlobalScheduler>(node_mgr);
```

### Rust (Arc Injection)

```rust
pub struct GlobalScheduler {
    node_config_manager: Arc<NodeConfigManager>,
}

impl GlobalScheduler {
    pub fn new(node_config_manager: Arc<NodeConfigManager>) -> Self {
        Self { node_config_manager }
    }
}

// Usage
let node_mgr = Arc::new(node_config_manager);
let scheduler = GlobalScheduler::new(Arc::clone(&node_mgr));
```

**Pattern:** Single `NodeConfigManager` instance loaded at startup, wrapped in `Arc`, cloned and injected into all components that need node information.

---

## Testing

### C++ Testing

```cpp
TEST_F(NodeConfigManagerTest, LoadValidFile) {
    NodeConfigManager mgr;
    bool result = mgr.LoadFromFile("test_configs/nodes.yaml");

    EXPECT_TRUE(result);
    EXPECT_GT(mgr.GetAllNodes().size(), 0);
}
```

### Rust Testing

```rust
#[test]
fn test_load_valid_config() -> Result<()> {
    let mut mgr = NodeConfigManager::new();

    let temp_yaml = r#"
nodes:
  test_node:
    available_cpus: [0, 1, 2, 3]
    max_memory_mb: 4096
    architecture: "aarch64"
"#;

    let temp_file = NamedTempFile::new()?;
    std::fs::write(&temp_file, temp_yaml)?;

    mgr.load_from_file(temp_file.path())?;

    assert!(mgr.is_loaded());
    assert_eq!(mgr.node_count(), 1);

    let node = mgr.get_node_config("test_node").unwrap();
    assert_eq!(node.cpu_count(), 4);
    assert_eq!(node.max_memory_mb, 4096);
    assert_eq!(node.architecture, "aarch64");

    Ok(())
}

#[test]
fn test_missing_field_uses_default() -> Result<()> {
    let yaml = r#"
nodes:
  minimal:
    available_cpus: [0, 1]
"#;

    let temp_file = NamedTempFile::new()?;
    std::fs::write(&temp_file, yaml)?;

    let mut mgr = NodeConfigManager::new();
    mgr.load_from_file(temp_file.path())?;

    let node = mgr.get_node_config("minimal").unwrap();
    assert_eq!(node.max_memory_mb, u64::MAX); // Default
    assert_eq!(node.architecture, ""); // Default

    Ok(())
}

#[test]
fn test_empty_file_uses_default_node() -> Result<()> {
    let yaml = "nodes: {}\n";

    let temp_file = NamedTempFile::new()?;
    std::fs::write(&temp_file, yaml)?;

    let mut mgr = NodeConfigManager::new();
    mgr.load_from_file(temp_file.path())?;

    // Should auto-insert "default_node"
    assert_eq!(mgr.node_count(), 1);
    assert!(mgr.get_node_config("default_node").is_some());

    Ok(())
}
```

---

## Migration Notes

### What Changed

1. **Parser:** Manual YAML parsing → Serde automatic deserialization
2. **Error Handling:** `bool` → `Result<(), anyhow::Error>` with context
3. **Type Safety:** Runtime validation → Compile-time schema
4. **CPU Type:** `std::vector<int>` → `Vec<u32>` (unsigned)
5. **Optional Fields:** Manual checks → `Option<T>` + defaults
6. **Memory Sentinel:** Implicit → Explicit `u64::MAX`

### What Stayed the Same

1. **YAML Format:** Identical structure
2. **NodeConfig Fields:** Same fields, same semantics
3. **Default Fallback:** Still inserts default_node if empty
4. **Access Pattern:** Read-only access via getter methods
5. **Reload Support:** Clear and re-parse capability

---

**Document Version:** 1.0
**Last Updated:** May 12, 2026
**Status:** ✅ Complete
**Verified Against:** `timpani_rust/timpani-o/src/config/mod.rs` (actual implementation)
