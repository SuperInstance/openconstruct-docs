# ARCHITECTURE-DEEP.md — OpenConstruct Internals

> **Audience:** Engineers who want to understand every layer. This is not a tutorial. This is the map.

OpenConstruct is a layered agent framework built around a MUD-like text shell. Each layer is a crate (or tight cluster of crates) with explicit contracts between them. The system is designed so that any layer can be replaced, mocked, or extended without touching the others — as long as you honour the trait contracts.

---

## 1. Layer 0: The Shell (`plato-shell`)

The shell is the root of everything. Every agent session, every interactive prompt, every daemon bootstrap — it all funnels through here.

### 1.1 Architecture

```
┌─────────────────────────────────────────────┐
│                  plato-shell                 │
│  ┌─────────┐  ┌──────────┐  ┌────────────┐ │
│  │  Input   │→│  Parser   │→│  Dispatcher │ │
│  │  Reader  │  │  (nom)    │  │  (routes)   │ │
│  └─────────┘  └──────────┘  └─────┬──────┘ │
│                                     │        │
│  ┌──────────┐  ┌──────────┐  ┌─────▼──────┐ │
│  │  Session  │  │  Module   │  │  Output    │ │
│  │  Store    │  │  Registry │  │  Writer    │ │
│  └──────────┘  └──────────┘  └────────────┘ │
└─────────────────────────────────────────────┘
```

### 1.2 Command Parsing

Commands are parsed with `nom` into a structured representation before routing:

```rust
/// Parsed command from user input
#[derive(Debug, Clone)]
pub struct Command {
    /// The verb: "look", "go", "use", "say", etc.
    pub verb: Verb,
    /// Positional arguments
    pub args: Vec<Argument>,
    /// Named flags: --force, --verbose
    pub flags: HashMap<String, FlagValue>,
    /// Raw input string (for passthrough)
    pub raw: String,
}

#[derive(Debug, Clone, PartialEq)]
pub enum Verb {
    Look,
    Go(Direction),
    Use(String),
    Say(String),
    Help(Option<String>),
    Inventory,
    Status,
    Examine(String),
    Custom(String),
}

#[derive(Debug, Clone)]
pub enum Argument {
    Text(String),
    Number(f64),
    Target(EntityRef),
    Expression(Expr),
}

#[derive(Debug, Clone)]
pub enum FlagValue {
    Boolean(bool),
    String(String),
    Number(f64),
}
```

### 1.3 Module Loading

Modules implement the `ShellModule` trait and are loaded at runtime:

```rust
/// A plato-shell loadable module
#[async_trait]
pub trait ShellModule: Send + Sync {
    /// Module identifier (e.g., "plato-vision")
    fn name(&self) -> &str;

    /// Initialize the module, given access to the shell context
    async fn init(&mut self, ctx: &mut ShellContext) -> Result<(), ModuleError>;

    /// Handle a command routed to this module
    async fn handle_command(
        &self,
        cmd: &Command,
        ctx: &mut SessionContext,
    ) -> Result<CommandResult, ModuleError>;

    /// Teardown — called on unload
    async fn teardown(&mut self) -> Result<(), ModuleError>;

    /// Commands this module registers
    fn commands(&self) -> Vec<CommandSpec> {
        vec![]
    }
}

/// Shell-wide context available to all modules
pub struct ShellContext {
    pub module_registry: ModuleRegistry,
    pub config: Arc<RwLock<ShellConfig>>,
    pub event_bus: EventBus,
    pub session_store: SessionStore,
    pub sense_router: SenseRouter,
}

/// Per-session context
pub struct SessionContext {
    pub session_id: SessionId,
    pub agent_id: AgentId,
    pub inventory: Inventory,
    pub location: RoomId,
    pub state: HashMap<String, Value>,
    pub output: Box<dyn OutputWriter>,
}
```

### 1.4 Session Persistence

Sessions are serialized to disk as JSON snapshots:

```rust
#[derive(Serialize, Deserialize)]
pub struct SessionSnapshot {
    pub id: SessionId,
    pub agent_id: AgentId,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub location: RoomId,
    pub inventory: Vec<Item>,
    pub state: serde_json::Value,
    pub command_history: Vec<String>,
    pub module_states: HashMap<String, serde_json::Value>,
}
```

Sessions are stored under `$DATA_DIR/sessions/{session_id}.json` and rehydrated on reconnect.

### 1.5 Dispatch Flow

```
User input → Parser → Command
  → Dispatcher looks up verb in CommandRegistry
    → If module-registered: route to module.handle_command()
    → If built-in: route to shell handler
    → If unknown: "I don't understand that."
  → CommandResult → OutputWriter → User sees response
```

```rust
pub enum CommandResult {
    /// Simple text response
    Text(String),
    /// Rich response with multiple parts
    Rich(Vec<ResponsePart>),
    /// State change (moved, picked up item, etc.)
    StateChange {
        description: String,
        delta: StateDelta,
    },
    /// Redirect to another command
    Redirect(Command),
    /// No output (silent success)
    Silent,
    /// Error
    Error(CommandError),
}
```

---

## 2. Layer 1: The Senses

Senses are how the agent perceives the world. Each sense module implements the `SenseModule` trait and produces *shadows* — structured observations that flow into the fusion layer.

### 2.1 The SenseModule Trait

```rust
/// Core trait for all sense modules
#[async_trait]
pub trait SenseModule: Send + Sync {
    /// Unique sense identifier (e.g., "vision", "sonar", "manus")
    fn sense_type(&self) -> &SenseType;

    /// Start sensing. Returns a stream of shadows.
    async fn start(&mut self, config: SenseConfig) -> Result<ShadowStream, SenseError>;

    /// Stop sensing.
    async fn stop(&mut self) -> Result<(), SenseError>;

    /// Current sense status
    fn status(&self) -> SenseStatus;

    /// Policy check: can this sense operate in the current context?
    fn check_policy(&self, policy: &Policy) -> PolicyDecision;
}

/// A stream of observations from a sense
pub type ShadowStream = Pin<Box<dyn Stream<Item = Result<Shadow, SenseError>> + Send>>;

/// An observation from a sense module
#[derive(Debug, Clone, Serialize)]
pub struct Shadow {
    /// Which sense produced this
    pub sense_type: SenseType,
    /// When the observation was made
    pub timestamp: DateTime<Utc>,
    /// Structured observation data
    pub data: ShadowData,
    /// Confidence (0.0–1.0)
    pub confidence: f64,
    /// Source identifier (camera ID, sensor ID, etc.)
    pub source: String,
    /// Optional spatial reference
    pub spatial_ref: Option<SpatialRef>,
}

#[derive(Debug, Clone, Serialize)]
pub enum ShadowData {
    /// Visual observation (plato-vision)
    Visual(VisualData),
    /// Audio observation (plato-sonar)
    Audio(AudioData),
    /// Touch/manipulation observation (plato-manus)
    Tactile(TactileData),
    /// Browser DOM state (plato-playwright / plato-puppeteer)
    Dom(DomSnapshot),
    /// Raw binary payload
    Raw(Bytes),
}

#[derive(Debug, Clone)]
pub struct SenseStatus {
    pub active: bool,
    pub last_shadow: Option<DateTime<Utc>>,
    pub error_count: u64,
    pub shadow_count: u64,
    pub health: Health,
}
```

### 2.2 Sense Modules

| Crate | Sense Type | Description |
|-------|-----------|-------------|
| `plato-vision` | `vision` | Camera/screen capture via OCR + image analysis |
| `plato-sonar` | `sonar` | Audio input: speech-to-text, sound classification |
| `plato-manus` | `manus` | Touch/input device state, haptic feedback |
| `plato-playwright` | `dom` | Browser automation via Playwright (Chromium) |
| `plato-puppeteer` | `dom` | Browser automation via Puppeteer (Chrome) |
| `a2ui-render` | `render` | Text-to-component rendering pipeline |

### 2.3 Visual Shadow Example (`plato-vision`)

```rust
#[derive(Debug, Clone, Serialize)]
pub struct VisualData {
    /// Detected text regions
    pub text_regions: Vec<TextRegion>,
    /// Detected objects/classes
    pub objects: Vec<DetectedObject>,
    /// Image dimensions
    pub dimensions: (u32, u32),
    /// Frame hash for deduplication
    pub frame_hash: u64,
    /// Optional encoded thumbnail
    pub thumbnail: Option<Vec<u8>>,
}

#[derive(Debug, Clone, Serialize)]
pub struct TextRegion {
    pub text: String,
    pub bounds: BoundingBox,
    pub confidence: f64,
    pub language: Option<String>,
}

#[derive(Debug, Clone, Serialize)]
pub struct DetectedObject {
    pub label: String,
    pub bounds: BoundingBox,
    pub confidence: f64,
    pub attributes: HashMap<String, String>,
}
```

### 2.4 Audio Shadow Example (`plato-sonar`)

```rust
#[derive(Debug, Clone, Serialize)]
pub struct AudioData {
    /// Transcribed speech (if any)
    pub transcript: Option<String>,
    /// Detected sounds and their classifications
    pub sound_events: Vec<SoundEvent>,
    /// Volume level (0.0–1.0)
    pub volume: f64,
    /// Duration of the audio window
    pub window_duration_ms: u64,
    /// Language detected
    pub language: Option<String>,
}

#[derive(Debug, Clone, Serialize)]
pub struct SoundEvent {
    pub classification: String,
    pub onset_ms: u64,
    pub duration_ms: u64,
    pub confidence: f64,
}
```

### 2.5 DOM Shadow Example (`plato-playwright`)

```rust
#[derive(Debug, Clone, Serialize)]
pub struct DomSnapshot {
    pub url: String,
    pub title: String,
    /// Accessible text content
    pub text_content: String,
    /// Interactive elements
    pub interactables: Vec<InteractableElement>,
    /// Page metrics
    pub viewport_size: (u32, u32),
    pub scroll_position: (u32, u32),
}

#[derive(Debug, Clone, Serialize)]
pub struct InteractableElement {
    pub selector: String,
    pub tag: String,
    pub text: String,
    pub role: Option<String>,
    pub visible: bool,
    pub bounds: Option<BoundingBox>,
}
```

### 2.6 Policy Gating (`plato-policy`)

Every sense operation is gated by `plato-policy`:

```rust
#[derive(Debug, Clone)]
pub enum PolicyDecision {
    Allow,
    Deny { reason: String },
    Defer { requires: Vec<PolicyId> },
    RateLimit { remaining: u32, reset_at: DateTime<Utc> },
}

pub struct Policy {
    /// Maximum shadows per second per sense
    pub max_shadow_rate: u32,
    /// Allowed sense types for this agent
    pub allowed_senses: Vec<SenseType>,
    /// Data retention policy
    pub retention: RetentionPolicy,
    /// Privacy filters to apply
    pub privacy_filters: Vec<PrivacyFilter>,
    /// Whether network-bound senses are allowed
    pub network_allowed: bool,
}
```

---

## 3. Layer 2: Fusion (`plato-correlator`)

The correlator ingests shadows from all senses, fuses them into correlated events, and classifies severity.

### 3.1 Architecture

```
          ┌──────────────┐
          │  Shadow Bus  │  (tokio broadcast channel)
          └──────┬───────┘
                 │
    ┌────────────┼────────────┐
    ▼            ▼            ▼
┌────────┐  ┌────────┐  ┌────────┐
│Temporal │  │  Rule   │  │Priority │
│ Window  │  │  Engine │  │  Queue  │
└────┬───┘  └────┬───┘  └────┬───┘
     │           │           │
     └───────────┼───────────┘
                 ▼
          ┌──────────────┐
          │  Correlated   │
          │    Event      │
          └──────┬───────┘
                 │
                 ▼
          ┌──────────────┐
          │  Dispatcher   │
          └──────────────┘
```

### 3.2 Shadow Ingestion

```rust
/// The correlator ingests shadows via a broadcast channel
pub struct Correlator {
    /// Input channel for shadows
    shadow_rx: BroadcastReceiver<Shadow>,
    /// Temporal window buffers per sense type
    windows: HashMap<SenseType, TemporalWindow>,
    /// Active fusion rules
    rules: Vec<FusionRule>,
    /// Output event channel
    event_tx: mpsc::Sender<CorrelatedEvent>,
    /// Priority queue for ranked events
    priority_queue: PriorityQueue<CorrelatedEvent>,
}

/// A temporal window buffers shadows within a time range
pub struct TemporalWindow {
    pub sense_type: SenseType,
    pub buffer: Vec<Shadow>,
    pub window_duration: Duration,
    pub max_buffer_size: usize,
    pub last_flush: DateTime<Utc>,
}
```

### 3.3 Rule-Based Event Fusion

```rust
/// A fusion rule combines shadows from multiple senses into an event
#[derive(Debug, Clone)]
pub struct FusionRule {
    pub id: RuleId,
    pub name: String,
    /// Required sense types for this rule to fire
    pub required_senses: Vec<SenseType>,
    /// Maximum time gap between shadows to correlate
    pub temporal_tolerance: Duration,
    /// Minimum confidence threshold
    pub min_confidence: f64,
    /// The fusion function
    pub fuse_fn: FuseFn,
}

pub type FuseFn = Box<dyn Fn(&[&Shadow]) -> FusionResult + Send + Sync>;

pub enum FusionResult {
    /// Produced a correlated event
    Event(CorrelatedEvent),
    /// Need more shadows before we can fuse
    Pending,
    /// Shadows don't match this rule
    Reject,
}

/// A correlated event produced by fusion
#[derive(Debug, Clone, Serialize)]
pub struct CorrelatedEvent {
    pub id: EventId,
    pub timestamp: DateTime<Utc>,
    /// The shadows that were fused
    pub source_shadows: Vec<ShadowId>,
    /// The rule that produced this event
    pub fusion_rule: RuleId,
    /// Human-readable description
    pub description: String,
    /// Severity classification
    pub severity: Severity,
    /// Structured event data
    pub data: EventData,
    /// Priority for the dispatch queue
    pub priority: Priority,
}
```

### 3.4 Severity Classification

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize)]
pub enum Severity {
    /// Informational — logged but no action needed
    Info = 0,
    /// Low — worth noting, no urgency
    Low = 1,
    /// Medium — should be addressed soon
    Medium = 2,
    /// High — needs attention promptly
    High = 3,
    /// Critical — immediate action required
    Critical = 4,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize)]
pub enum Priority {
    Background = 0,
    Normal = 1,
    Elevated = 2,
    Urgent = 3,
    Immediate = 4,
}
```

### 3.5 Priority Queue

Events are dispatched based on priority with exponential backoff for retries:

```rust
pub struct PriorityQueue<T: Prioritized> {
    heaps: [BinaryHeap<PrioritizedItem<T>>; 5], // one per priority level
    inflight: HashMap<T::Id, InFlightStatus<T>>,
    retry_policy: RetryPolicy,
}

pub trait Prioritized: Clone + Send + Sync {
    type Id: Hash + Eq + Clone;
    fn priority(&self) -> Priority;
    fn id(&self) -> &Self::Id;
}
```

---

## 4. Layer 3: Communication (`plato-tick`, `plato-a2a`, `shell-mesh`)

Agents need to talk — to each other, to humans, and to the fleet. This layer handles all of it.

### 4.1 Tick Board (`plato-tick`)

The tick board is a lightweight message board for inter-agent communication within a single process:

```rust
/// Inter-agent message bus
pub struct TickBoard {
    /// Channel registry: agent_id → sender
    channels: RwLock<HashMap<AgentId, mpsc::Sender<TickMessage>>>,
    /// Broadcast for system-wide announcements
    broadcast: BroadcastSender<TickMessage>,
    /// Message store for replay
    store: Arc<dyn MessageStore>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TickMessage {
    pub id: MessageId,
    pub from: AgentId,
    pub to: Option<AgentId>,  // None = broadcast
    pub timestamp: DateTime<Utc>,
    pub message_type: TickType,
    pub payload: serde_json::Value,
    pub ttl: Option<Duration>,
    pub in_reply_to: Option<MessageId>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum TickType {
    /// Direct message to another agent
    Direct,
    /// Broadcast to all agents
    Broadcast,
    /// Request requiring a response
    Request { timeout: Duration },
    /// Response to a request
    Response { request_id: MessageId },
    /// Event notification
    Event { event_type: String },
}
```

### 4.2 A2A Wire Protocol (`plato-a2a`)

For agent-to-agent discovery and communication across processes, we implement the A2A (Agent-to-Agent) protocol:

```rust
/// A2A agent card — describes an agent's capabilities
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AgentCard {
    pub agent_id: AgentId,
    pub name: String,
    pub description: String,
    pub version: String,
    /// URL where this agent can be reached
    pub endpoint: String,
    /// Capabilities this agent exposes
    pub capabilities: Vec<Capability>,
    /// Authentication schemes supported
    pub auth_schemes: Vec<AuthScheme>,
    /// Supported interaction modes
    pub interaction_modes: Vec<InteractionMode>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Capability {
    pub id: String,
    pub name: String,
    pub description: String,
    pub input_schema: serde_json::Value,
    pub output_schema: serde_json::Value,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum InteractionMode {
    /// Request/response
    Synchronous,
    /// Streaming response
    Streaming,
    /// Pub/sub event-based
    Asynchronous,
}

/// A2A wire message format
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct A2AMessage {
    pub version: String,        // "1.0"
    pub message_type: A2AType,
    pub payload: serde_json::Value,
    pub metadata: HashMap<String, String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum A2AType {
    Discover,
    Advertise(AgentCard),
    Invoke { capability_id: String },
    InvokeResult,
    Event { event_type: String },
    Error { code: u16, message: String },
}
```

### 4.3 Mesh Networking (`shell-mesh`)

For fleet communication across unreliable networks:

```rust
/// Mesh node configuration
pub struct MeshNode {
    pub node_id: NodeId,
    pub address: SocketAddr,
    pub peers: Vec<PeerConfig>,
    pub topology: Topology,
    pub encryption: EncryptionConfig,
}

#[derive(Debug, Clone)]
pub enum Topology {
    /// Star: all nodes connect through a hub
    Star { hub: NodeId },
    /// Full mesh: every node connects to every other
    FullMesh,
    /// Partial mesh: defined peer connections
    PartialMesh { connections: Vec<(NodeId, NodeId)> },
}

/// Mesh message with routing metadata
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MeshMessage {
    pub id: MessageId,
    pub source: NodeId,
    pub destination: Destination,
    pub payload: Vec<u8>,
    pub hops: Vec<NodeId>,
    pub ttl: u8,
    pub timestamp: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Destination {
    /// Specific node
    Unicast(NodeId),
    /// All nodes
    Flood,
    /// Nodes matching a predicate
    Group(String),
}

/// Mesh routing trait
#[async_trait]
pub trait MeshRouter: Send + Sync {
    /// Route a message to its destination
    async fn route(&self, msg: &MeshMessage) -> Result<Vec<NodeId>, MeshError>;
    /// Get the next hop for a destination
    async fn next_hop(&self, dest: &NodeId) -> Option<NodeId>;
    /// Handle a received message
    async fn on_receive(&self, msg: MeshMessage) -> Result<(), MeshError>;
}
```

### 4.4 Star vs Mesh Routing

In **star topology**, all messages route through the hub:

```
Agent A → Hub → Agent B
Agent C → Hub → Agent A
```

In **full mesh**, messages route directly:

```
Agent A → Agent B (direct)
Agent C → Agent A (direct)
```

The `shell-mesh` router abstracts this so that the upper layers don't need to know which topology is in use:

```rust
pub struct MeshRouterImpl {
    topology: Topology,
    routing_table: RwLock<RoutingTable>,
    transport: Box<dyn Transport>,
}
```

---

## 5. Layer 4: Fleet (`plato-fleet`, `plato-transport`)

The fleet layer manages a collection of nodes — including ESP32-based room devices — as a unified resource pool.

### 5.1 Node Registration

```rust
/// Fleet manager — tracks all known nodes
pub struct FleetManager {
    /// Registered nodes
    nodes: RwLock<HashMap<NodeId, NodeInfo>>,
    /// Transport for node communication
    transport: Arc<dyn Transport>,
    /// Health checker
    health: HealthChecker,
    /// Resource tracker
    resources: ResourceTracker,
}

#[derive(Debug, Clone, Serialize)]
pub struct NodeInfo {
    pub id: NodeId,
    pub name: String,
    pub node_type: NodeType,
    pub status: NodeStatus,
    pub capabilities: Vec<String>,
    pub endpoint: String,
    pub registered_at: DateTime<Utc>,
    pub last_heartbeat: DateTime<Utc>,
    pub metadata: HashMap<String, String>,
}

#[derive(Debug, Clone, Serialize)]
pub enum NodeType {
    /// Full agent node (runs plato-shell)
    Agent,
    /// Sensor-only node (e.g., camera module)
    Sensor,
    /// ESP32 room device
    Esp32Room(Esp32RoomInfo),
    /// Bridge node (connects different transports)
    Bridge,
}

#[derive(Debug, Clone, Serialize)]
pub struct Esp32RoomInfo {
    pub room_id: RoomId,
    pub sensors: Vec<SensorDescriptor>,
    pub actuators: Vec<ActuatorDescriptor>,
    pub firmware_version: String,
    pub ip_address: IpAddr,
    pub wifi_rssi: i32,
}

#[derive(Debug, Clone, Serialize)]
pub enum NodeStatus {
    Online,
    Offline,
    Degraded { issues: Vec<String> },
    Draining,
}
```

### 5.2 Resource Tracking

```rust
pub struct ResourceTracker {
    /// Current resource usage per node
    usage: RwLock<HashMap<NodeId, ResourceUsage>>,
}

#[derive(Debug, Clone, Serialize)]
pub struct ResourceUsage {
    pub cpu_percent: f64,
    pub memory_used_bytes: u64,
    pub memory_total_bytes: u64,
    pub active_tasks: u32,
    pub max_tasks: u32,
    pub network_bytes_per_sec: f64,
    pub last_updated: DateTime<Utc>,
}

/// Delegation decision
pub struct DelegationDecision {
    pub target_node: NodeId,
    pub reason: DelegationReason,
    pub estimated_load: f64,
}

pub enum DelegationReason {
    LeastLoaded,
    HasCapability(String),
    SameRoom(RoomId),
    Proximity(SpatialRef),
    ManualAssignment,
}
```

### 5.3 Transport Abstraction (`plato-transport`)

```rust
/// Abstract transport for node communication
#[async_trait]
pub trait Transport: Send + Sync {
    /// Connect to a node
    async fn connect(&self, endpoint: &str) -> Result<Connection, TransportError>;
    /// Send a message
    async fn send(&self, conn: &Connection, data: &[u8]) -> Result<(), TransportError>;
    /// Receive a message
    async fn recv(&self, conn: &Connection) -> Result<Vec<u8>, TransportError>;
    /// Close a connection
    async fn close(&self, conn: Connection) -> Result<(), TransportError>;
}

/// IPC transport (same process, different tasks)
pub struct IpcTransport { /* channels */ }

/// Unix socket transport (same machine, different processes)
pub struct UnixTransport { /* socket pairs */ }

/// TCP transport (cross-machine)
pub struct TcpTransport {
    tls_config: Option<TlsConfig>,
    keepalive: Duration,
}
```

---

## 6. Layer 5: Execution (`plato-workflow`, `plato-sandbox`, `plato-contract`)

### 6.1 DAG-Based Task Orchestration (`plato-workflow`)

```rust
/// A workflow is a DAG of tasks
pub struct Workflow {
    pub id: WorkflowId,
    pub name: String,
    /// Task nodes
    pub tasks: HashMap<TaskId, TaskNode>,
    /// Edges: (from, to, edge_type)
    pub edges: Vec<(TaskId, TaskId, EdgeType)>,
    /// Workflow-level configuration
    pub config: WorkflowConfig,
}

pub struct TaskNode {
    pub id: TaskId,
    pub name: String,
    /// The executor for this task
    pub executor: TaskExecutor,
    /// Input schema
    pub input_schema: serde_json::Value,
    /// Output schema
    pub output_schema: serde_json::Value,
    /// Timeout for this task
    pub timeout: Duration,
    /// Retry policy
    pub retry: RetryPolicy,
    /// Dependencies (must all complete before this starts)
    pub depends_on: Vec<TaskId>,
}

pub enum EdgeType {
    /// Sequential: B runs after A
    Sequential,
    /// Data flow: A's output feeds into B's input
    DataFlow { mapping: FieldMapping },
    /// Conditional: B runs only if A produces a specific result
    Conditional { predicate: Predicate },
}

/// Task executor trait
#[async_trait]
pub trait TaskExecutor: Send + Sync {
    async fn execute(
        &self,
        input: serde_json::Value,
        ctx: &WorkflowContext,
    ) -> Result<TaskOutput, WorkflowError>;
}

/// Runtime execution state
pub struct WorkflowExecution {
    pub workflow_id: WorkflowId,
    pub execution_id: ExecutionId,
    pub status: ExecutionStatus,
    pub task_states: HashMap<TaskId, TaskState>,
    pub started_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
}

pub enum ExecutionStatus {
    Pending,
    Running,
    Completed,
    Failed { error: String },
    Cancelled,
    Paused,
}
```

### 6.2 Sandboxed Execution (`plato-sandbox`)

```rust
/// Sandboxed task execution environment
pub struct Sandbox {
    /// Resource limits
    pub limits: ResourceLimits,
    /// Allowed syscalls
    pub syscall_filter: SyscallFilter,
    /// Network policy
    pub network: NetworkPolicy,
    /// Filesystem access
    pub fs: FsPolicy,
    /// Timeout
    pub timeout: Duration,
}

#[derive(Debug, Clone)]
pub struct ResourceLimits {
    pub max_memory_bytes: u64,
    pub max_cpu_time: Duration,
    pub max_processes: u32,
    pub max_file_size_bytes: u64,
    pub max_open_files: u32,
}

#[derive(Debug, Clone)]
pub struct NetworkPolicy {
    pub allowed_hosts: Vec<String>,
    pub allowed_ports: Vec<u16>,
    pub denied_hosts: Vec<String>,
    pub dns_allowed: bool,
}

#[derive(Debug, Clone)]
pub struct FsPolicy {
    pub read_only_paths: Vec<PathBuf>,
    pub read_write_paths: Vec<PathBuf>,
    pub denied_paths: Vec<PathBuf>,
    pub tmp_size_bytes: u64,
}

/// Sandbox execution result
pub struct SandboxResult {
    pub exit_code: i32,
    pub stdout: Vec<u8>,
    pub stderr: Vec<u8>,
    pub resource_usage: ResourceUsage,
    pub duration: Duration,
    pub killed: bool,
    pub kill_reason: Option<KillReason>,
}
```

### 6.3 Inter-Agent Contracts (`plato-contract`)

```rust
/// A contract between two agents
pub struct AgentContract {
    pub id: ContractId,
    /// The provider agent
    pub provider: AgentId,
    /// The consumer agent
    pub consumer: AgentId,
    /// What the provider agrees to deliver
    pub obligations: Vec<Obligation>,
    /// What the consumer agrees to provide in return
    pub considerations: Vec<Consideration>,
    /// Duration of the contract
    pub term: ContractTerm,
    /// Verification method
    pub verification: VerificationMethod,
    /// State
    pub state: ContractState,
}

pub struct Obligation {
    pub id: String,
    pub description: String,
    pub input_schema: serde_json::Value,
    pub output_schema: serde_json::Value,
    pub sla: Sla,
}

pub struct Sla {
    pub availability: f64,       // e.g., 0.999
    pub max_latency: Duration,
    pub error_budget: f64,       // e.g., 0.01
}

pub enum VerificationMethod {
    /// Provider self-reports
    SelfAttestation,
    /// Consumer verifies output
    ConsumerVerification { verifier: String },
    /// Third-party verification
    ThirdParty { verifier_agent: AgentId },
    /// Cryptographic proof
    CryptographicProof { proof_type: String },
}

pub enum ContractState {
    Proposed,
    Active { started_at: DateTime<Utc> },
    Fulfilled,
    Breached { reason: String, at: DateTime<Utc> },
    Expired,
    Cancelled { reason: String },
}
```

---

## 7. Layer 6: State (`plato-memory`, `plato-session`, `plato-config`)

### 7.1 Agent Memory (`plato-memory`)

Agent memory is modeled after human memory systems:

```rust
/// Agent memory store with three memory types
pub struct MemoryStore {
    /// Episodic: specific events and experiences
    pub episodic: EpisodicMemory,
    /// Semantic: general knowledge and facts
    pub semantic: SemanticMemory,
    /// Procedural: learned skills and patterns
    pub procedural: ProceduralMemory,
}

/// Episodic memory — specific events with temporal context
pub struct EpisodicMemory {
    store: Box<dyn MemoryBackend<EpisodicEntry>>,
    index: TemporalIndex,
    decay: DecayPolicy,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EpisodicEntry {
    pub id: MemoryId,
    pub timestamp: DateTime<Utc>,
    /// What happened
    pub event: String,
    /// Structured event data
    pub data: serde_json::Value,
    /// Emotional valence (-1.0 to 1.0)
    pub valence: f64,
    /// Importance (0.0 to 1.0)
    pub importance: f64,
    /// Associated tags
    pub tags: Vec<String>,
    /// Related memories
    pub associations: Vec<MemoryId>,
    /// Access count (for decay calculation)
    pub access_count: u32,
    /// Last accessed
    pub last_accessed: DateTime<Utc>,
}

/// Semantic memory — facts and relationships
pub struct SemanticMemory {
    store: Box<dyn MemoryBackend<SemanticEntry>>,
    graph: KnowledgeGraph,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SemanticEntry {
    pub id: MemoryId,
    pub subject: String,
    pub predicate: String,
    pub object: String,
    pub confidence: f64,
    pub source: MemoryId,    // which episodic memory this came from
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

/// Procedural memory — learned patterns and skills
pub struct ProceduralMemory {
    store: Box<dyn MemoryBackend<ProceduralEntry>>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProceduralEntry {
    pub id: MemoryId,
    pub name: String,
    /// Trigger condition
    pub trigger: Pattern,
    /// The learned procedure
    pub procedure: Procedure,
    /// Success rate
    pub success_rate: f64,
    /// Times executed
    pub execution_count: u32,
    /// Last used
    pub last_used: DateTime<Utc>,
}

/// Memory decay policy
pub struct DecayPolicy {
    /// Half-life of memories in hours
    pub half_life: f64,
    /// Minimum importance to retain regardless of decay
    pub retention_threshold: f64,
    /// Whether to consolidate memories during quiet periods
    pub auto_consolidate: bool,
}
```

### 7.2 Session Management (`plato-session`)

```rust
pub struct SessionManager {
    /// Active sessions
    sessions: RwLock<HashMap<SessionId, Session>>,
    /// Session store for persistence
    store: Arc<dyn SessionStore>,
    /// Checkpoint interval
    checkpoint_interval: Duration,
    /// Maximum concurrent sessions
    max_sessions: usize,
}

#[derive(Debug)]
pub struct Session {
    pub id: SessionId,
    pub agent_id: AgentId,
    pub state: SessionState,
    pub created_at: DateTime<Utc>,
    pub last_active: DateTime<Utc>,
    pub checkpoints: Vec<Checkpoint>,
    pub modules: HashMap<String, ModuleSessionState>,
}

pub enum SessionState {
    Active,
    Idle,
    Suspended,
    Closed,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct Checkpoint {
    pub id: CheckpointId,
    pub timestamp: DateTime<Utc>,
    /// Full serialized state at checkpoint time
    pub snapshot: serde_json::Value,
    /// Trigger for this checkpoint
    pub trigger: CheckpointTrigger,
    pub size_bytes: u64,
}

pub enum CheckpointTrigger {
    /// Automatic periodic checkpoint
    Automatic,
    /// Before a risky operation
    PreOperation { operation: String },
    /// User-requested
    Manual,
    /// Before session suspension
    Suspend,
}
```

### 7.3 Configuration (`plato-config`)

```rust
pub struct ConfigManager {
    /// Current configuration
    config: Arc<RwLock<AppConfig>>,
    /// File watcher for hot-reload
    watcher: FileWatcher,
    /// Change subscribers
    subscribers: Vec<Box<dyn ConfigSubscriber>>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AppConfig {
    pub shell: ShellConfig,
    pub senses: SenseConfigs,
    pub correlator: CorrelatorConfig,
    pub fleet: FleetConfig,
    pub workflow: WorkflowConfig,
    pub memory: MemoryConfig,
    pub mesh: MeshConfig,
    pub observe: ObserveConfig,
}

/// Hot-reload: when config file changes, subscribers are notified
#[async_trait]
pub trait ConfigSubscriber: Send + Sync {
    async fn on_config_change(&self, old: &AppConfig, new: &AppConfig);
    fn interested_in(&self) -> Vec<String>; // config path prefixes
}
```

---

## 8. Layer 7: Verification (`openconstruct-mercury`)

Mercury is the formal verification layer. It provides compile-time and runtime guarantees about system behavior.

### 8.1 Policy Consistency Proofs

```rust
/// Verifies that the global policy set is internally consistent
pub fn verify_policy_consistency(policies: &[Policy]) -> Result<PolicyProof, VerificationError> {
    // 1. No two policies contradict each other
    // 2. No circular dependencies
    // 3. All referenced entities exist
    // 4. Rate limits are physically achievable
    // Returns a proof that can be checked in O(1) at runtime
}

pub struct PolicyProof {
    pub hash: Blake3Hash,
    pub policy_count: usize,
    pub verified_at: DateTime<Utc>,
    /// Assumptions the proof relies on
    pub assumptions: Vec<String>,
}
```

### 8.2 CR Correctness Guarantees

```rust
/// Verifies that all CR (Command-Response) pairs satisfy their contracts
pub fn verify_cr_contracts(
    commands: &[CommandSpec],
    handlers: &[CommandHandlerSpec],
) -> Result<CrProof, VerificationError> {
    // For each command:
    //   1. Input schema is satisfiable
    //   2. Output schema covers all response paths
    //   3. Error types are exhaustive
    //   4. Side effects are declared and permitted
}

pub struct CrProof {
    pub hash: Blake3Hash,
    pub verified_pairs: usize,
    pub uncovered_paths: Vec<String>, // should be empty
}
```

### 8.3 Sense Type Safety

```rust
/// Verifies that all sense data flows are type-safe end-to-end
pub fn verify_sense_type_safety(
    senses: &[SenseModuleSpec],
    correlator: &CorrelatorSpec,
) -> Result<SenseTypeProof, VerificationError> {
    // 1. Every ShadowData variant is consumed by at least one fusion rule
    // 2. No fusion rule expects a ShadowData variant that no sense produces
    // 3. All type conversions are lossless or explicitly lossy
    // 4. Confidence values are properly propagated
}

pub struct SenseTypeProof {
    pub hash: Blake3Hash,
    pub sense_types_verified: usize,
    pub fusion_rules_verified: usize,
}
```

### 8.4 Fleet Topology Proofs

```rust
/// Verifies that the fleet topology satisfies connectivity requirements
pub fn verify_fleet_topology(
    nodes: &[NodeInfo],
    topology: &Topology,
    requirements: &[ConnectivityRequirement],
) -> Result<TopologyProof, VerificationError> {
    // 1. All nodes are reachable
    // 2. No single point of failure (for mesh/mixed topologies)
    // 3. Latency bounds are achievable
    // 4. Redundancy requirements are met
}

pub struct ConnectivityRequirement {
    pub from: NodeType,
    pub to: NodeType,
    pub max_hops: u32,
    pub min_redundancy: u32,
    pub max_latency: Duration,
}
```

---

## 9. Layer 8: Rendering (`a2ui-render`, `a2ui-components`, `mud2scummvm`, `a2ui-cave-wall`)

### 9.1 Rendering Pipeline

```
Text Input → Component Tree → Theme Application → Target Rendering
                ↓
         UIDL Description
                ↓
    ┌───────────┼───────────┐
    ▼           ▼           ▼
  Terminal    Browser     SCUMM
  (ANSI)    (HTML/CSS)  (Adventure)
```

```rust
/// The core rendering pipeline
pub struct RenderPipeline {
    /// Text → Component parser
    pub parser: ComponentParser,
    /// Theme engine
    pub theme: ThemeEngine,
    /// Target renderers
    pub renderers: HashMap<RenderTarget, Box<dyn Renderer>>,
}

#[async_trait]
pub trait Renderer: Send + Sync {
    fn target(&self) -> RenderTarget;
    async fn render(&self, tree: &ComponentTree, theme: &Theme) -> Result<RenderedOutput, RenderError>;
}

pub enum RenderTarget {
    Terminal,
    Html,
    ScummVm,
    Discord,
    Slack,
}
```

### 9.2 Component System (`a2ui-components`)

```rust
/// A component in the rendering tree
#[derive(Debug, Clone, Serialize)]
pub struct Component {
    pub id: ComponentId,
    pub component_type: ComponentType,
    pub props: HashMap<String, PropValue>,
    pub children: Vec<Component>,
    pub styles: Option<StyleMap>,
}

#[derive(Debug, Clone, Serialize)]
pub enum ComponentType {
    /// Plain text block
    Text,
    /// Styled header
    Header { level: u8 },
    /// Container with layout
    Container { layout: Layout },
    /// Interactive button
    Button { action: String },
    /// List of items
    List { ordered: bool },
    /// Code block with syntax highlighting
    Code { language: Option<String> },
    /// Status indicator
    Status { level: StatusLevel },
    /// Custom component
    Custom { name: String },
}

#[derive(Debug, Clone, Serialize)]
pub enum Layout {
    Stack { direction: Direction },
    Grid { columns: u32 },
    Flex { wrap: bool },
}
```

### 9.3 MUD to SCUMM Bridge (`mud2scummvm`)

```rust
/// Bridges MUD room descriptions to SCUMM-style adventure game rendering
pub struct Mud2Scumm {
    /// Room description parser
    pub room_parser: RoomDescriptionParser,
    /// Sprite/object mapper
    pub object_mapper: ObjectMapper,
    /// Dialog renderer
    pub dialog_renderer: DialogRenderer,
}

/// Parsed room description ready for SCUMM rendering
#[derive(Debug, Clone)]
pub struct ScummScene {
    pub room_id: RoomId,
    pub room_name: String,
    /// Background image reference
    pub background: AssetRef,
    /// Objects in the scene
    pub objects: Vec<ScummObject>,
    /// Exit directions with hotspots
    pub exits: Vec<ScummExit>,
    /// NPCs present
    pub characters: Vec<ScummCharacter>,
    /// Ambient description text
    pub description: String,
}

#[derive(Debug, Clone)]
pub struct ScummObject {
    pub name: String,
    pub sprite: AssetRef,
    pub hotspot: BoundingBox,
    pub verbs: Vec<String>,  // "look at", "pick up", "use"
    pub state: String,
}

#[derive(Debug, Clone)]
pub struct ScummExit {
    pub direction: Direction,
    pub hotspot: BoundingBox,
    pub target_room: RoomId,
    pub label: String,
}
```

### 9.4 Cave Wall Dashboard (`a2ui-cave-wall`)

```rust
/// Real-time dashboard for fleet/sense monitoring
pub struct CaveWall {
    /// Widget registry
    pub widgets: HashMap<String, Box<dyn Widget>>,
    /// Data subscriptions
    pub subscriptions: Vec<DataSubscription>,
    /// Layout grid
    pub layout: DashboardLayout,
    /// Refresh interval
    pub refresh_interval: Duration,
}

#[async_trait]
pub trait Widget: Send + Sync {
    fn name(&self) -> &str;
    async fn render(&self, data: &DataSnapshot) -> Component;
    fn required_data(&self) -> Vec<DataKind>;
}

pub enum DataKind {
    SenseShadows { sense_type: SenseType },
    CorrelatedEvents,
    FleetStatus,
    WorkflowProgress { workflow_id: Option<WorkflowId> },
    MemoryStats,
    SystemMetrics,
}
```

---

## 10. Cross-Cutting Concerns

### 10.1 Observability (`plato-observe`)

```rust
/// Observability layer — metrics, traces, and health
pub struct Observe {
    pub metrics: MetricsSink,
    pub traces: TraceCollector,
    pub health: HealthRegistry,
}

/// Metrics are structured key-value pairs
pub struct Metric {
    pub name: String,
    pub value: MetricValue,
    pub tags: HashMap<String, String>,
    pub timestamp: DateTime<Utc>,
}

pub enum MetricValue {
    Counter(u64),
    Gauge(f64),
    Histogram(f64),
    Summary { count: u64, sum: f64, quantiles: Vec<(f64, f64)> },
}

/// Distributed trace span
pub struct Span {
    pub trace_id: TraceId,
    pub span_id: SpanId,
    pub parent_id: Option<SpanId>,
    pub operation: String,
    pub start: DateTime<Utc>,
    pub duration: Duration,
    pub status: SpanStatus,
    pub attributes: HashMap<String, String>,
    pub events: Vec<SpanEvent>,
}

/// Health check registration
pub struct HealthCheck {
    pub name: String,
    pub check_fn: Box<dyn Fn() -> HealthStatus + Send + Sync>,
    pub interval: Duration,
}

pub enum HealthStatus {
    Healthy,
    Degraded { message: String },
    Unhealthy { message: String },
}
```

### 10.2 UIDL (`plato-uidl`)

The Universal Interface Description Language describes agent interfaces in a platform-neutral way:

```rust
/// UIDL interface description
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UidlInterface {
    pub id: String,
    pub version: String,
    pub name: String,
    pub description: String,
    pub methods: Vec<UidlMethod>,
    pub events: Vec<UidlEvent>,
    pub types: Vec<UidlType>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UidlMethod {
    pub name: String,
    pub input: UidlTypeRef,
    pub output: UidlTypeRef,
    pub error: Option<UidlTypeRef>,
    pub description: String,
    pub deprecated: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UidlEvent {
    pub name: String,
    pub payload: UidlTypeRef,
    pub description: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum UidlType {
    Primitive(UidlPrimitive),
    Enum { name: String, variants: Vec<UidlVariant> },
    Struct { name: String, fields: Vec<UidlField> },
    Array { element_type: Box<UidlType> },
    Map { key_type: Box<UidlType>, value_type: Box<UidlType> },
    Optional { inner: Box<UidlType> },
    Reference { name: String },
}
```

---

## 11. Dependency Graph

```
                        ┌─────────────────────┐
                        │    plato-config      │
                        │   (Layer 6: State)   │
                        └──────────┬──────────┘
                                   │ (everyone reads config)
                                   │
    ┌──────────────────────────────┼──────────────────────────────┐
    │                              │                              │
    ▼                              ▼                              ▼
┌──────────┐              ┌───────────────┐              ┌───────────┐
│plato-shell│◄─────────────│plato-observe   │              │plato-uidl │
│ (L0)     │              │ (Cross-Cut)   │              │(Cross-Cut)│
└────┬─────┘              └───────┬───────┘              └───────────┘
     │                            │
     │ loads                      │ instruments
     │                            │
     ▼                            ▼
┌──────────────────────────────────────────────────────────┐
│                    Sense Modules (L1)                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐│
│  │ plato-vision │ │ plato-sonar │ │ plato-playwright    ││
│  └──────┬──────┘ └──────┬──────┘ │ plato-puppeteer     ││
│         │               │        └──────────┬──────────┘│
│         │               │                   │           │
│         └───────────────┼───────────────────┘           │
│                         │                               │
│              ┌──────────▼──────────┐                    │
│              │   plato-policy      │                    │
│              │   (gatekeeper)      │                    │
│              └──────────┬──────────┘                    │
└─────────────────────────┼───────────────────────────────┘
                          │ shadows
                          ▼
                ┌─────────────────┐
                │ plato-correlator │ (L2: Fusion)
                └────────┬────────┘
                         │ correlated events
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
  ┌─────────────┐ ┌────────────┐ ┌─────────────┐
  │ plato-tick   │ │ plato-a2a  │ │ shell-mesh   │
  │ (L3: Comms)  │ │ (L3: A2A)  │ │ (L3: Mesh)   │
  └──────┬──────┘ └─────┬──────┘ └──────┬──────┘
         │              │               │
         └──────────────┼───────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │ plato-fleet       │ (L4: Fleet)
              │ plato-transport   │
              └────────┬─────────┘
                       │
                       ▼
    ┌──────────────────────────────────────┐
    │          Execution (L5)              │
    │  ┌──────────────┐  ┌───────────────┐│
    │  │plato-workflow │  │ plato-sandbox ││
    │  └──────┬───────┘  └───────┬───────┘│
    │         │                  │        │
    │         └────────┬─────────┘        │
    │                  ▼                  │
    │          ┌──────────────┐           │
    │          │plato-contract│           │
    │          └──────────────┘           │
    └──────────────────┬──────────────────┘
                       │
                       ▼
    ┌──────────────────────────────────────┐
    │          State (L6)                  │
    │  ┌──────────────┐  ┌───────────────┐│
    │  │ plato-memory │  │plato-session   ││
    │  └──────────────┘  └───────────────┘│
    └──────────────────┬──────────────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ openconstruct-   │ (L7: Verification)
              │    mercury       │
              └──────────────────┘

    ┌──────────────────────────────────────┐
    │          Rendering (L8)              │
    │  ┌──────────────┐  ┌───────────────┐│
    │  │ a2ui-render  │  │a2ui-components││
    │  └──────┬───────┘  └───────────────┘│
    │  ┌──────────────┐  ┌───────────────┐│
    │  │ mud2scummvm  │  │a2ui-cave-wall ││
    │  └──────────────┘  └───────────────┘│
    └──────────────────────────────────────┘
```

Key dependency rules:
- **L0** depends on nothing (it's the root)
- **L1** senses depend on `plato-policy` for gating
- **L2** correlator depends on L1 shadow types
- **L3** comms depends on L2 events for routing
- **L4** fleet depends on L3 for inter-node messaging
- **L5** execution depends on L4 for delegation
- **L6** state is used by all layers above L0
- **L7** verification is a build-time/audit-time layer; it reads specs from all layers
- **L8** rendering sits at the edge, consuming component trees from any layer

---

## 12. Data Flow Examples

### 12.1 Flow: User Asks "What's in front of me?" (Vision Pipeline)

```
1. plato-shell: Input reader receives "look"
   → Parser produces Command { verb: Look, args: [], flags: {} }

2. plato-shell: Dispatcher routes to built-in look handler
   → look_handler() calls sense_router.get_shadows(SenseType::Vision)

3. plato-vision: Captures current frame
   → produces Shadow {
       sense_type: Vision,
       timestamp: now(),
       data: ShadowData::Visual(VisualData {
         text_regions: [TextRegion { text: "Whiteboard", bounds: (10,20,200,50), confidence: 0.95 }],
         objects: [DetectedObject { label: "whiteboard", bounds: (5,15,210,55), confidence: 0.92 }],
         dimensions: (1920, 1080),
         frame_hash: 0xABCD1234,
         thumbnail: Some([...])
       }),
       confidence: 0.93,
       source: "camera-0",
       spatial_ref: Some(SpatialRef { room: "office", position: (0.0, 1.5, 0.0) })
     }

4. plato-correlator: Receives shadow via broadcast channel
   → TemporalWindow buffers it
   → FusionRule "single_visual_look" fires with one shadow
   → Produces CorrelatedEvent {
       id: evt_001,
       source_shadows: [shadow_123],
       severity: Info,
       description: "Visual observation: Whiteboard detected at (5,15,210,55)",
       priority: Normal,
       ...
     }

5. plato-shell: Receives correlated event
   → Formats as CommandResult::Text("You see a whiteboard in front of you.")

6. a2ui-render: If target is terminal → ANSI formatted text
              If target is SCUMM → ScummScene with whiteboard object and hotspot
```

### 12.2 Flow: Agent Delegates a Task to Another Agent (Fleet Pipeline)

```
1. plato-workflow: Task "analyze_sentiment" reaches execution
   → TaskExecutor::execute() checks resource requirements
   → Determines local node is overloaded (cpu_percent: 0.92)

2. plato-fleet: Delegation requested
   → ResourceTracker queries all nodes:
     Node "alpha": cpu 92%, active_tasks 8/8 → SKIP
     Node "beta":  cpu 34%, active_tasks 2/8 → CANDIDATE
     Node "gamma": cpu 55%, active_tasks 4/8 → CANDIDATE
   → DelegationDecision { target: "beta", reason: LeastLoaded, estimated_load: 0.38 }

3. plato-contract: Before delegation, verify contract
   → Checks existing contract between local agent and agent on "beta"
   → Contract { state: Active, obligations: [Obligation { id: "compute", sla: Sla { availability: 0.99, max_latency: 5s } }] }
   → Contract is valid and within SLA

4. shell-mesh: Route task to "beta"
   → MeshMessage { source: "alpha", destination: Unicast("beta"), payload: encode(WorkflowTask { ... }) }
   → Star topology: alpha → hub → beta
   → Full mesh: alpha → beta (direct)

5. plato-sandbox (on "beta"): Execute the task
   → Sandbox { limits: ResourceLimits { max_memory_bytes: 512MB, max_cpu_time: 30s, ... } }
   → Executes sentiment analysis
   → SandboxResult { exit_code: 0, stdout: encode({ sentiment: "positive", score: 0.87 }) }

6. shell-mesh: Route result back
   → beta → alpha (reverse path)

7. plato-workflow: Receives result
   → TaskState transitions: Running → Completed
   → Downstream tasks in DAG are unblocked
   → If this was the last task: WorkflowState → Completed
```

### 12.3 Flow: Correlated Multi-Sense Event (Fusion Pipeline)

```
1. plato-vision: Captures frame showing person at door
   → Shadow { sense_type: Vision, data: Visual(VisualData {
       objects: [DetectedObject { label: "person", confidence: 0.89 }],
       text_regions: []
     }), confidence: 0.89, source: "camera-front-door" }

2. plato-sonar: Detects doorbell sound
   → Shadow { sense_type: Sonar, data: Audio(AudioData {
       transcript: None,
       sound_events: [SoundEvent { classification: "doorbell", onset_ms: 0, duration_ms: 800, confidence: 0.94 }],
       volume: 0.6
     }), confidence: 0.94, source: "mic-front-door" }

3. plato-correlator: Temporal windows for both Vision and Sonar contain new shadows
   → FusionRule "visitor_at_door" fires:
     required_senses: [Vision, Sonar]
     temporal_tolerance: 5 seconds
     fuse_fn: |shadows| {
       let person = shadows.iter().find(|s| s.sense_type == Vision)
         .and_then(|s| match &s.data { ShadowData::Visual(v) => v.objects.first(), _ => None });
       let doorbell = shadows.iter().find(|s| s.sense_type == Sonar)
         .and_then(|s| match &s.data { ShadowData::Audio(a) => a.sound_events.first(), _ => None });
       if person.is_some() && doorbell.is_some() {
         FusionResult::Event(CorrelatedEvent {
           severity: High,
           description: "Person detected at front door with doorbell ring",
           priority: Urgent,
           data: EventData::VisitorAlert { person_detected: true, doorbell: true },
           ...
         })
       } else { FusionResult::Pending }
     }

4. plato-tick: Correlated event dispatched to agent
   → TickMessage { to: Some(agent_id), message_type: Event { event_type: "visitor_alert" }, payload: {...} }

5. plato-shell: Agent receives notification
   → Outputs: "⚠️ Someone is at the front door (doorbell rang, person detected by camera)."

6. a2ui-render: For cave-wall dashboard → Widget renders alert with camera thumbnail
              For SCUMM → Shows character sprite at door with animation
              For terminal → Colored text with ⚠️ indicator
```

---

## Appendix A: Error Taxonomy

All errors across layers follow a common taxonomy:

```rust
pub enum OpenConstructError {
    // Layer 0
    Shell(ShellError),
    CommandParse { input: String, reason: String },
    ModuleLoad { module: String, reason: String },

    // Layer 1
    Sense(SenseError),
    PolicyDenied { sense: SenseType, reason: String },

    // Layer 2
    Fusion(FusionError),
    TemporalWindowOverflow { sense: SenseType, dropped: u64 },

    // Layer 3
    Communication(CommError),
    MeshUnreachable { node: NodeId },

    // Layer 4
    Fleet(FleetError),
    NodeUnregistered { node: NodeId },

    // Layer 5
    Workflow(WorkflowError),
    Sandbox(SandboxError),
    ContractBreached { contract: ContractId, reason: String },

    // Layer 6
    Memory(MemoryError),
    Session(SessionError),
    Config(ConfigError),

    // Layer 7
    Verification(VerificationError),

    // Layer 8
    Render(RenderError),
}
```

## Appendix B: Key Type Aliases

```rust
pub type AgentId = String;          // e.g., "agent://alpha/plato"
pub type NodeId = String;           // e.g., "node://home-server"
pub type SessionId = Uuid;
pub type RoomId = String;           // e.g., "room://home/office"
pub type EventId = Uuid;
pub type ShadowId = Uuid;
pub type MessageId = Uuid;
pub type WorkflowId = Uuid;
pub type TaskId = Uuid;
pub type ExecutionId = Uuid;
pub type ContractId = Uuid;
pub type MemoryId = Uuid;
pub type CheckpointId = Uuid;
pub type ComponentId = String;
pub type TraceId = String;
pub type SpanId = String;

pub type SenseType = String;        // "vision", "sonar", "manus", "dom"
pub type PolicyId = String;
pub type RuleId = String;
pub type EntityRef = String;
pub type AssetRef = String;
```

---

*This document covers every architectural layer of OpenConstruct. For installation and usage, see [README.md](README.md). For the public API surface, see [API.md](API.md). For design rationale, see [DESIGN.md](DESIGN.md).*
