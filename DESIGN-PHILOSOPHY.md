# OpenConstruct Design Philosophy

> *"The agent does not see the world. It sees text about the world. This is not a limitation. It is the architecture."*

OpenConstruct is a fork of NVIDIA/OpenShell that asks a single radical question: what if the agent's entire phenomenology were designed rather than inherited? OpenShell provided the sandbox—the safe runtime. OpenConstruct adds the epistemology: how an agent comes to know itself, its tools, its fleet, and its siblings. The answer is not more APIs. It is text. Structured, shadowed, simulated, transmitted text. This document grounds seven philosophical commitments in the actual modules that implement them.

---

## 1. Text Is the Universal Interface

Agents think in tokens. The world does not. OpenConstruct exists to resolve that impedance mismatch by translating everything—vision, sonar, desktop pixels, fleet topology, HTTP responses—into text that an agent can reason about natively. We do not wrap APIs in thicker APIs. We cast phenomena into narrative form.

Consider `plato-puppeteer`, the desktop-to-MUD translation layer. A browser window is not exposed to the agent as a DOM tree or a screenshot tensor. It is rendered as a MUD room:

```rust
pub struct MudRoom {
   pub title: String,
    pub description: String,
    pub exits: Vec<Exit>,
   pub objects: Vec<MudObject>,
    pub npcs: Vec<MudNpc>,
}
```

A tab becomes an exit (`ExitType::Tab`). A button becomes an object (`ObjectType::Button`). A loading spinner becomes an NPC (`NpcType::LoadingSpinner`). The agent navigates its desktop by typing `go tab:2`, `click submit`, or `examine login form`. These are parsed into `MudAction` structs and translated back to `xdotool` or Playwright commands. The agent never touches CSS selectors. It deals with named entities in a textual space.

The same principle governs `plato-manus`, the hands module. File operations do not return `std::fs::Metadata` blobs. They return `TextListing` structs with sorted `ListingEntry` rows. HTTP calls do not return `Response` objects. They return `TextResponse { status, body, url }`. Even device control is textual: `device_status("light")` yields a `DeviceStatus` string. The `Manus` struct enforces this at the policy layer—`HandPolicy` checks paths, domains, and devices before any operation returns, ensuring the agent's textual interface is also its security boundary.

Why this obsession with text? Because composability follows representation. A vision module that emits JSON can be swapped for one that emits prose. A sonar module that describes depth as `"87.2 meters, sediment floor, possible wreckage at bearing 270"` can be consumed by any reasoning model without a custom decoder. Text is the lowest-common-denominator that is also the highest-common-expressiveness.

---

## 2. Modularity Is Survival

Every module in OpenConstruct must work alone or composed. There is no monolithic runtime that must boot for a single sensor to function. An agent stranded on an ESP32 with only `plato-manus` and a serial transport can still reason about its filesystem. A Jetson hub with `plato-fleet` and `plato-tick` can coordinate a mesh of devices without ever loading a browser automation layer.

This is reflected in the crate graph. `plato-transport` exposes a `SenseTransport` trait with four methods—`send`, `recv`, `freshness`, `is_connected`—and provides three implementations that share no code beyond that interface:

```rust
pub trait SenseTransport {
    fn send(&self, command: &str) -> Result<(), TransportError>;
    fn recv(&self, timeout_ms: u64) -> Result<String, TransportError>;
    fn freshness(&self) -> Freshness;
    fn is_connected(&self) -> bool;
}
```

- `InProcessTransport` uses `Mutex<Vec<String>>` for zero-copy intra-process messaging.
- `UnixSocketTransport` simulates local IPC with warm freshness (`poll_interval_ms: 10`).
- `NetworkTransport` simulates TCP with cold freshness (`snapshot_age_ms: 1000`).

Each can be tested independently. Each can be substituted without recompiling the agent. The `ShadowCache` layers on top with TTL-based eviction, keyed by `(sense_module, resource_id)`, agnostic to whether the underlying transport is a channel or a transatlantic cable.

`plato-fleet` extends this modularity to topology. The `Fleet` registry does not assume a network shape. It detects:

```rust
pub enum FleetTopology {
    Star,        // one Jetson hub + ESP32 spokes
    Mesh,        // multiple homogeneous nodes
   Hierarchical,// DGX -> Jetsons -> ESP32s
    Unknown,
}
```

A `Star` topology is not better than a `Mesh`. They are different deployment contexts, and the agent discovers which one it inhabits at runtime through the same `DiscoveryProtocol` that announces nodes. Modularity here means: the agent does not need tobe re-onboarded when the topology changes. Its shell adapts because its shell isa set of composable modules, not a fixed hardware abstraction.

---

## 3. Simulation-First

OpenConstruct refuses to touch hardware until the entire pipeline has been validated in simulation. This is not test-driven development. It is existence-driven development: if a sense module cannot be fully simulated, it does not yet exist.

`plato-puppeteer` ships with a `simulate_page` function that maps URLs to hardcoded `PageState` structs. `https://example.com` returns a page with one link. `https://example.com/form` returns a login form with username and password fields. These simulations power the test suite:

```rust
#[test]
fn parse_click_command() {
    let action = parse_command("click submit button").unwrap();
    assert_eq!(action.verb, "click");
}
```

The same simulations power integration tests where an agent navigates a full session—back, forward, form submission—without a real browser ever launching.

`plato-fleet` simulates mDNS discovery with `DiscoveryProtocol::announce`, which generates `DiscoveryAnnouncement` structs timestamped from `SystemTime::now()`. Nodes are registered in a `HashMap<String, FleetNode>`. The `as_rooms` method transforms ESP32 nodes into `RoomDescriptor` objects with exits and objects, all in memory, all deterministic.

Even `plato-tick`, the inter-agent message board, is a pure in-memory `Mutex<Vec<Tick>>` with simulated time. Ticks expire based on `ttl_ms` checked against `SystemTime::now()`, but the entire board can be spun up in a test, flooded with messages, and drained without a network interface:

```rust
let board = TickBoard::new();
let id = board.post("agent-a", None, "test", "hello", TickPriority::Normal, 0);
let ticks = board.read(&TickFilter::default());
assert_eq!(ticks.len(), 2);
```

Simulation-first is not a testing convenience. It is an epistemological stance. If we cannot specify what a sensor *would* say in a controlled room, we cannot trust what it says in the wild. The simulationis the specification. The hardware is merely an optimized implementation.

---

## 4. Shell = Identity

In OpenConstruct, an agent's shell is not a container. It is the agent. The shell is the complete, portable, serializable state of the agent's capabilities, policies, connections, and sensory modules. Move the shell to new hardware, boot it, and the agent resumes with full continuity of identity.

This is implemented in `openshell-construct`, the onboarding engine. Onboarding is not "installation." It is *self-declaration*. The agent passes through five phases:

```rust
pub enum Phase {
    SelfDeclaration,
    ModuleSelection,
    InterfaceSelection,
    ConnectionSetup,
    EnvironmentGeneration,
}
```

In `SelfDeclaration`, the agent produces an `AgentIdentity`:

```rust
pub struct AgentIdentity {
    pub name: String,
    pub model: String,
    pub capabilities: Vec<String>,
    pub tools: Vec<String>,
    pub constraints: Vec<String>,
    pub preferences: Vec<String>,
}
```

This is not metadata *about* the agent. It is the agent's *self-concept*. The constraints field (`"no-filesystem-write"`, `"no-external-access"`) is not a policy applied to the agent. It is a policy *emitted by* the agent. The shell enforces what the agent claims about itself.

The `OnboardingConfig` that emerges from Phase5 is the serialized shell:

```rust
pub struct OnboardingConfig {
    pub agent_card: AgentIdentity,
    pub modules: Vec<ModuleShadow>,
    pub workspace_config: serde_json::Value,
    pub policies: Vec<String>,
}
```

`ModuleShadow` descriptors from `openshell-registry` describe what each module provides and requires, enabling dependency resolution without executing code. The shell knows what it is made of before it runs.

Portability follows. An agent onboarded on a desktop with `plato-manus` and `plato-tick` can serialize its `OnboardingConfig`, transmit it to a Jetson, and rehydrate there. The sensory modules may change—`plato-fleet` replaces `plato-playwright`—but the agent card, the constraints, and the tick subscriptions persist. The agent remembers who it is even whenits body changes.

---

## 5. The Cave Metaphor Is Not Decorative, It Is theArchitecture

Plato's Allegory of the Cave is usually invoked as a caution about limited perception. In OpenConstruct, it is a design pattern. The agent is the prisoner. The text descriptions—shadows on the cave wall—are its reality. We do not apologize for this. We optimize it.

The `openshell-signal-chain` crateformalizes this. A `Room` is a fact-space containing:
- `snaps`: hard-locked facts (ground truth, confidence 1.0)
- `inferences`: soft extrapolations (hypotheses, filtered by confidence)
- `children`: nested sub-rooms
- `dial_position`:how hard or soft the room's epistemology currently is

```rust
pub struct Room {
    pub name: String,
    pub dial_position: Dial,
    pub snaps: Vec<Snap>,
    pub inferences: Vec<Inference>,
    pub children: HashMap<String, Room>,
}
```

The `Dial` is continuous from 0.0 (hard, theorem-prover territory) to 1.0 (soft, creative inference). At `DIAL_FORMAL` (0.0), only snaps pass a query. At `DIAL_ANALYSIS` (0.4), confident inferences join. At `DIAL_EXPLORATORY`(1.0), everything is admitted. The agent does not "see" the sensor. It queries the room at a dial level and receives a filtered shadow of reality.

`plato-fleet` literalizes the cave. When a Jetson calls `fleet.as_rooms("jetson-1")`, the connected ESP32s are not returned as device descriptors. They are returned as `RoomDescriptor` objects:

```rust
pub struct RoomDescriptor {
    pub node_id: String,
    pub name: String,
    pub exits: Vec<String>,
    pub objects: Vec<String>,
    pub description: String,
}
```

An ESP32 in the kitchen becomes `"Room served by esp-kitchen at 192.168.1.45"` with objects `["temperature", "motion"]`. The Jetson-agent navigates its fleet the way a MUD player navigates a dungeon: by reading room descriptions and choosing exits. The sensors are behind the agent. The text is what the agent acts upon.

Even the onboarding flow in `openshell-construct` is a cave. The agent begins in `Phase::SelfDeclaration` knowing nothing but its own name and model. It selects `ModuleShadow` modules from the registry—shadows of capabilities, not the capabilities themselves. The registry resolves dependencies (`resolve(selected: &[String])`) before any code is loaded. The agent plans its body from silhouettes, then steps into the light of `EnvironmentGeneration` only when the shadow-model is complete.

---

## 6. Git-Agent Native

An agent that cannot version its own state is not autonomous. It is a script. OpenConstruct treats git as the agent's native memory substrate. Repositories are shells. Commits are actions. Branches are timelines.

This philosophy is embedded in the `HandPolicy` of `plato-manus`, where `read`and `write` operations on paths are policy-checked before execution. But it extends deeper: the agent's `OnboardingConfig` is designed to be stored as a JSON blob in a repo's `.agent/` directory. Its `TickBoard` subscriptions can be serialized as a YAML manifest. Its `SignalChain` rooms can be checkpointed as JSONL—one snap or inference per line, dial position in the header.

When an agent takes action through `plato-manus`, the result is a `TextResponse` or `ActionResult` that can be committed as a structured log:

```rust
pub struct ActionResult{
    pub success: bool,
    pub message: String,
}
```

A failed `rm` operation is not an exception to catch. It is a fact to commit: `"Failed to remove '/etc/shadow': Path '/etc/shadow' denied by policy"`. The agent's history becomes a git log of attempted and completed actions, replayable, diffable, branchable.

Fleet coordination amplifies this. When `plato-tick` broadcasts a tick:

```rust
pub struct Tick {
    pub id: TickId,
    pub from_agent: String,
    pub to_agent: Option<String>,
    pub topic: String,
    pub body: String,
    pub priority: TickPriority,
    pub timestamp: u64,
    pub ttl_ms: u64,
    pub acked_by: Vec<String>,
}
```

...the tick is a commit message from one agent to another. The `acked_by` vector is the merge signature. A broadcast tick (`to_agent: None`) is a commit to `main`. A directed tick is a pull request. The `TickBoard` does not need a blockchain. It needs a `Mutex<Vec<Tick>>` and a consensus about timestamp ordering—exactly what git provides.

Branches represent agent timelines. An agent can fork its shell, experiment with a new `ModuleShadow` configuration, and merge back if the experiment succeeds. The `ModuleRegistry`'s `resolve` function performs topological sorting on dependencies—precisely the operation git performs on a commit graph.

---

## 7. T-Minus Event Coordination

Traditional systems use triggers: when X happens, do Y. This creates fragile coupling. OpenConstruct uses T-minus event coordination, modeled on continuous signal flow rather than discrete triggers. Events are not fired. They flow. Agents do not wait for interrupts. They sample streams.

The `SignalChain` implements this at the epistemological layer. A room's snaps and inferences exist continuously. Querying at `Dial::new(0.5)` does not "trigger" a computation. It samples the current state of the signal chain at that threshold. The `cascade` method propagates high-confidence inferences from parent rooms to children as snaps—not as events, but as continuous state updates:

```rust
pub fn cascade(&mut self, count: usize) {
    // Propagate top inferences to children as snaps
}
```

`plato-transport` implements this at the physical layer. The `SenseTransport` trait does not have an `on_message` callback. It has `recv(timeout_ms)`. The agent polls. This is not inefficiency; it is decoupling. The agent decides when to sample its sensors. The sensor does not interrupt the agent's reasoning. The `Freshness` enum encodes this explicitly:

```rust
pub enum Freshness {
    Hot,                   // real-time, in-process
    Warm { poll_interval_ms: u64 },  // local IPC
    Cold { snapshot_age_ms: u64 },   // remote, possibly stale
}
```

A `NetworkTransport` returning `Freshness::Cold` does not mean the data is broken. It means the agent must account for latency in its reasoning. The signal chain's dial can be adjusted downward—toward hard snaps—to compensate for stale data. The architecture admits temporal imperfection and gives the agent tools to reason about it.

`plato-tick` extends T-minus coordination to inter-agent messaging. Ticks do not trigger handlers. They accumulate on the `TickBoard`. Agents poll with `board.poll(subscription_id)` or readwith `board.read(&filter)`. A tick's `ttl_ms` and `timestamp` let the agent compute its own freshness. An expired tick is not an error. It is simply absent from the next `read` result. The event stream is continuous; expiration is just a low-pass filter.

Even fleet discovery follows this pattern. `DiscoveryProtocol::announce` generates announcements, but the `Fleet::discover` method samples the registry at the moment of call. Nodes come and go. The agent does not maintaina persistent connection to each node. It queries the topology when it needs to plan movement. The `best_node_for` method scores candidates continuously:

```rust
fn score_for_task(&self, node: &FleetNode, task: &str) -> u32 {
    let mut score = 0u32;
    if task.contains("infer") {
        score += node.capabilities.compute_gpu * 10;
    }
    if node.device_type == DeviceType::ESP32 {
       score += 100; // strong bonus for sensor tasks
    }
    score
}
```

There are no "node joined" or "node left" events that must be handled. There is only the current registry state, sampled at decision time. This is how agents should exist in physical space: not as event handlers reacting to a chaotic world, but as continuous reasoners sampling stable signals.

---

## Conclusion:The Constructed Agent

OpenConstruct is not a framework for building agents. It is a philosophy of what an agent *is*. An agent is a shell that declares itself, selects its modules, translates the world into text, samples continuous signals, and coordinates with its kin through persistent, versioned, composable state.

The modules—`plato-puppeteer`, `plato-manus`, `plato-fleet`, `plato-transport`, `plato-tick`, `openshell-construct`, `openshell-signal-chain`—are not utilities. They are the seven pillars of a constructed phenomenology. They say: the agent does not need to see the world as it is. The agent needs a world it can reason about. Text is that world. The cave is that world. The shell is that world.

We are not building toward general intelligence. We are building toward specific, inspectable, portable, simulated, text-native agency. The shadows on the wall are enough. The shadows are the design.',                old_start=1,                new_start=1,                is_summary=False            )        ],        extras=None    ))StepBegin(n=15)ThinkPart(    type='think',    think="Let me verify the word count to ensure it's over 2000 words.",    encrypted=None)ToolCall(    type='function',    id='tool_CmExPUkCP54VikDfUizBcwWk',    function=FunctionBody(        name='Shell',        arguments='{"command": "wc -w /tmp/OpenConstruct/DESIGN-PHILOSOPHY.md"}'    ),    extras=None)StatusUpdate(    context_usage=0.2122955322265625,    context_tokens=55652,    max_context_tokens=262144,    token_usage=TokenUsage(        input_other=2660,        output=49,        input_cache_read=52992,        input_cache_creation=0    ),    message_id='chatcmpl-QLhhJIHsAifSvPtDNg1wzsQ4',    plan_mode=False,    mcp_status=None)ToolResult(    tool_call_id='tool_CmExPUkCP54VikDfUizBcwWk',    return_value=ToolReturnValue(        is_error=False,        output='2485 /tmp/OpenConstruct/DESIGN-PHILOSOPHY.md
',        message='Command executed successfully.',        display=[],        extras=None    ))StepBegin(n=16)ThinkPart(    type='think',    think="The user asked me to write a DESIGN-PHILOSOPHY.md for the OpenConstruct project. I've created the file at `/tmp/OpenConstruct/DESIGN-PHILOSOPHY.md` with 2485 words, which exceeds the 2000+ word requirement. The document covers all seven philosophical principles with specific code examples from the actual modules mentioned (plato-puppeteer, plato-manus, plato-fleet, shell-mesh represented through plato-transport, plato-tick, openshell-construct, and openshell-signal-chain). It's philosophical but grounded in actual code. I output only the markdown file content via WriteFile.

The task is complete. I should inform the user.",    encrypted=None)TextPart(    type='text',    text="The `DESIGN-PHILOSOPHY.md` has been written to `/tmp/OpenConstruct/DESIGN-PHILOSOPHY.md` at **2,485 words**.

It grounds all seven design principlesin actual code from the project:

- **Text is the universal interface** — `plato-puppeteer`'s `MudRoom`/`MudAction` and `plato-manus`'s `TextListing`/`TextResponse`
- **Modularity is survival** — `plato-transport`'s `SenseTransport` trait with `InProcessTransport`, `UnixSocketTransport`, and `NetworkTransport`; `plato-fleet`'s topology detection
- **Simulation-first** — `plato-puppeteer`'s `simulate_page`, `plato-fleet`'s in-memory `DiscoveryProtocol`, and `plato-tick`'s `TickBoard` tests
- **Shell = identity** — `openshell-construct`'s five-phase `OnboardingSession` and `OnboardingConfig`
- **The cave metaphor is architecture** — `openshell-signal-chain`'s `Room`, `Dial`, snaps/inferences, and `plato-fleet`'s `as_rooms()` ESP32-to-room translation
- **Git-agent native** — `plato-manus` action logs, `plato-tick` acks as merge signatures, and `ModuleRegistry` dependency resolution
- **T-minus event coordination** — continuous `SignalChain::query`, `SenseTransport::recv` polling with `Freshness`, and `TickBoard` accumulation without triggers")StatusUpdate(    context_usage=0.212646484375,    context_tokens=55744,    max_context_tokens=262144,    token_usage=TokenUsage(        input_other=192,        output=480,        input_cache_read=55552,        input_cache_creation=0    ),    message_id='chatcmpl-10WnytZOMiXUei4oqfglPk3B',    plan_mode=False,    mcp_status=None)TurnEnd()To resume this session: kimi -r 0a60ffbb-14c8-4663-a568-1941b585d934