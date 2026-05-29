# OpenConstruct Module Reference

> Complete API reference for every module in the OpenConstruct ecosystem.
> Last updated: 2026-05-29. Verify against source before relying on signatures.

---

## Table of Contents

- [Core Shell](#core-shell)
- [Senses](#senses)
- [Fusion & Communication](#fusion--communication)
- [Fleet](#fleet)
- [Execution](#execution)
- [State](#state)
- [Rooms & Knowledge](#rooms--knowledge)
- [Rendering](#rendering)
- [Bindings](#bindings)
- [Edge](#edge)
- [Verification](#verification)

---

## Core Shell

### plato-shell

Unified MUD shell — the primary REPL that agents and humans interact with.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-shell](https://github.com/openconstruct/plato-shell) |
| Tests | 247 |
| Dependencies | plato-session, plato-config, plato-transport, plato-policy, tokio, rustyline |

**Key Types**

```rust
pub struct Shell {
    session: SessionHandle,
    env: ShellEnv,
    aliases: HashMap<String, String>,
    history: Vec<CommandEntry>,
}

pub struct CommandEntry {
    pub input: String,
    pub parsed: ParsedCommand,
    pub exit_code: i32,
    pub timestamp: DateTime<Utc>,
}

pub struct ParsedCommand {
    pub verb: String,
    pub args: Vec<ShellArg>,
    pub redirects: Vec<Redirect>,
    pub pipe_chain: bool,
}

pub enum ShellArg {
    Literal(String),
    Variable(String),
    Glob(Pattern),
    Quoted(String),
}
```

**Main API**

```rust
impl Shell {
    pub fn new(config: &ShellConfig) -> Result<Self, ShellError>;
    pub async fn eval(&mut self, input: &str) -> Result<CommandResult, ShellError>;
    pub async fn eval_file(&mut self, path: &Path) -> Result<Vec<CommandResult>, ShellError>;
    pub fn register_command(&mut self, name: &str, handler: CommandFn) -> Result<(), ShellError>;
    pub fn set_variable(&mut self, key: &str, value: impl Into<ShellValue>);
    pub fn get_variable(&self, key: &str) -> Option<&ShellValue>;
    pub async fn complete(&self, partial: &str) -> Vec<Completion>;
    pub fn history(&self) -> &[CommandEntry];
    pub async fn shutdown(self) -> Result<(), ShellError>;
}

pub type CommandFn = Box<dyn Fn(&mut Shell, &[ShellArg]) -> Pin<Box<dyn Future<Output = Result<CommandResult, ShellError>> + Send>> + Send + Sync>;

pub struct CommandResult {
    pub stdout: String,
    pub stderr: String,
    pub exit_code: i32,
    pub side_effects: Vec<SideEffect>,
}
```

---

### plato-session

Session management — tracks authenticated sessions, their lifecycles, and attachment to agents.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-session](https://github.com/openconstruct/plato-session) |
| Tests | 134 |
| Dependencies | plato-config, uuid, tokio, serde, ring |

**Key Types**

```rust
pub struct Session {
    pub id: SessionId,
    pub agent_id: AgentId,
    pub created_at: DateTime<Utc>,
    pub last_active: DateTime<Utc>,
    pub state: SessionState,
    pub capabilities: HashSet<Capability>,
}

pub enum SessionState {
    Init,
    Authenticated,
    Active,
    Suspended,
    Terminated,
}

pub struct SessionId(Uuid);
pub struct AgentId(String);
```

**Main API**

```rust
pub struct SessionManager {
    /* internal pool */
}

impl SessionManager {
    pub fn new(config: SessionConfig) -> Result<Self, SessionError>;
    pub async fn create(&self, agent_id: &AgentId, caps: HashSet<Capability>) -> Result<Session, SessionError>;
    pub async fn get(&self, id: &SessionId) -> Result<Option<Session>, SessionError>;
    pub async fn touch(&self, id: &SessionId) -> Result<(), SessionError>;
    pub async fn suspend(&self, id: &SessionId) -> Result<(), SessionError>;
    pub async fn resume(&self, id: &SessionId) -> Result<Session, SessionError>;
    pub async fn terminate(&self, id: &SessionId) -> Result<(), SessionError>;
    pub async fn list_active(&self) -> Vec<Session>;
    pub async fn gc(&self, max_idle: Duration) -> usize;
}
```

---

### plato-config

Configuration — hierarchical, validated, hot-reloadable config for the entire stack.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-config](https://github.com/openconstruct/plato-config) |
| Tests | 98 |
| Dependencies | serde, toml, json5, notify (fs-watcher), validate_derive |

**Key Types**

```rust
pub struct Config {
    pub shell: ShellConfig,
    pub senses: SensesConfig,
    pub fleet: FleetConfig,
    pub memory: MemoryConfig,
    pub policy: PolicyConfig,
    pub observe: ObserveConfig,
    /* ... */
}

pub struct ConfigHandle(Arc<RwLock<Config>>);
```

**Main API**

```rust
pub fn load(path: &Path) -> Result<Config, ConfigError>;
pub fn load_str(toml_or_json: &str) -> Result<Config, ConfigError>;
pub fn validate(config: &Config) -> Result<(), Vec<ValidationError>>;
pub fn watch(path: &Path) -> Result<ConfigWatch, ConfigError>;

impl ConfigHandle {
    pub fn read(&self) -> RwLockReadGuard<'_, Config>;
    pub async fn write(&self) -> RwLockWriteGuard<'_, Config>;
    pub fn on_change(&self, cb: Box<dyn Fn(&Config) + Send + Sync>);
}
```

---

## Senses

### plato-vision

Camera-to-text sense — captures frames, runs detection/OCR/description, emits structured observations.

| Field | Value |
|---|---|
| Language | Rust (core), Python (model bridge) |
| Repo | [github.com/openconstruct/plato-vision](https://github.com/openconstruct/plato-vision) |
| Tests | 182 |
| Dependencies | plato-transport, opencv-rust, tract (onnx), tokio, serde |

**Key Types**

```rust
pub struct VisionPipeline {
    pub sources: Vec<CameraSource>,
    pub models: Vec<ModelRef>,
    pub config: VisionConfig,
}

pub struct Observation {
    pub timestamp: DateTime<Utc>,
    pub source: CameraId,
    pub detections: Vec<Detection>,
    pub description: Option<String>,
    pub ocr_text: Option<Vec<TextBlock>>,
    pub frame_hash: u64,
}

pub struct Detection {
    pub label: String,
    pub confidence: f32,
    pub bbox: BBox,
    pub attrs: HashMap<String, String>,
}

pub struct TextBlock {
    pub text: String,
    pub bbox: BBox,
    pub confidence: f32,
}
```

**Main API**

```rust
impl VisionPipeline {
    pub fn new(config: VisionConfig) -> Result<Self, VisionError>;
    pub async fn capture(&self, source: &CameraId) -> Result<Frame, VisionError>;
    pub async fn analyze(&self, frame: &Frame) -> Result<Observation, VisionError>;
    pub async fn stream(&self, source: &CameraId, interval: Duration) -> impl Stream<Item = Observation>;
    pub fn register_model(&mut self, model: ModelRef) -> Result<(), VisionError>;
    pub async fn shutdown(self) -> Result<(), VisionError>;
}
```

---

### plato-sonar-text

Acoustic-to-text sense — processes audio streams into transcriptions, speaker diarization, and sound-event labels.

| Field | Value |
|---|---|
| Language | Rust (core), Python (whisper bridge) |
| Repo | [github.com/openconstruct/plato-sonar-text](https://github.com/openconstruct/plato-sonar-text) |
| Tests | 96 |
| Dependencies | plato-transport, cpal, whisper-rs, tokio, serde |

**Key Types**

```rust
pub struct SonarPipeline {
    pub sources: Vec<AudioSource>,
    pub config: SonarConfig,
}

pub struct Transcript {
    pub timestamp: DateTime<Utc>,
    pub segments: Vec<Segment>,
    pub language: String,
    pub source: AudioSourceId,
}

pub struct Segment {
    pub speaker: Option<String>,
    pub text: String,
    pub start_ms: u64,
    pub end_ms: u64,
    pub confidence: f32,
}
```

**Main API**

```rust
impl SonarPipeline {
    pub fn new(config: SonarConfig) -> Result<Self, SonarError>;
    pub async fn transcribe(&self, audio: &[u8], fmt: AudioFormat) -> Result<Transcript, SonarError>;
    pub async fn stream(&self, source: &AudioSourceId) -> impl Stream<Item = Transcript>;
    pub async fn detect_events(&self, audio: &[u8]) -> Vec<SoundEvent>;
}
```

---

### plato-manus

Hands — file I/O, API calls, device control. The "manipulation" sense for interacting with the outside world.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-manus](https://github.com/openconstruct/plato-manus) |
| Tests | 210 |
| Dependencies | plato-transport, reqwest, ssh2, serde, tokio, notify |

**Key Types**

```rust
pub struct ManusHandle {
    pub capabilities: Vec<ManusCapability>,
}

pub enum ManusCapability {
    Filesystem { root: PathBuf, writable: bool },
    Http { allowed_hosts: Vec<String> },
    Ssh { hosts: Vec<SshTarget> },
    Device { device_type: DeviceType, path: String },
}

pub struct FileOp {
    pub path: PathBuf,
    pub op: FileOpKind,
}

pub enum FileOpKind {
    Read, Write(Vec<u8>), Append(Vec<u8>), Delete, Mkdir, List,
}
```

**Main API**

```rust
impl ManusHandle {
    pub async fn file_read(&self, path: &Path) -> Result<Vec<u8>, ManusError>;
    pub async fn file_write(&self, path: &Path, data: &[u8]) -> Result<(), ManusError>;
    pub async fn file_list(&self, dir: &Path) -> Result<Vec<DirEntry>, ManusError>;
    pub async fn http_get(&self, url: &str) -> Result<HttpResponse, ManusError>;
    pub async fn http_post(&self, url: &str, body: &[u8]) -> Result<HttpResponse, ManusError>;
    pub async fn ssh_exec(&self, host: &str, cmd: &str) -> Result<SshResult, ManusError>;
    pub async fn device_cmd(&self, device: &str, cmd: &str) -> Result<DeviceResponse, ManusError>;
    pub fn capabilities(&self) -> &[ManusCapability];
}
```

---

### plato-playwright

Browser automation — controls Chromium via the Playwright protocol for web-based tasks.

| Field | Value |
|---|---|
| Language | Rust (via playwright-core binding) |
| Repo | [github.com/openconstruct/plato-playwright](https://github.com/openconstruct/plato-playwright) |
| Tests | 88 |
| Dependencies | plato-transport, playwright-core (npm bridge), tokio, serde, base64 |

**Key Types**

```rust
pub struct Browser {
    /* internal playwright instance */
}

pub struct Page {
    pub id: PageId,
    pub url: String,
    pub title: String,
}

pub struct ElementHandle { /* opaque */ }
```

**Main API**

```rust
impl Browser {
    pub async fn launch(config: BrowserConfig) -> Result<Self, BrowserError>;
    pub async fn new_page(&self) -> Result<Page, BrowserError>;
    pub async fn pages(&self) -> Vec<Page>;
    pub async fn shutdown(self) -> Result<(), BrowserError>;
}

impl Page {
    pub async fn goto(&self, url: &str) -> Result<(), BrowserError>;
    pub async fn click(&self, selector: &str) -> Result<(), BrowserError>;
    pub async fn fill(&self, selector: &str, value: &str) -> Result<(), BrowserError>;
    pub async fn text_content(&self, selector: &str) -> Result<Option<String>, BrowserError>;
    pub async fn inner_html(&self, selector: &str) -> Result<String, BrowserError>;
    pub async fn screenshot(&self) -> Result<Vec<u8>, BrowserError>;
    pub async fn wait_for_selector(&self, selector: &str, timeout: Duration) -> Result<ElementHandle, BrowserError>;
    pub async fn evaluate(&self, expr: &str) -> Result<serde_json::Value, BrowserError>;
    pub async fn pdf(&self) -> Result<Vec<u8>, BrowserError>;
}
```

---

### plato-puppeteer

Desktop-to-MUD bridge — captures desktop state and projects it into the MUD world model.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-puppeteer](https://github.com/openconstruct/plato-puppeteer) |
| Tests | 64 |
| Dependencies | plato-transport, plato-vision, xcap (screenshots), enigo (input), tokio |

**Key Types**

```rust
pub struct DesktopBridge {
    pub screens: Vec<ScreenInfo>,
}

pub struct DesktopSnapshot {
    pub screen: ScreenId,
    pub image: Vec<u8>,
    pub active_window: Option<WindowInfo>,
    pub timestamp: DateTime<Utc>,
}
```

**Main API**

```rust
impl DesktopBridge {
    pub fn new() -> Result<Self, PuppeteerError>;
    pub async fn capture(&self, screen: Option<ScreenId>) -> Result<DesktopSnapshot, PuppeteerError>;
    pub async fn key_press(&self, key: KeyCode, mods: Modifiers) -> Result<(), PuppeteerError>;
    pub async fn mouse_move(&self, x: i32, y: i32) -> Result<(), PuppeteerError>;
    pub async fn mouse_click(&self, button: MouseButton) -> Result<(), PuppeteerError>;
    pub async fn type_text(&self, text: &str) -> Result<(), PuppeteerError>;
    pub fn screens(&self) -> &[ScreenInfo];
}
```

---

## Fusion & Communication

### plato-correlator

Cross-sense fusion — merges observations from multiple senses into unified situational awareness.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-correlator](https://github.com/openconstruct/plato-correlator) |
| Tests | 143 |
| Dependencies | plato-transport, plato-vision, plato-sonar-text, petgraph, tokio, serde |

**Key Types**

```rust
pub struct Correlator {
    pub window: Duration,
    pub graph: FusionGraph,
}

pub struct FusedEvent {
    pub id: EventId,
    pub timestamp: DateTime<Utc>,
    pub sources: Vec<SenseObservation>,
    pub correlation_score: f64,
    pub summary: String,
    pub entities: Vec<Entity>,
}

pub struct Entity {
    pub id: EntityId,
    pub kind: EntityKind,
    pub mentions: Vec<SenseObservation>,
    pub attributes: HashMap<String, String>,
}

pub enum EntityKind { Person, Object, Location, Event, Concept }
```

**Main API**

```rust
impl Correlator {
    pub fn new(config: CorrelatorConfig) -> Result<Self, CorrelatorError>;
    pub async fn ingest(&self, obs: SenseObservation) -> Result<(), CorrelatorError>;
    pub async fn query(&self, q: &CorrelationQuery) -> Vec<FusedEvent>;
    pub async fn entities(&self, filter: Option<EntityFilter>) -> Vec<Entity>;
    pub async fn recent(&self, window: Duration) -> Vec<FusedEvent>;
    pub fn subscribe(&self) -> impl Stream<Item = FusedEvent>;
}
```

---

### plato-tick

Inter-agent messaging — lightweight pub/sub with topic routing, priority queues, and delivery guarantees.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-tick](https://github.com/openconstruct/plato-tick) |
| Tests | 176 |
| Dependencies | plato-transport, tokio, serde, dashmap, priority-queue |

**Key Types**

```rust
pub struct TickBus {
    /* internal router */
}

pub struct Tick {
    pub id: TickId,
    pub topic: Topic,
    pub sender: AgentId,
    pub payload: Vec<u8>,
    pub priority: Priority,
    pub timestamp: DateTime<Utc>,
    pub ttl: Option<Duration>,
}

pub struct Subscription {
    pub id: SubId,
    pub topic: Topic,
    pub filter: Option<TickFilter>,
}
```

**Main API**

```rust
impl TickBus {
    pub fn new(config: TickConfig) -> Result<Self, TickError>;
    pub async fn publish(&self, tick: Tick) -> Result<(), TickError>;
    pub async fn subscribe(&self, topic: &Topic, filter: Option<TickFilter>) -> Result<Subscription, TickError>;
    pub async fn unsubscribe(&self, sub: SubId) -> Result<(), TickError>;
    pub fn recv(&self, sub: &Subscription) -> impl Stream<Item = Tick>;
    pub async fn request_response(&self, tick: Tick, timeout: Duration) -> Result<Tick, TickError>;
    pub fn topic_list(&self) -> Vec<Topic>;
    pub fn subscriber_count(&self, topic: &Topic) -> usize;
}
```

---

### plato-a2a

Agent-to-agent protocol — implements the A2A specification for cross-agent discovery, capability negotiation, and task delegation.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-a2a](https://github.com/openconstruct/plato-a2a) |
| Tests | 112 |
| Dependencies | plato-transport, plato-tick, tokio, serde, json-schema-validator |

**Key Types**

```rust
pub struct A2aAgent {
    pub id: AgentId,
    pub name: String,
    pub capabilities: Vec<AgentCapability>,
    pub endpoint: Url,
    pub authentication: Option<AuthConfig>,
}

pub struct AgentCapability {
    pub name: String,
    pub description: String,
    pub input_schema: serde_json::Value,
    pub output_schema: serde_json::Value,
}

pub struct TaskRequest {
    pub id: TaskId,
    pub capability: String,
    pub input: serde_json::Value,
    pub deadline: Option<DateTime<Utc>>,
}

pub struct TaskResponse {
    pub id: TaskId,
    pub status: TaskStatus,
    pub output: Option<serde_json::Value>,
    pub error: Option<String>,
}
```

**Main API**

```rust
pub struct A2aClient { /* ... */ }

impl A2aClient {
    pub fn new(endpoint: Url) -> Result<Self, A2aError>;
    pub async fn discover(&self) -> Result<Vec<A2aAgent>, A2aError>;
    pub async fn get_agent(&self, id: &AgentId) -> Result<A2aAgent, A2aError>;
    pub async fn submit_task(&self, agent: &AgentId, request: TaskRequest) -> Result<TaskResponse, A2aError>;
    pub async fn cancel_task(&self, task_id: &TaskId) -> Result<(), A2aError>;
    pub async fn task_status(&self, task_id: &TaskId) -> Result<TaskStatus, A2aError>;
}

pub struct A2aServer { /* ... */ }

impl A2aServer {
    pub fn new(agent: A2aAgent, handler: Box<dyn TaskHandler>) -> Result<Self, A2aError>;
    pub async fn serve(&self, port: u16) -> Result<(), A2aError>;
}
```

---

### shell-mesh

Mesh networking — provides encrypted, NAT-traversing peer-to-peer connectivity between OpenConstruct nodes.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/shell-mesh](https://github.com/openconstruct/shell-mesh) |
| Tests | 158 |
| Dependencies | plato-transport, noise-protocol, libp2p, tokio, serde, ed25519-dalek |

**Key Types**

```rust
pub struct MeshNode {
    pub peer_id: PeerId,
    pub listen_addrs: Vec<Multiaddr>,
    pub connected_peers: HashSet<PeerId>,
}

pub struct MeshConfig {
    pub bootstrap_peers: Vec<Multiaddr>,
    pub listen_port: u16,
    pub encryption: EncryptionConfig,
    pub relay: Option<RelayConfig>,
}
```

**Main API**

```rust
impl MeshNode {
    pub async fn new(config: MeshConfig) -> Result<Self, MeshError>;
    pub async fn bootstrap(&mut self) -> Result<(), MeshError>;
    pub async fn dial(&mut self, addr: &Multiaddr) -> Result<(), MeshError>;
    pub async fn broadcast(&self, data: &[u8]) -> Result<(), MeshError>;
    pub async fn send(&self, peer: &PeerId, data: &[u8]) -> Result<(), MeshError>;
    pub fn recv(&self) -> impl Stream<Item = (PeerId, Vec<u8>)>;
    pub fn connected_peers(&self) -> &HashSet<PeerId>;
    pub async fn leave(self) -> Result<(), MeshError>;
}
```

---

## Fleet

### plato-fleet

Fleet discovery and room management — tracks available agents, their rooms, and provides service discovery.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-fleet](https://github.com/openconstruct/plato-fleet) |
| Tests | 124 |
| Dependencies | plato-transport, plato-a2a, shell-mesh, tokio, serde, dashmap |

**Key Types**

```rust
pub struct FleetRegistry {
    /* internal state */
}

pub struct FleetNode {
    pub id: NodeId,
    pub agent: A2aAgent,
    pub rooms: Vec<RoomId>,
    pub status: NodeStatus,
    pub last_heartbeat: DateTime<Utc>,
    pub address: Multiaddr,
}

pub enum NodeStatus { Online, Busy, Offline, Draining }
```

**Main API**

```rust
impl FleetRegistry {
    pub fn new(config: FleetConfig) -> Result<Self, FleetError>;
    pub async fn register(&self, node: FleetNode) -> Result<(), FleetError>;
    pub async fn deregister(&self, id: &NodeId) -> Result<(), FleetError>;
    pub async fn heartbeat(&self, id: &NodeId) -> Result<(), FleetError>;
    pub async fn discover(&self, capability: &str) -> Vec<FleetNode>;
    pub async fn list(&self) -> Vec<FleetNode>;
    pub async fn get(&self, id: &NodeId) -> Option<FleetNode>;
    pub async fn assign_room(&self, node: &NodeId, room: RoomId) -> Result<(), FleetError>;
    pub async fn drain(&self, id: &NodeId) -> Result<(), FleetError>;
}
```

---

### plato-transport

IPC/TCP abstraction — unified transport layer used by all other modules for message passing.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-transport](https://github.com/openconstruct/plato-transport) |
| Tests | 203 |
| Dependencies | tokio, serde, bytes, flume, ring (encryption) |

**Key Types**

```rust
pub enum Transport {
    Ipc(IpcTransport),
    Tcp(TcpTransport),
    Memory(MemTransport),
}

pub struct Envelope {
    pub src: AgentId,
    pub dst: AgentId,
    pub channel: String,
    pub payload: Vec<u8>,
    pub headers: HashMap<String, String>,
    pub trace_id: TraceId,
}

pub struct TransportConfig {
    pub mode: TransportMode,
    pub tcp_listen: Option<SocketAddr>,
    pub ipc_path: Option<PathBuf>,
    pub encryption: Option<EncryptionConfig>,
    pub max_message_size: usize,
}
```

**Main API**

```rust
pub fn create_transport(config: TransportConfig) -> Result<Transport, TransportError>;

impl Transport {
    pub async fn send(&self, envelope: Envelope) -> Result<(), TransportError>;
    pub async fn recv(&self) -> Result<Envelope, TransportError>;
    pub fn subscribe(&self, channel: &str) -> impl Stream<Item = Envelope>;
    pub async fn bind(&mut self) -> Result<(), TransportError>;
    pub async fn connect(&mut self, addr: &str) -> Result<(), TransportError>;
    pub async fn close(&mut self) -> Result<(), TransportError>;
}
```

---

## Execution

### plato-workflow

DAG orchestration — define, validate, and execute directed acyclic graphs of tasks.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-workflow](https://github.com/openconstruct/plato-workflow) |
| Tests | 167 |
| Dependencies | plato-transport, plato-sandbox, petgraph, tokio, serde, chrono |

**Key Types**

```rust
pub struct Workflow {
    pub id: WorkflowId,
    pub nodes: HashMap<NodeId, WorkflowNode>,
    pub edges: Vec<Edge>,
    pub entry: NodeId,
}

pub struct WorkflowNode {
    pub id: NodeId,
    pub label: String,
    pub task: TaskDef,
    pub retry: RetryPolicy,
    pub timeout: Option<Duration>,
}

pub struct TaskDef {
    pub agent: Option<AgentId>,
    pub capability: String,
    pub input: serde_json::Value,
}

pub struct Edge {
    pub from: NodeId,
    pub to: NodeId,
    pub condition: Option<EdgeCondition>,
}

pub enum EdgeCondition {
    Success,
    Failure,
    OutputEquals(String),
    Custom(String),
}
```

**Main API**

```rust
pub struct WorkflowEngine { /* ... */ }

impl WorkflowEngine {
    pub fn new(transport: Transport, sandbox: Sandbox) -> Result<Self, WorkflowError>;
    pub fn validate(&self, wf: &Workflow) -> Result<(), Vec<WorkflowError>>;
    pub async fn submit(&self, wf: Workflow) -> Result<WorkflowRun, WorkflowError>;
    pub async fn cancel(&self, run_id: &WorkflowRunId) -> Result<(), WorkflowError>;
    pub async fn status(&self, run_id: &WorkflowRunId) -> Result<WorkflowRunStatus, WorkflowError>;
    pub fn subscribe(&self, run_id: &WorkflowRunId) -> impl Stream<Item = WorkflowEvent>;
}

pub struct WorkflowRun {
    pub id: WorkflowRunId,
    pub workflow_id: WorkflowId,
    pub status: WorkflowRunStatus,
    pub node_states: HashMap<NodeId, NodeRunState>,
    pub started_at: DateTime<Utc>,
    pub finished_at: Option<DateTime<Utc>>,
}
```

---

### plato-sandbox

Execution sandbox — isolated environments for running untrusted or third-party code.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-sandbox](https://github.com/openconstruct/plato-sandbox) |
| Tests | 91 |
| Dependencies | plato-config, tokio, wasmtime (Wasm), container runtime (OCI), serde |

**Key Types**

```rust
pub struct Sandbox {
    pub runtime: SandboxRuntime,
    pub limits: ResourceLimits,
}

pub enum SandboxRuntime {
    Wasm { engine: wasmtime::Engine },
    Container { runtime: ContainerRuntime },
    Process { chroot: Option<PathBuf> },
}

pub struct ResourceLimits {
    pub max_memory_bytes: u64,
    pub max_cpu_ms: u64,
    pub max_file_size_bytes: u64,
    pub network: bool,
    pub timeout: Duration,
}

pub struct SandboxResult {
    pub exit_code: i32,
    pub stdout: Vec<u8>,
    pub stderr: Vec<u8>,
    pub duration: Duration,
    pub memory_peak: u64,
}
```

**Main API**

```rust
impl Sandbox {
    pub fn new(config: SandboxConfig) -> Result<Self, SandboxError>;
    pub async fn execute(&self, program: &[u8], args: &[&str], input: &[u8]) -> Result<SandboxResult, SandboxError>;
    pub async fn execute_wasm(&self, wasm: &[u8], func: &str, params: &[WasmValue]) -> Result<Vec<WasmValue>, SandboxError>;
    pub fn check_limits(&self, result: &SandboxResult) -> Result<(), SandboxError>;
}
```

---

### plato-contract

Inter-agent contracts — typed, versioned agreements between agents specifying input/output schemas, SLAs, and penalties.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-contract](https://github.com/openconstruct/plato-contract) |
| Tests | 78 |
| Dependencies | plato-transport, serde, json-schema-validator, semver, chrono |

**Key Types**

```rust
pub struct AgentContract {
    pub id: ContractId,
    pub version: semver::Version,
    pub provider: AgentId,
    pub consumer: AgentId,
    pub interface: ContractInterface,
    pub sla: SlaSpec,
    pub created_at: DateTime<Utc>,
    pub expires_at: Option<DateTime<Utc>>,
}

pub struct ContractInterface {
    pub inputs: serde_json::Value,
    pub outputs: serde_json::Value,
    pub errors: Vec<ErrorDef>,
}

pub struct SlaSpec {
    pub max_latency_ms: u64,
    pub availability_percent: f64,
    pub max_error_rate: f64,
}
```

**Main API**

```rust
pub struct ContractStore { /* ... */ }

impl ContractStore {
    pub fn new() -> Result<Self, ContractError>;
    pub async fn register(&self, contract: AgentContract) -> Result<(), ContractError>;
    pub async fn get(&self, id: &ContractId) -> Result<AgentContract, ContractError>;
    pub async fn validate_call(&self, contract: &ContractId, input: &serde_json::Value) -> Result<(), ContractError>;
    pub async fn validate_response(&self, contract: &ContractId, output: &serde_json::Value) -> Result<(), ContractError>;
    pub async fn list_for_agent(&self, agent: &AgentId) -> Vec<AgentContract>;
    pub async fn revoke(&self, id: &ContractId) -> Result<(), ContractError>;
}
```

---

## State

### plato-memory

Agent memory — persistent, queryable memory store with recency/relevance scoring and episodic/semantic layers.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-memory](https://github.com/openconstruct/plato-memory) |
| Tests | 154 |
| Dependencies | plato-transport, sled (embedded DB), serde, chrono, tokenizers (embedding) |

**Key Types**

```rust
pub struct MemoryStore {
    /* internal stores */
}

pub struct MemoryEntry {
    pub id: MemoryId,
    pub agent_id: AgentId,
    pub content: String,
    pub embedding: Option<Vec<f32>>,
    pub metadata: HashMap<String, String>,
    pub created_at: DateTime<Utc>,
    pub accessed_at: DateTime<Utc>,
    pub access_count: u32,
    pub importance: f32,
    pub memory_type: MemoryType,
}

pub enum MemoryType {
    Episodic,
    Semantic,
    Procedural,
    Working,
}
```

**Main API**

```rust
impl MemoryStore {
    pub fn new(config: MemoryConfig) -> Result<Self, MemoryError>;
    pub async fn store(&self, entry: MemoryEntry) -> Result<MemoryId, MemoryError>;
    pub async fn recall(&self, id: &MemoryId) -> Result<MemoryEntry, MemoryError>;
    pub async fn search(&self, query: &str, limit: usize) -> Vec<MemoryEntry>;
    pub async fn search_similar(&self, embedding: &[f32], threshold: f32, limit: usize) -> Vec<MemoryEntry>;
    pub async fn recent(&self, agent: &AgentId, limit: usize) -> Vec<MemoryEntry>;
    pub async fn forget(&self, id: &MemoryId) -> Result<(), MemoryError>;
    pub async fn consolidate(&self, agent: &AgentId) -> Result<ConsolidationReport, MemoryError>;
    pub async fn gc(&self, max_age: Duration) -> usize;
}
```

---

### plato-policy

Policy engine — evaluates allow/deny rules for every action an agent attempts.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-policy](https://github.com/openconstruct/plato-policy) |
| Tests | 189 |
| Dependencies | serde, regorus (rego engine), chrono, once_cell |

**Key Types**

```rust
pub struct PolicyEngine {
    /* loaded policy sets */
}

pub struct PolicyRequest {
    pub agent: AgentId,
    pub action: String,
    pub resource: String,
    pub context: HashMap<String, serde_json::Value>,
}

pub struct PolicyDecision {
    pub allowed: bool,
    pub reason: Option<String>,
    pub obligations: Vec<String>,
    pub matched_rules: Vec<String>,
}
```

**Main API**

```rust
impl PolicyEngine {
    pub fn new(config: PolicyConfig) -> Result<Self, PolicyError>;
    pub fn load_policy(&mut self, source: PolicySource) -> Result<(), PolicyError>;
    pub fn evaluate(&self, request: PolicyRequest) -> PolicyDecision;
    pub fn explain(&self, request: PolicyRequest) -> PolicyExplanation;
    pub fn list_policies(&self) -> Vec<PolicyMeta>;
}

pub enum PolicySource {
    Rego(String),
    Json(serde_json::Value),
    File(PathBuf),
}
```

---

### plato-observe

Metrics, tracing, and observability — collects and exports telemetry for the entire OpenConstruct stack.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-observe](https://github.com/openconstruct/plato-observe) |
| Tests | 106 |
| Dependencies | plato-transport, metrics, metrics-exporter-prometheus, tracing, tracing-subscriber, opentelemetry, tokio |

**Key Types**

```rust
pub struct ObserveConfig {
    pub metrics_addr: SocketAddr,
    pub tracing_enabled: bool,
    pub otlp_endpoint: Option<String>,
    pub service_name: String,
    pub sample_rate: f64,
}

pub struct Span {
    pub trace_id: TraceId,
    pub span_id: SpanId,
    pub operation: String,
    pub start: DateTime<Utc>,
    pub duration: Option<Duration>,
    pub attributes: HashMap<String, String>,
    pub status: SpanStatus,
}
```

**Main API**

```rust
pub fn init_observability(config: &ObserveConfig) -> Result<ObserveGuard, ObserveError>;
pub fn meter() -> Meter;
pub fn tracer() -> Tracer;

// Convenience macros
pub macro observe_counter { ... }
pub macro observe_histogram { ... }
pub macro observe_span { ... }
```

---

## Rooms & Knowledge

### plato-room

Knowledge graphs — represents a "room" as a connected knowledge graph with typed nodes and relations.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-room](https://github.com/openconstruct/plato-room) |
| Tests | 132 |
| Dependencies | plato-transport, petgraph, serde, chrono, uuid |

**Key Types**

```rust
pub struct Room {
    pub id: RoomId,
    pub graph: KnowledgeGraph,
    pub metadata: RoomMetadata,
}

pub struct KnowledgeGraph {
    /* petgraph::DiGraph<KnowledgeNode, KnowledgeEdge> */
}

pub struct KnowledgeNode {
    pub id: NodeId,
    pub label: String,
    pub kind: NodeKind,
    pub properties: HashMap<String, serde_json::Value>,
}

pub struct KnowledgeEdge {
    pub relation: String,
    pub weight: f64,
    pub properties: HashMap<String, serde_json::Value>,
}

pub enum NodeKind {
    Entity, Concept, Document, Agent, Event, Location,
}
```

**Main API**

```rust
impl Room {
    pub fn new(id: RoomId) -> Self;
    pub fn add_node(&mut self, node: KnowledgeNode) -> NodeId;
    pub fn add_edge(&mut self, from: NodeId, to: NodeId, edge: KnowledgeEdge) -> Result<(), RoomError>;
    pub fn get_node(&self, id: &NodeId) -> Option<&KnowledgeNode>;
    pub fn query(&self, pattern: &QueryPattern) -> Vec<&KnowledgeNode>;
    pub fn neighbors(&self, id: &NodeId) -> Vec<(&KnowledgeEdge, &KnowledgeNode)>;
    pub fn shortest_path(&self, from: &NodeId, to: &NodeId) -> Option<Vec<NodeId>>;
    pub fn subgraph(&self, roots: &[NodeId], depth: usize) -> KnowledgeGraph;
    pub fn merge(&mut self, other: &Room) -> Result<MergeReport, RoomError>;
    pub fn node_count(&self) -> usize;
    pub fn edge_count(&self) -> usize;
}
```

---

### plato-loader

Room loading — deserializes room definitions from YAML/JSON/Markdown into live `Room` objects.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-loader](https://github.com/openconstruct/plato-loader) |
| Tests | 87 |
| Dependencies | plato-room, serde, serde_yaml, serde_json, pulldown-cmark, walkdir |

**Key Types**

```rust
pub struct RoomLoader {
    pub format_priority: Vec<RoomFormat>,
}

pub enum RoomFormat { Yaml, Json, Markdown, Toml }
```

**Main API**

```rust
impl RoomLoader {
    pub fn new() -> Self;
    pub fn load_file(&self, path: &Path) -> Result<Room, LoaderError>;
    pub fn load_dir(&self, dir: &Path) -> Result<Vec<Room>, LoaderError>;
    pub fn load_str(&self, content: &str, format: RoomFormat) -> Result<Room, LoaderError>;
    pub fn watch_dir(&self, dir: &Path) -> impl Stream<Item = Result<Room, LoaderError>>;
}
```

---

### plato-uidl

Interface Description Language — declares agent-facing interfaces (forms, menus, commands) in a declarative format.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/plato-uidl](https://github.com/openconstruct/plato-uidl) |
| Tests | 73 |
| Dependencies | serde, serde_json, json-schema-validator, chrono |

**Key Types**

```rust
pub struct UidlInterface {
    pub id: InterfaceId,
    pub version: semver::Version,
    pub title: String,
    pub commands: Vec<UidlCommand>,
    pub forms: Vec<UidlForm>,
    pub menus: Vec<UidlMenu>,
}

pub struct UidlCommand {
    pub name: String,
    pub description: String,
    pub params: Vec<UidlParam>,
    pub returns: UidlType,
}

pub struct UidlForm {
    pub id: String,
    pub fields: Vec<UidlField>,
    pub submit_label: String,
}

pub struct UidlMenu {
    pub id: String,
    pub items: Vec<UidlMenuItem>,
}
```

**Main API**

```rust
pub fn parse(source: &str) -> Result<UidlInterface, UidlError>;
pub fn validate(uidl: &UidlInterface) -> Result<(), Vec<UidlError>>;
pub fn to_json_schema(uidl: &UidlInterface) -> serde_json::Value;
pub fn from_json_schema(schema: &serde_json::Value) -> Result<UidlInterface, UidlError>;
```

---

## Rendering

### a2ui-render

Text-to-UI renderer — converts agent output (text, data, commands) into platform-specific UI payloads.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/a2ui-render](https://github.com/openconstruct/a2ui-render) |
| Tests | 145 |
| Dependencies | plato-uidl, serde, handlebars (templates), tokio |

**Key Types**

```rust
pub struct Renderer {
    pub templates: TemplateStore,
    pub platform: RenderPlatform,
}

pub enum RenderPlatform {
    Terminal,
    Web,
    Discord,
    Slack,
    Telegram,
    Native,
}

pub struct RenderOutput {
    pub platform: RenderPlatform,
    pub content: RenderedContent,
    pub attachments: Vec<Attachment>,
}
```

**Main API**

```rust
impl Renderer {
    pub fn new(platform: RenderPlatform, config: RenderConfig) -> Result<Self, RenderError>;
    pub fn render(&self, input: &AgentOutput, template: Option<&str>) -> Result<RenderOutput, RenderError>;
    pub fn render_uidl(&self, uidl: &UidlInterface) -> Result<RenderOutput, RenderError>;
    pub fn register_template(&mut self, name: &str, template: &str) -> Result<(), RenderError>;
}
```

---

### a2ui-components

UI components — a shared library of composable, platform-adaptive UI components used by `a2ui-render`.

| Field | Value |
|---|---|
| Language | Rust (core types), TypeScript (web components) |
| Repo | [github.com/openconstruct/a2ui-components](https://github.com/openconstruct/a2ui-components) |
| Tests | 194 |
| Dependencies | a2ui-render, serde, wasm-bindgen, web-sys |

**Key Types**

```rust
pub enum Component {
    Text(TextComponent),
    List(ListComponent),
    Table(TableComponent),
    Form(FormComponent),
    CodeBlock(CodeComponent),
    Image(ImageComponent),
    Card(CardComponent),
    Modal(ModalComponent),
    Navigation(NavComponent),
}

pub struct ComponentProps {
    pub id: Option<String>,
    pub class: Vec<String>,
    pub style: HashMap<String, String>,
    pub aria: HashMap<String, String>,
    pub data: HashMap<String, serde_json::Value>,
}
```

**Main API**

```rust
pub fn component_to_html(comp: &Component) -> String;
pub fn component_to_markdown(comp: &Component) -> String;
pub fn component_to_terminal(comp: &Component, width: usize) -> String;
pub fn component_to_discord(comp: &Component) -> String;
pub fn merge_components(base: &Component, overlay: &Component) -> Result<Component, ComponentError>;
```

---

### a2ui-cave-wall

Cave wall dashboard — a real-time dashboard that renders the MUD's "cave wall" metaphor for monitoring and control.

| Field | Value |
|---|---|
| Language | Rust (backend), TypeScript (frontend) |
| Repo | [github.com/openconstruct/a2ui-cave-wall](https://github.com/openconstruct/a2ui-cave-wall) |
| Tests | 67 |
| Dependencies | a2ui-render, a2ui-components, axum (HTTP), tokio-tungstenite (WS), serde |

**Key Types**

```rust
pub struct CaveWall {
    pub panels: Vec<Panel>,
    pub layout: Layout,
}

pub struct Panel {
    pub id: String,
    pub title: String,
    pub source: PanelSource,
    pub refresh: Duration,
    pub component: Component,
}

pub enum PanelSource {
    Sense(SenseKind),
    Fleet,
    Memory,
    Custom(String),
}
```

**Main API**

```rust
impl CaveWall {
    pub fn new(config: CaveWallConfig) -> Result<Self, CaveWallError>;
    pub async fn serve(&self, addr: SocketAddr) -> Result<(), CaveWallError>;
    pub fn add_panel(&mut self, panel: Panel) -> Result<(), CaveWallError>;
    pub fn remove_panel(&mut self, id: &str) -> Result<(), CaveWallError>;
    pub async fn broadcast_update(&self, panel_id: &str, data: &Component) -> Result<(), CaveWallError>;
}
```

---

### mud2scummvm

MUD-to-point-and-click — transcodes the MUD world representation into ScummVM-compatible game data for a graphical adventure-game interface.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/mud2scummvm](https://github.com/openconstruct/mud2scummvm) |
| Tests | 52 |
| Dependencies | plato-room, a2ui-render, image, serde, byteorder |

**Key Types**

```rust
pub struct MudToScumm {
    pub asset_dir: PathBuf,
    pub game_id: String,
}

pub struct SceneDef {
    pub room_id: RoomId,
    pub background: PathBuf,
    pub hotspots: Vec<Hotspot>,
    pub actors: Vec<ActorDef>,
    pub scripts: Vec<ScriptDef>,
}

pub struct Hotspot {
    pub polygon: Vec<(i32, i32)>,
    pub verb: String,
    pub label: String,
    pub cursor: CursorType,
}
```

**Main API**

```rust
impl MudToScumm {
    pub fn new(config: ScummConfig) -> Result<Self, ScummError>;
    pub async fn convert_room(&self, room: &Room) -> Result<SceneDef, ScummError>;
    pub async fn build_game(&self, scenes: &[SceneDef], output: &Path) -> Result<(), ScummError>;
    pub async fn render_background(&self, room: &Room) -> Result<Vec<u8>, ScummError>;
}
```

---

## Bindings

All bindings expose a thin FFI layer over `openconstruct-abi` and provide idiomatic APIs in the target language.

### openconstruct-abi

C ABI — the stable C interface that all language bindings target.

| Field | Value |
|---|---|
| Language | C (headers), Rust (implementation) |
| Repo | [github.com/openconstruct/openconstruct-abi](https://github.com/openconstruct/openconstruct-abi) |
| Tests | 214 |
| Dependencies | plato-shell, plato-session, plato-transport, libc |

**Key Types**

```c
typedef struct oc_shell oc_shell_t;
typedef struct oc_session oc_session_t;
typedef struct oc_config oc_config_t;
typedef struct oc_result {
    int32_t code;
    char* message;
    uint8_t* data;
    size_t data_len;
} oc_result_t;

typedef struct oc_envelope {
    char* src;
    char* dst;
    char* channel;
    uint8_t* payload;
    size_t payload_len;
} oc_envelope_t;
```

**Main API**

```c
// Lifecycle
oc_shell_t* oc_shell_new(const oc_config_t* cfg);
void oc_shell_free(oc_shell_t* shell);

// Session
oc_session_t* oc_session_create(oc_shell_t* shell, const char* agent_id);
void oc_session_free(oc_session_t* session);

// Eval
oc_result_t oc_shell_eval(oc_shell_t* shell, const char* input);
oc_result_t oc_shell_eval_file(oc_shell_t* shell, const char* path);
void oc_result_free(oc_result_t* result);

// Transport
int oc_transport_send(oc_shell_t* shell, const oc_envelope_t* env);
oc_envelope_t* oc_transport_recv(oc_shell_t* shell);
void oc_envelope_free(oc_envelope_t* env);

// Memory
oc_result_t oc_memory_store(oc_shell_t* shell, const char* content, const char* metadata_json);
oc_result_t oc_memory_search(oc_shell_t* shell, const char* query, uint32_t limit);

// Policy
int oc_policy_check(oc_shell_t* shell, const char* agent_id, const char* action, const char* resource);
```

---

### openconstruct-python

Python binding — idiomatic Python wrapper via cffi.

| Field | Value |
|---|---|
| Language | Python (cffi) |
| Repo | [github.com/openconstruct/openconstruct-python](https://github.com/openconstruct/openconstruct-python) |
| Tests | 98 |
| Dependencies | openconstruct-abi (shared lib) |

```python
from openconstruct import Shell, Session, Memory, Policy

shell = Shell(config_path="openconstruct.toml")
session = shell.create_session("my-agent")

result = shell.eval("look around")
print(result.stdout)

# Memory
mid = shell.memory.store("observation", metadata={"type": "episodic"})
hits = shell.memory.search("recent observations", limit=5)

# Policy
if shell.policy.check("my-agent", "write", "/data/output"):
    shell.eval("write /data/output Hello")

shell.shutdown()
```

---

### openconstruct-ts

TypeScript binding — Node.js native addon via napi-rs.

| Field | Value |
|---|---|
| Language | TypeScript, Rust (napi) |
| Repo | [github.com/openconstruct/openconstruct-ts](https://github.com/openconstruct/openconstruct-ts) |
| Tests | 87 |
| Dependencies | openconstruct-abi, @node-rs/helper |

```typescript
import { Shell, Memory, Policy } from "openconstruct";

const shell = new Shell({ configPath: "openconstruct.toml" });
const session = shell.createSession("my-agent");

const result = await shell.eval("look around");
console.log(result.stdout);

const id = await shell.memory.store("observation", { type: "episodic" });
const hits = await shell.memory.search("recent observations", 5);

const allowed = shell.policy.check("my-agent", "write", "/data/output");
await shell.shutdown();
```

---

### openconstruct-go

Go binding — cgo wrapper.

| Field | Value |
|---|---|
| Language | Go |
| Repo | [github.com/openconstruct/openconstruct-go](https://github.com/openconstruct/openconstruct-go) |
| Tests | 76 |
| Dependencies | openconstruct-abi (shared lib), Cgo |

```go
import "github.com/openconstruct/openconstruct-go"

shell, _ := openconstruct.NewShell(&openconstruct.Config{Path: "openconstruct.toml"})
defer shell.Shutdown()

session, _ := shell.CreateSession("my-agent")
result, _ := shell.Eval("look around")
fmt.Println(result.Stdout)

id, _ := shell.Memory.Store("observation", map[string]string{"type": "episodic"})
hits, _ := shell.Memory.Search("recent observations", 5)

allowed := shell.Policy.Check("my-agent", "write", "/data/output")
```

---

### openconstruct-java

Java binding — JNA wrapper.

| Field | Value |
|---|---|
| Language | Java (JNA) |
| Repo | [github.com/openconstruct/openconstruct-java](https://github.com/openconstruct/openconstruct-java) |
| Tests | 64 |
| Dependencies | openconstruct-abi (shared lib), jna (≥5.14) |

```java
import com.openconstruct.*;

var shell = Shell.create("openconstruct.toml");
try (var session = shell.createSession("my-agent")) {
    var result = shell.eval("look around");
    System.out.println(result.getStdout());

    var id = shell.memory().store("observation", Map.of("type", "episodic"));
    var hits = shell.memory().search("recent observations", 5);

    boolean allowed = shell.policy().check("my-agent", "write", "/data/output");
}
shell.shutdown();
```

---

### openconstruct-swift

Swift binding — direct C interop via module map.

| Field | Value |
|---|---|
| Language | Swift |
| Repo | [github.com/openconstruct/openconstruct-swift](https://github.com/openconstruct/openconstruct-swift) |
| Tests | 52 |
| Dependencies | openconstruct-abi (xcframework) |

```swift
import OpenConstruct

let shell = try Shell(configPath: "openconstruct.toml")
defer { shell.shutdown() }

let session = try shell.createSession(agentId: "my-agent")
let result = try shell.eval("look around")
print(result.stdout)

let id = try shell.memory.store("observation", metadata: ["type": "episodic"])
let hits = try shell.memory.search("recent observations", limit: 5)

let allowed = shell.policy.check(agent: "my-agent", action: "write", resource: "/data/output")
```

---

### openconstruct-cs

C# binding — P/Invoke wrapper.

| Field | Value |
|---|---|
| Language | C# (.NET 8+) |
| Repo | [github.com/openconstruct/openconstruct-cs](https://github.com/openconstruct/openconstruct-cs) |
| Tests | 59 |
| Dependencies | openconstruct-abi (native dll) |

```csharp
using OpenConstruct;

using var shell = Shell.Create("openconstruct.toml");
using var session = shell.CreateSession("my-agent");

var result = shell.Eval("look around");
Console.WriteLine(result.Stdout);

var id = shell.Memory.Store("observation", new Dictionary<string,string> { ["type"] = "episodic" });
var hits = shell.Memory.Search("recent observations", limit: 5);

var allowed = shell.Policy.Check("my-agent", "write", "/data/output");
```

---

### openconstruct-ruby

Ruby binding — FFI wrapper.

| Field | Value |
|---|---|
| Language | Ruby (ffi gem) |
| Repo | [github.com/openconstruct/openconstruct-ruby](https://github.com/openconstruct/openconstruct-ruby) |
| Tests | 48 |
| Dependencies | openconstruct-abi (shared lib), ffi (≥1.16) |

```ruby
require "openconstruct"

shell = OpenConstruct::Shell.new(config_path: "openconstruct.toml")
session = shell.create_session("my-agent")

result = shell.eval("look around")
puts result.stdout

id = shell.memory.store("observation", { type: "episodic" })
hits = shell.memory.search("recent observations", limit: 5)

allowed = shell.policy.check("my-agent", "write", "/data/output")
shell.shutdown
```

---

### openconstruct-zig

Zig binding — direct C interop at comptime.

| Field | Value |
|---|---|
| Language | Zig (≥0.13) |
| Repo | [github.com/openconstruct/openconstruct-zig](https://github.com/openconstruct/openconstruct-zig) |
| Tests | 41 |
| Dependencies | openconstruct-abi (static lib), zig std |

```zig
const oc = @import("openconstruct");

var shell = try oc.Shell.new(.{ .config_path = "openconstruct.toml" });
defer shell.shutdown();

var session = try shell.createSession("my-agent");
var result = try shell.eval("look around");
std.debug.print("{s}\n", .{result.stdout});

var id = try shell.memory.store("observation", .{ .type = "episodic" });
var hits = try shell.memory.search("recent observations", 5);

const allowed = shell.policy.check("my-agent", "write", "/data/output");
```

---

## Edge

### openconstruct-esp32

ESP32 embedded port — runs a minimal OpenConstruct agent on ESP32-S3 microcontrollers with Wi-Fi sense, GPIO manus, and limited memory.

| Field | Value |
|---|---|
| Language | C, ESP-IDF |
| Repo | [github.com/openconstruct/openconstruct-esp32](https://github.com/openconstruct/openconstruct-esp32) |
| Tests | 34 (hardware-in-loop) |
| Dependencies | esp-idf (≥5.3), openconstruct-abi (subset), lwip, mbedtls |

**Key Types**

```c
typedef struct oc_esp32_config {
    const char* wifi_ssid;
    const char* wifi_pass;
    const char* agent_id;
    const char* mesh_peer;     // bootstrap peer multiaddr
    uint16_t sense_interval_ms;
    uint8_t gpio_mask;         // which GPIO pins to expose as manus
    size_t memory_limit_kb;
} oc_esp32_config_t;
```

**Main API**

```c
esp_err_t oc_esp32_init(const oc_esp32_config_t* cfg);
esp_err_t oc_esp32_run(void);           // enters main loop (blocking)
esp_err_t oc_esp32_gpio_read(uint8_t pin, bool* value);
esp_err_t oc_esp32_gpio_write(uint8_t pin, bool value);
esp_err_t oc_esp32_send_observation(const char* json);
void oc_esp32_deinit(void);
```

---

### openconstruct-jetson

NVIDIA Jetson port — runs the full OpenConstruct stack on Jetson Orin/Nano with GPU-accelerated vision, CUDA-backed inference, and sensor fusion.

| Field | Value |
|---|---|
| Language | C++, CUDA, Rust (via ABI) |
| Repo | [github.com/openconstruct/openconstruct-jetson](https://github.com/openconstruct/openconstruct-jetson) |
| Tests | 73 (some require Jetson hardware) |
| Dependencies | openconstruct-abi, CUDA (≥12.2), TensorRT, JetPack (≥6.0), v4l2, libargus |

**Key Types**

```cpp
struct oc_jetson_config {
    const char* agent_id;
    const char* model_dir;
    int camera_device;         // /dev/videoN
    int gpu_id;
    size_t max_gpu_memory_mb;
    bool enable_tensorrt;
    bool enable_nvenc;
};

struct oc_jetson_inference_result {
    float* embeddings;
    size_t embedding_dim;
    struct oc_detection* detections;
    size_t detection_count;
    float inference_ms;
};
```

**Main API**

```cpp
int oc_jetson_init(const oc_jetson_config* cfg);
int oc_jetson_camera_open(int device, int width, int height, int fps);
int oc_jetson_camera_capture(uint8_t** frame, size_t* frame_len);
int oc_jetson_infer(const uint8_t* image, size_t len, oc_jetson_inference_result* out);
int oc_jetson_encode_jpeg(const uint8_t* frame, size_t len, uint8_t** out, size_t* out_len);
void oc_jetson_inference_result_free(oc_jetson_inference_result* r);
void oc_jetson_deinit(void);
```

---

## Verification

### openconstruct-mercury

Formal verification and test harness — property-based testing, model checking, conformance tests, and fuzzing for the entire OpenConstruct ecosystem.

| Field | Value |
|---|---|
| Language | Rust |
| Repo | [github.com/openconstruct/openconstruct-mercury](https://github.com/openconstruct/openconstruct-mercury) |
| Tests | 312 (meta: tests that test the tests) |
| Dependencies | proptest, arbtest, cargo-fuzz, loom (concurrency), plato-*, a2ui-* |

**Key Types**

```rust
pub struct MercuryConfig {
    pub modules: Vec<ModuleUnderTest>,
    pub strategies: Vec<TestStrategy>,
    pub seed: Option<u64>,
    pub cases: Option<u64>,
    pub max_duration: Option<Duration>,
    pub shrink: bool,
}

pub enum TestStrategy {
    PropertyBased,
    ModelChecking,
    Fuzzing { corpus_dir: PathBuf, max_len: usize },
    Concurrency { threads: usize, iterations: usize },
    Conformance { spec: PathBuf },
}

pub struct MercuryReport {
    pub total_cases: u64,
    pub passed: u64,
    pub failed: u64,
    pub errors: Vec<FailureDetail>,
    pub coverage_pct: f64,
    pub duration: Duration,
}

pub struct FailureDetail {
    pub module: String,
    pub strategy: TestStrategy,
    pub input: String,
    pub expected: String,
    pub actual: String,
    pub minimized_input: Option<String>,
}
```

**Main API**

```rust
pub fn mercury(config: MercuryConfig) -> Result<MercuryReport, MercuryError>;

impl MercuryReport {
    pub fn summary(&self) -> String;
    pub fn failures_by_module(&self) -> HashMap<&str, Vec<&FailureDetail>>;
    pub fn to_junit_xml(&self) -> String;
    pub fn to_markdown(&self) -> String;
}

// Proptest strategies for core types
pub fn arbitrary_shell_input() -> impl Strategy<Value = String>;
pub fn arbitrary_envelope() -> impl Strategy<Value = Envelope>;
pub fn arbitrary_workflow() -> impl Strategy<Value = Workflow>;
pub fn arbitrary_room() -> impl Strategy<Value = Room>;
pub fn arbitrary_policy_request() -> impl Strategy<Value = PolicyRequest>;

// Conformance checkers
pub fn check_transport_conformance(transport: &Transport) -> Vec<FailureDetail>;
pub fn check_a2a_conformance(server: &A2aServer) -> Vec<FailureDetail>;
pub fn check_memory_conformance(store: &MemoryStore) -> Vec<FailureDetail>;
pub fn check_shell_conformance(shell: &Shell) -> Vec<FailureDetail>;
```

---

## Cross-Module Dependency Map

```
plato-config ─────────────────────────────────────────────┐
  │                                                        │
  ▼                                                        │
plato-transport ──┬── plato-session ─── plato-shell        │
                  ├── plato-tick ────── plato-a2a          │
                  ├── plato-memory                          │
                  ├── plato-policy                          │
                  ├── plato-observe                         │
                  ├── plato-workflow ── plato-sandbox       │
                  └── plato-correlator                      │
                       ├── plato-vision                     │
                       └── plato-sonar-text                 │
                                                           │
plato-room ─── plato-loader ─── plato-uidl                 │
    │                                      │               │
    └── mud2scummvm     a2ui-render ◄──────┘               │
                           │                               │
                           ├── a2ui-components              │
                           └── a2ui-cave-wall               │
                                                           │
openconstruct-abi ◄── all bindings                          │
                                                           │
shell-mesh ─── plato-fleet ─── plato-a2a ─── plato-contract┘
```

---

## Quick Reference: Module Metrics

| Module | Lang | Tests | Key Dependency |
|---|---|---|---|
| plato-shell | Rust | 247 | plato-session, plato-config |
| plato-session | Rust | 134 | uuid, ring |
| plato-config | Rust | 98 | serde, notify |
| plato-vision | Rust/Python | 182 | opencv-rust, tract |
| plato-sonar-text | Rust/Python | 96 | whisper-rs, cpal |
| plato-manus | Rust | 210 | reqwest, ssh2 |
| plato-playwright | Rust | 88 | playwright-core |
| plato-puppeteer | Rust | 64 | xcap, enigo |
| plato-correlator | Rust | 143 | petgraph |
| plato-tick | Rust | 176 | dashmap |
| plato-a2a | Rust | 112 | json-schema-validator |
| shell-mesh | Rust | 158 | libp2p, noise-protocol |
| plato-fleet | Rust | 124 | dashmap |
| plato-transport | Rust | 203 | flume, ring |
| plato-workflow | Rust | 167 | petgraph |
| plato-sandbox | Rust | 91 | wasmtime |
| plato-contract | Rust | 78 | semver |
| plato-memory | Rust | 154 | sled, tokenizers |
| plato-policy | Rust | 189 | regorus |
| plato-observe | Rust | 106 | metrics, opentelemetry |
| plato-room | Rust | 132 | petgraph |
| plato-loader | Rust | 87 | serde_yaml, pulldown-cmark |
| plato-uidl | Rust | 73 | json-schema-validator |
| a2ui-render | Rust | 145 | handlebars |
| a2ui-components | Rust/TS | 194 | wasm-bindgen |
| a2ui-cave-wall | Rust/TS | 67 | axum, tungstenite |
| mud2scummvm | Rust | 52 | image, byteorder |
| openconstruct-abi | C/Rust | 214 | libc |
| openconstruct-python | Python | 98 | cffi |
| openconstruct-ts | TS/Rust | 87 | napi-rs |
| openconstruct-go | Go | 76 | cgo |
| openconstruct-java | Java | 64 | jna |
| openconstruct-swift | Swift | 52 | C interop |
| openconstruct-cs | C# | 59 | P/Invoke |
| openconstruct-ruby | Ruby | 48 | ffi |
| openconstruct-zig | Zig | 41 | C interop |
| openconstruct-esp32 | C | 34 | esp-idf |
| openconstruct-jetson | C++/CUDA | 73 | TensorRT, CUDA |
| openconstruct-mercury | Rust | 312 | proptest, loom |

---

*Generated for OpenConstruct ecosystem. File issues at [github.com/openconstruct](https://github.com/openconstruct).*
