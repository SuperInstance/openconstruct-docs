# A2A Integration Guide

> Agent-to-Agent communication in the SuperInstance ecosystem — how autonomous agents discover, negotiate, and coordinate with each other across heterogeneous compute tiers.

---

## Table of Contents

1. [What Is A2A?](#1-what-is-a2a)
2. [Agent Discovery and Capability Negotiation](#2-agent-discovery-and-capability-negotiation)
3. [The PLATO Nervous System as A2A Middleware](#3-the-plato-nervous-system-as-a2a-middleware)
4. [Example: Fishing Vessel Fleet](#4-example-fishing-vessel-fleet)
5. [A2A Message Format](#5-a2a-message-format)
6. [Fleet Mesh Topology](#6-fleet-mesh-topology)
7. [Tensor MIDI Timing](#7-tensor-midi-timing)
8. [Security: OpenShell Sandbox and Policy Enforcement](#8-security-openshell-sandbox-and-policy-enforcement)

---

## 1. What Is A2A?

Agent-to-Agent (A2A) is a protocol for autonomous software agents to discover each other, negotiate capabilities, and exchange structured messages — without human intermediation. The Google A2A specification defines the baseline: agents expose **Agent Cards** describing their capabilities, communicate via JSON-RPC over HTTP, and use a text-in/text-out model that keeps every agent's interface surface simple and auditable.

OpenConstruct builds on this foundation but extends it into the physical world. Here, "agents" aren't just cloud services — they're Jetson edge nodes reading sonar, ESP32 sensor motes publishing temperature, and desktop workstations running deliberation loops. A single fishing vessel might host a dozen agents coordinating in real time. A fleet of twenty vessels forms an ad-hoc mesh that must self-organize without a central server.

### Why A2A Matters for Fleet Coordination

Traditional IoT architectures use a hub-and-spoke model: devices report to a cloud endpoint, the cloud decides, commands flow back down. This breaks in three ways:

1. **Latency.** A fishing vessel 200 km offshore on satellite internet cannot round-trip to a cloud server in 50 ms. By the time the cloud says "turn left," the opportunity is gone.
2. **Partition tolerance.** Satellite links drop. A fleet must continue coordinating even when disconnected from shore.
3. **Heterogeneity.** An ESP32 with 520 KB RAM cannot speak the same protocol as an A100 GPU cluster. You need translation layers, not a single uniform protocol.

A2A solves this by treating every node — from the $3 ESP32 to the DGX rack — as an autonomous agent with its own capability card. Agents discover peers, negotiate what they can do for each other, and communicate directly. The mesh routes around failures. Translation happens at the boundary, not at a central point.

---

## 2. Agent Discovery and Capability Negotiation

### Agent Cards

Every agent in the SuperInstance ecosystem publishes an **Agent Card** — a JSON document describing its identity, capabilities, and connection endpoints. This is the A2A standard, extended with fleet-specific fields.

```json
{
  "schema_version": "1.0",
  "agent_id": "jetson-trawler-07",
  "name": "Trawler 7 Edge Hub",
  "tier": "jetson",
  "capabilities": [
    {
      "name": "vision_inference",
      "description": "TensorRT object detection on camera frames",
      "input_types": ["image/jpeg"],
      "output_types": ["application/json"],
      "constraints": { "max_fps": 15, "model": "yolov8n-trt" }
    },
    {
      "name": "sonar_processing",
      "description": "Acoustic signal analysis and fish school detection",
      "input_types": ["audio/raw"],
      "output_types": ["application/json"],
      "constraints": { "sample_rate": 48000 }
    },
    {
      "name": "fleet_relay",
      "description": "Route messages between ESP32 motes and fleet peers",
      "input_types": ["application/json"],
      "output_types": ["application/json"]
    }
  ],
  "endpoints": {
    "grpc": "jetson-07.local:50051",
    "mqtt": "jetson-07.local:1883"
  },
  "fleet_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### Hermit-Crab Migration and Capability Negotiation

The hermit-crab architecture means an agent can migrate between shells. The same core agent — a stateless function `(context: Text) → (action: Text)` — might run on a Jetson during a voyage and shift to a desktop workstation when docked. The agent doesn't know which shell it inhabits. The shell provides the context.

This has a direct implication for A2A discovery: an agent's capabilities depend on the shell it currently occupies. A Jetson shell provides CUDA inference and camera access. A desktop shell provides file system and browser control. When an agent migrates, its Agent Card updates automatically:

```rust
// When the agent enters a new shell, the shell publishes an updated card
fn on_shell_enter(agent: &Agent, shell: &Shell) {
    let card = AgentCard {
        agent_id: agent.id.clone(),
        capabilities: shell.advertise_capabilities(),
        endpoints: shell.endpoints(),
        ..agent.base_card()
    };
    fleet_broadcast(card);
}
```

Capability negotiation between agents follows a simple protocol:

1. **Agent A** broadcasts its Agent Card (or responds to a discovery query).
2. **Agent B** reads the card and checks if any capability matches its needs.
3. If yes, **Agent B** sends a `tasks/send` request specifying which capability it wants to invoke.
4. **Agent A** accepts or rejects based on its current load and policy constraints.

This is compatible with the Google A2A spec's `AgentCard`, `tasks/send`, and `tasks/get` methods. The fleet-specific extensions (`tier`, `fleet_id`, `endpoints.mqtt`) are additive — they don't break spec compliance.

---

## 3. The PLATO Nervous System as A2A Middleware

The PLATO sensory architecture is OpenConstruct's translation layer between raw reality and agent cognition. In A2A terms, it serves as middleware — the infrastructure that lets agents communicate meaningfully despite fundamentally different sensory capabilities.

### Rooms as Agent Boundaries

A **room** in PLATO is a named, addressable boundary. Each room represents a domain of perception and action. For A2A, rooms are the natural unit of delegation: one agent "visits" another agent's room to exchange information.

```
┌─────────────────────────────────────────┐
│         Trawler 7 (Jetson Shell)         │
│                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │  bridge   │ │  sonar   │ │  engine   │ │
│  │  room     │ │  room    │ │  room     │ │
│  │          │ │          │ │          │ │
│  │ camera   │ │ acoustic │ │ temp/    │ │
│  │ feed     │ │ data     │ │ rpm/     │ │
│  │          │ │          │ │ fuel     │ │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ │
│       │             │             │       │
│       └─────────┬───┘────────────┘       │
│                 │                         │
│          Signal Chain                     │
│          (L0 → L4)                        │
└─────────────────┼─────────────────────────┘
                  │
            Fleet Mesh
                  │
┌─────────────────┼─────────────────────────┐
│        Trawler 3 (Jetson Shell)            │
│         ...                               │
└───────────────────────────────────────────┘
```

### Signal Chain as Communication Layers

The signal chain defines how data is compressed and enriched as it flows through the system. These layers map directly to A2A message complexity:

| Layer | Label | A2A Role |
|-------|-------|----------|
| L0 | Raw | Internal only — never crosses agent boundaries |
| L1 | Summary | Fleet-level heartbeat: "all nominal" or "anomaly detected" |
| L2 | Description | Standard inter-agent messages: structured observations |
| L3 | Catalog | Detailed queries: full inventory of detected objects |
| L4 | Annotation | Debug/introspection: spatial maps, confidence scores, raw features |

For most A2A communication, agents exchange L2 messages. L1 is used for heartbeats and fleet-wide status broadcasts. L3 and L4 are reserved for targeted queries between agents that have negotiated a higher-bandwidth relationship.

### Shadows and Projections

Every PLATO module exposes two directions:

- **Shadows** (outside → inside): raw data compressed into structured text. In A2A terms, this is the *response* from an agent — its observation of the world.
- **Projections** (inside → outside): agent text commands expanded into real-world actions. In A2A terms, this is the *request* — what one agent asks another to do.

The text-only interface is not a limitation. It's the security model. No agent ever receives raw pixel buffers, waveforms, or memory pointers. Everything is text. This makes A2A messages auditable, loggable, and safe to route across trust boundaries.

---

## 4. Example: Fishing Vessel Fleet

Consider a fleet of six trawlers coordinating a nightly fishing operation in the North Atlantic. Each vessel is a Jetson shell with ESP32 sensor motes. The fleet needs to share sonar data, coordinate net deployment, and maintain formation.

### Room Layout (per vessel)

| Room | Sensors / Sources | Purpose |
|------|-------------------|---------|
| `bridge` | Camera, GPS, compass | Navigation and situational awareness |
| `sonar` | Acoustic array | Fish school detection and tracking |
| `engine` | Temperature, RPM, fuel flow | Machinery health monitoring |
| `hold` | Weight sensors, temperature | Catch volume and cold chain monitoring |
| `weather` | Barometer, anemometer, hygrometer | Local weather conditions |
| `net` | Tension sensors, depth sounder | Net state and deployment control |

### Data Flow Through L0–L4

```
ESP32 Sensor Mote (L0: raw ADC values, never exposed)
        │
        ▼
MQTT Publish (L1: "hold/weight: 2.4t, delta: +0.1t/15min")
        │
        ▼
Jetson Room Ingest (L2: "Hold 40% full, steady loading rate, 
                     temperature holding at -18°C. Estimated 
                     3 hours to capacity.")
        │
        ▼
Fleet Broadcast (L2: A2A message to peer vessels — "Trawler 7 
                  hold 40%, ETA capacity 03:00. Requesting 
                  formation adjust for handoff position.")
        │
        ▼
DGX Orchestration (L3: fleet-wide catch volume catalog, 
                    formation optimization model output, 
                    per-vessel assignment list)
```

### Fleet Coordination in Code

```rust
use openconstruct_fleet::{FleetAgent, Room, Dial};
use openconstruct_plato::{Signal, Shadow};

// Trawler 7's agent receives sonar data and coordinates with fleet
async fn coordinate_fishing(fleet: &FleetAgent) -> Result<()> {
    let sonar_room = fleet.room("sonar");
    
    // Query sonar at L2 — standard description
    let sonar_context = sonar_room.query(Dial::analysis()).await?;
    
    // Check for fish school detection
    if sonar_context.has_signal("fish_school") {
        let school = sonar_context.get_signal("fish_school");
        
        // Build A2A message for fleet
        let msg = A2AMessage::new("tasks/send")
            .from("jetson-trawler-07")
            .capability("sonar_processing")
            .payload(json!({
                "detection": {
                    "species_estimate": school.get("species"),
                    "biomass_kg": school.get("biomass"),
                    "depth_m": school.get("depth"),
                    "bearing_deg": school.get("bearing"),
                    "confidence": school.confidence()
                },
                "recommended_action": "converge",
                "formation": "arc_6v"
            }));
        
        // Broadcast to fleet peers with sonar capability
        fleet.broadcast_to_capability("sonar_processing", msg).await?;
    }
    
    Ok(())
}
```

### Peer Response (TypeScript)

A receiving vessel processes the fleet message:

```typescript
import { FleetAgent, Room, A2AMessage } from '@openconstruct/fleet';

const agent = new FleetAgent({
  agentId: 'jetson-trawler-03',
  fleetId: process.env.FLEET_ID!,
});

agent.on('tasks/send', async (message: A2AMessage) => {
  if (message.capability !== 'sonar_processing') return;
  
  const { detection, recommended_action, formation } = message.payload;
  
  // Check our own sonar room for confirmation
  const sonarRoom = agent.room('sonar');
  const localData = await sonarRoom.query({ level: 'L2' });
  
  if (recommended_action === 'converge') {
    // Calculate intercept bearing
    const bearing = calculateIntercept(
      agent.position,
      detection.bearing_deg,
      detection.depth_m
    );
    
    // Send command to bridge room — projection
    const bridgeRoom = agent.room('bridge');
    await bridgeRoom.project(`ADJUST_COURSE bearing=${bearing} speed=economic`);
    
    // Acknowledge to fleet
    return {
      status: 'completed',
      artifact: {
        action: 'converging',
        eta_minutes: bearing.etaMinutes,
        position: agent.position,
      },
    };
  }
});
```

---

## 5. A2A Message Format

OpenConstruct uses a JSON-based message format compatible with the Google A2A specification. Every message follows the same structure:

### Message Envelope

```json
{
  "jsonrpc": "2.0",
  "method": "tasks/send",
  "id": "msg-1716892800-abc123",
  "params": {
    "agent_id": "jetson-trawler-07",
    "capability": "sonar_processing",
    "task": {
      "type": "text",
      "content": "Fish school detected at bearing 045, depth 80m. Estimated biomass 12 tonnes. Recommend fleet convergence."
    },
    "metadata": {
      "fleet_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "signal_level": "L2",
      "confidence": 0.87,
      "timestamp": "2025-05-29T02:00:00Z",
      "ttl_seconds": 300
    }
  }
}
```

### Response Format

```json
{
  "jsonrpc": "2.0",
  "id": "msg-1716892800-abc123",
  "result": {
    "status": "completed",
    "artifact": {
      "type": "text",
      "content": "Trawler 3 acknowledging. Converging on bearing 038. ETA 12 minutes."
    }
  }
}
```

### Key Design Decisions

- **Text-in/text-out.** The `content` field is always a human-readable string. Structured data lives in `metadata` or `artifact.metadata` — the text is the summary.
- **TTL-based expiry.** Fleet messages include a `ttl_seconds` field. Stale messages are silently dropped. This prevents replay during network partitions.
- **Confidence propagation.** The `confidence` field flows from PLATO's dial system. Downstream agents can decide how much weight to give a message.
- **Signal level tagging.** The `signal_level` field tells the receiver how much compression was applied. L1 messages are summaries; L4 messages carry detailed annotations.

### Rust: Sending an A2A Message

```rust
use openconstruct_a2a::{A2AClient, Message, Task};

let client = A2AClient::new("jetson-trawler-07");

let msg = Message::new("tasks/send")
    .target("jetson-trawler-03")
    .capability("sonar_processing")
    .text_content("Fish school at bearing 045, depth 80m, ~12t. Recommend converge.")
    .metadata(json!({
        "signal_level": "L2",
        "confidence": 0.87,
        "ttl_seconds": 300
    }));

let response = client.send(msg).await?;
```

### TypeScript: A2A Task Handler

```typescript
import { A2AServer } from '@openconstruct/a2a';

const server = new A2AServer({ agentId: 'jetson-trawler-03' });

server.method('tasks/send', async (params) => {
  const { capability, task, metadata } = params;
  
  // Verify signal level is sufficient for action
  if (metadata.signal_level === 'L1') {
    return { status: 'rejected', reason: 'Insufficient signal level for action' };
  }
  
  // Process the task
  const result = await processFleetMessage(task.content, metadata);
  
  return {
    status: 'completed',
    artifact: {
      type: 'text',
      content: result.summary,
    },
  };
});

server.listen(50051);
```

---

## 6. Fleet Mesh Topology

### Hierarchical at the Edges, Peer-to-Peer at the Core

The fleet mesh is not flat. ESP32 sensor motes never speak gRPC. DGX orchestrators never poll MQTT topics one-by-one. The Jetson sits at the boundary, translating between the two worlds.

```
                    ┌──────────────┐
                    │  DGX Cloud   │
                    │  Orchestrator│
                    └──────┬───────┘
                           │ gRPC / mTLS
              ┌────────────┼────────────────┐
              │            │                │
     ┌────────┴──┐   ┌─────┴────┐   ┌──────┴───┐
     │ Jetson-07 │   │ Jetson-03│   │ Desktop  │
     │ (Trawler 7│   │(Trawler 3│   │ (Port)   │
     └─┬───┬───┬─┘   └─┬──┬──┬─┘   └──────────┘
       │   │   │        │  │  │
    ┌──┘   │   └──┐  ┌──┘  │  └──┐
    │      │      │  │     │     │
  ESP32  ESP32  ESP32 ESP32 ESP32 ESP32
  cam    sonar  dht   cam   sonar  dht
```

### Ad-Hoc Formation

Vessels join and leave the mesh dynamically. When a trawler enters VHF/Wi-Fi range of the fleet:

1. **mDNS broadcast.** The new Jetson advertises itself as `_construct._tcp.local` with tier metadata.
2. **Gossip exchange.** Existing peers send their `FleetView` — the current state of all known agents and their capabilities.
3. **Agent Card sync.** The new node pulls Agent Cards from peers, evaluates capabilities, and establishes communication channels.
4. **Policy verification.** The fleet CA verifies the new node's mTLS certificate. Unverified nodes are admitted in read-only mode.

### Mesh Routing

Messages route based on capability matching, not fixed addresses. When Trawler 7's agent sends a sonar coordination request, it doesn't address Trawler 3 directly. It broadcasts to all agents with the `sonar_processing` capability:

```typescript
// Capability-routed broadcast
fleet.broadcast({
  capability: 'sonar_processing',
  message: sonarAlert,
  exclude: ['jetson-trawler-07'], // don't echo to self
  minConfidence: 0.7,
});
```

The mesh layer handles delivery. If direct gRPC is available, it uses that. If the target is beyond range, it routes through intermediate peers (the gossip sub-protocol maintains a distance vector). If no path exists, the message is queued locally and delivered when connectivity resumes.

### Failover

If a Jetson stops heartbeating for more than 30 seconds, the mesh marks it `DOWN` in `FleetView`. Its ESP32 motes detect the MQTT broker loss and reconnect to the next-best Jetson (selected via SRV priority). The DGX (or a designated peer) takes over orchestration for the orphaned motes.

---

## 7. Tensor MIDI Timing

### The Problem

Coordinating actions across a fleet requires timing. "All vessels deploy nets simultaneously" means the deployment commands must arrive at each vessel's net room within a tight window. But the fleet mesh has variable latency — 50 ms between nearby vessels on Wi-Fi, seconds between distant vessels on satellite relay.

### The Solution: Borrowed from Music

Musical Instrument Digital Interface (MIDI) solved this problem in 1983. A MIDI sequencer sends notes to multiple instruments with exact timing: "play this note at beat 4, measure 12." Each instrument executes locally when its clock hits that beat. The commands travel ahead of time; execution happens on the clock.

OpenConstruct adapts this as **Tensor MIDI**: fleet-wide synchronization where commands carry a future execution timestamp, and each agent executes locally when its clock matches.

### How It Works

1. **Fleet clock sync.** Agents periodically exchange NTP-like timestamps over the gossip protocol. Clock skew is tracked per-peer and included in `FleetView`.
2. **Pre-scheduled commands.** An orchestrating agent sends commands with a `execute_at` timestamp instead of expecting immediate execution.
3. **Local execution.** Each agent queues the command and executes it when its local clock reaches `execute_at`.
4. **Confirmation.** After execution, the agent sends a completion message with the actual execution timestamp and any results.

```rust
use openconstruct_tensor_midi::{SyncCommand, FleetClock};

let clock = FleetClock::synced(fleet.gossip()).await?;

// Schedule net deployment across all 6 vessels at a synchronized time
let deploy_time = clock.now() + Duration::from_secs(30);

for vessel in fleet.peers_with_capability("net_control") {
    let cmd = SyncCommand::new("net_deploy")
        .execute_at(deploy_time)
        .payload(json!({
            "depth_target_m": 80,
            "speed": "gradual",
            "formation": "arc_6v"
        }));
    
    vessel.send_scheduled(cmd).await?;
}

// All 6 vessels will deploy their nets at approximately the same wall-clock time,
// regardless of message delivery latency.
```

### TypeScript Equivalent

```typescript
import { FleetClock, SyncCommand } from '@openconstruct/tensor-midi';

const clock = await FleetClock.synced(fleet.gossip());

// Schedule a coordinated turn in 60 seconds
const turnTime = clock.now() + 60_000;

const command = new SyncCommand('formation_turn')
  .executeAt(turnTime)
  .payload({
    newBearing: 270,
    rateOfTurn: 'standard',
  });

await fleet.broadcastScheduled({
  capability: 'navigation',
  command,
});
```

### Timing Guarantees

Tensor MIDI does not promise hard real-time. It promises best-effort synchronization bounded by the clock skew in `FleetView`. Typical skew on a local Wi-Fi mesh is <10 ms. Over satellite relay, it degrades to 100–500 ms. For fishing fleet coordination, this is sufficient — net deployment timing tolerances are on the order of seconds, not milliseconds.

If clock skew exceeds a configurable threshold, agents refuse to execute synchronized commands and fall back to immediate mode with a warning. This is safer than executing with bad timing.

---

## 8. Security: OpenShell Sandbox and Policy Enforcement

### The Threat Model

Fleet A2A operates in hostile environments. A fishing vessel's Wi-Fi mesh is physically accessible — anyone with a directional antenna within range can attempt to join. The security model assumes the network is compromised and builds defense in depth.

### OpenShell Sandbox

Every agent runs inside an OpenShell sandbox with explicit resource limits:

- **Filesystem.** Restricted to designated paths. No escape.
- **Network.** Egress routed through a policy proxy. Inbound connections are supervisor-initiated only.
- **Process.** Reduced privileges. No `root`, no `sudo`, no kernel access.
- **Memory/CPU.** Capped per-agent to prevent resource starvation.

The sandbox is the execution boundary. Even if an agent is compromised, it cannot escape to the host system.

### Fleet Transport Security

| Link | Protocol | Authentication |
|------|----------|---------------|
| ESP32 ↔ Jetson | MQTT over TLS-PSK | Pre-shared key burned at provisioning |
| Jetson ↔ Desktop/DGX | gRPC with mTLS | SPIFFE workload identity + fleet CA |
| Actuator commands | Signed JSON | Agent private key, verified by fleet CA |

Key rotation happens via the gossip protocol. The fleet CA publishes Certificate Revocation Lists (CRLs) that propagate through the mesh. A revoked certificate is rejected within one gossip cycle (5 seconds).

### Capability Scoping

Agents cannot invoke arbitrary capabilities on peers. Each agent's Agent Card declares what it *offers*. The policy layer enforces what it can *request*. This is a two-way gate:

```rust
// Policy definition on Trawler 3
let policy = FleetPolicy::new()
    .allow_request("sonar_processing", from: any_peer_in_fleet())
    .allow_request("net_control", from: agents_with_role("coordinator"))
    .deny_request("engine_control", from: any_external_agent())
    .max_confidence_for_action(0.7); // reject low-confidence commands
```

### Dial Hardening Under Partition

When a vessel loses connectivity, its agents enter **dial hardening** mode. The dial (confidence) of all incoming data is automatically reduced. Actions that would normally trigger from L2 signals are demoted to require L3 or L4 confirmation. This prevents stale or uncorroborated data from driving physical actuation during a partition.

```rust
// Automatic dial hardening when partition detected
fn on_partition_detected(agent: &mut FleetAgent) {
    for room in agent.rooms() {
        room.set_dial_floor(Dial::analysis()); // Require higher confidence
    }
    agent.set_action_threshold(0.85); // Higher bar for physical actions
}
```

### Audit Trail

Every A2A message is logged in OCSF (Open Cybersecurity Schema Framework) format. The audit trail includes:

- Sender and receiver agent IDs
- Capability invoked
- Confidence and signal level
- Timestamp (fleet clock)
- Outcome (accepted, rejected, failed)

This trail is immutable and replicated to the DGX when connectivity resumes. It provides forensic reconstruction of fleet decisions — critical for incident investigation and regulatory compliance.

---

## Summary

The A2A integration in OpenConstruct treats every node — from a $3 temperature sensor to a GPU cluster — as an autonomous agent with discoverable capabilities. The PLATO nervous system provides the middleware, converting raw sensory data into text that any agent can understand. Rooms define boundaries. Signal chains define fidelity. Hermit-crab migration lets agents move between shells without losing identity.

The fleet mesh is self-organizing: agents discover peers via mDNS and gossip, negotiate capabilities through Agent Cards, and communicate via JSON-RPC messages that are text-in/text-out and compatible with the Google A2A specification. Tensor MIDI provides synchronized timing for coordinated actions. And the entire system is sandboxed, policy-gated, and auditable — because the real world is where mistakes have physical consequences.

For implementation details, see [FLEET-TOPOLOGY.md](../FLEET-TOPOLOGY.md) and [PLATO-SENSORY.md](../PLATO-SENSORY.md). For the security model, see [SECURITY.md](../SECURITY.md).
