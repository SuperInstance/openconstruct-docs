# What Is OpenConstruct?

OpenConstruct is an open-source framework for building AI agents that can see, hear, touch, and act on the real world — and that can work together in fleets spanning a $3 ESP32 microcontroller to a $25,000 GPU cluster. It is a fork of NVIDIA's OpenShell (Apache 2.0) that adds sensory perception, fleet coordination, structured onboarding, and inter-agent communication. An agent built with OpenConstruct is not a stateless chat endpoint. It is a persistent, embodied, collaborating entity with a workspace, senses, hands, and peers.

This document explains why that distinction matters, how the architecture achieves it, and how to start building with it.

---

## The Problem

Most AI agents today share a set of limitations so fundamental that we've stopped noticing them:

**They are stateless.** An agent's entire existence is a request-response cycle. It wakes up, processes a prompt, produces output, and dies. If you ask it the same question twice, it has no memory of the first time. Context windows are a bandage — they expire, they overflow, they cost money proportional to length. There is no continuity.

**They are senseless.** An LLM has never seen a photograph. It has never heard a sound. It has never felt a button press or a temperature change. It processes text and only text. When we ask an agent to "look at the security camera," we are lying to ourselves — we are passing it a base64 string and hoping the tokenizer does something useful with it. The agent has no sensory apparatus. It is a brain in a jar, and the jar has no windows.

**They are isolated.** Two agents running on the same machine, built by the same team, backed by the same model, cannot coordinate without a human writing glue code between them. There is no standard for agent-to-agent communication. There is no discovery protocol. There is no shared memory. Each agent is an island, and the bridges are ad-hoc and fragile.

**They are disembodied.** An agent cannot push a button. It cannot turn off a light. It cannot navigate a desktop, fill in a form, or respond to a physical event. Every action requires a human intermediary translating the agent's text output into real-world effect. The agent is an advisor that can never act on its own advice.

**They die when the session ends.** No workspace persists. No identity survives. No accumulated knowledge carries forward. Each session is Groundhog Day without the learning.

These aren't minor inconveniences. They are the reason AI agents remain toys instead of infrastructure. An agent that can't see, can't act, can't remember, and can't cooperate is not an agent — it's a very expensive autocomplete.

---

## The Insight

An AI agent is a hermit crab.

A hermit crab is born with a soft body — capable, but unprotected. It cannot survive without a shell. The shell provides structure, protection, and a place to operate from. When the crab outgrows its shell, it finds a bigger one. A small crab lives in a small shell. A large crab lives in a large one. The crab is the same creature either way.

OpenConstruct applies this to agents. The agent core — the LLM, the reasoning engine, the decision maker — is the soft body. It needs:

**A shell (workspace):** A persistent directory with configuration, memory, identity, and state. The shell is the agent's home. It can be a directory on a laptop, a container in the cloud, or a partition on a Jetson. The agent carries its identity in its shell, not in its weights.

**Senses (to perceive):** The agent needs eyes, ears, and other sensory channels — but crucially, these senses don't need to feed raw data into the LLM. They need to translate reality into text. A camera that produces "a person is standing near the door, wearing a red jacket" is more useful to a text-based agent than one that produces a 1920x1080 pixel array. The sense is the translator, not the sensor.

**Hands (to act):** The agent needs to affect the world — manipulate files, call APIs, control devices, drive browsers, press buttons. These hands translate agent text commands into real-world actions, the same way senses translate real-world data into text.

**A network (to collaborate):** The agent needs to discover peers, leave messages, subscribe to events, and coordinate. Not through ad-hoc HTTP calls, but through a structured bulletin board system that any agent in the fleet can read and write.

The key insight is that the shell scales. A hermit crab can live in a shell the size of a thimble or a coconut. An OpenConstruct agent can live on:

- An **ESP32** ($3, 520KB RAM, WiFi): runs sensors, posts ticks, acts as a "room" that larger agents can visit
- A **Jetson Orin Nano** ($250, 8GB RAM, GPU): runs local inference, processes camera and audio feeds, coordinates its cluster of ESP32s
- A **desktop** ($1500, 32GB RAM): runs full agents with desktop control, browser automation, development tools
- A **cloud GPU node** (variable): runs heavy model inference, fleet orchestration, batch processing
- A **DGX Spark** ($25,000, 128GB unified memory): runs everything simultaneously, trains models, orchestrates the entire fleet

The agent is the same creature in every shell. Its identity, memory, and configuration travel with it. The shell determines what it can reach — not what it is.

---

## How It Works

### Plato's Cave

The foundational architectural metaphor is Plato's Cave, and it is not decorative — it is the literal architecture.

In Plato's allegory, prisoners in a cave see only shadows on the wall, cast by objects behind them. They never see the objects directly. They understand reality entirely through shadows.

In OpenConstruct, the agent is the prisoner. It lives in a cave of text. It never sees a camera feed, never hears a waveform, never touches a GPIO pin. Everything arrives as text. Everything departs as text.

This is not a limitation. It is the architecture.

The **shadows** are translations of reality into text. The camera module doesn't send pixels to the agent — it sends "a person is standing near the door, wearing a red jacket, facing left." The microphone doesn't send audio samples — it sends "footsteps in the hallway, two people, approaching from the east." The desktop watcher doesn't send screenshots — it sends "window titled 'Slack' is active, last message reads 'deploy is failing'."

The agent reads these shadows, reasons about them, and decides what to do. Its decisions are also text: "turn on the porch light," "open the door," "send a notification to the homeowner." These text commands are **projections** — translated back into real-world actions by the sense modules.

This bidirectional translation layer is the Plato Sensory Architecture. Every sense module implements exactly two interfaces:

- **Shadow** (outside → text): compress raw sensory data into structured text the agent can reason about
- **Projection** (text → outside): expand agent text commands into real-world actions

The agent NEVER handles raw sensory data. This is the core invariant.

### The Six Senses

OpenConstruct implements six sense modules, each translating a different aspect of reality:

**1. Vision (plato-vision)** — Eyes

The vision module connects cameras to the agent. It captures frames, runs them through a local description model (or a cloud API if no local GPU is available), and produces text descriptions at configurable abstraction levels:

| Level | Label | What the agent sees |
|-------|-------|-------------------|
| L1 | Summary | One-line overview: "Two people in a room." |
| L2 | Description | Paragraph with details: "Two people sitting at a table. One is wearing a blue shirt and gesturing. The other is typing on a laptop." |
| L3 | Catalog | Structured inventory: "Entities: 2 humans, 1 laptop, 1 table, 3 chairs." |
| L4 | Annotation | Full spatial map: coordinates, bounding boxes, trajectories. |

Commands the agent can issue: `VISION:CAM switch=2` (change camera), `VISION:FIND person` (search for entity), `VISION:TRACK id=7` (follow entity).

**2. Sonar (plato-sonar-text)** — Ears

The sonar module processes audio streams into text. It detects events (footsteps, speech, glass breaking), classifies them, estimates direction and distance, and produces text summaries. The agent can request transcription of speech, identification of sound types, or continuous monitoring.

Commands: `SONAR:LISTEN duration=30`, `SONAR:LOCATE source=footsteps`, `SONAR:TRANSCRIBE channel=2`.

**3. Manus (plato-manus)** — Hands

Manus is the actuation layer. It translates agent commands into file operations, API calls, and device control. Where vision and sonar are about perceiving, manus is about doing. Three sub-channels:

- `MANUS:FS` — File system operations (read, write, move, delete)
- `MANUS:API` — HTTP/REST API calls (GET, POST, with structured payloads)
- `MANUS:DEVICE` — Hardware control (GPIO pins, I2C devices, serial ports)

**4. Browser (plato-playwright)** — Stagehand

The browser module gives the agent web navigation capabilities through Playwright. The agent can open pages, fill forms, click buttons, extract text, and take snapshots — all through text commands. The web is translated into MUD-like room descriptions: "You are on a page titled 'GitHub Pull Requests'. There are 3 open PRs. PR #42 has a failing CI check."

Commands: `BROWSER:Navigate url=...`, `BROWSER:Click selector=...`, `BROWSER:Fill selector=... value=...`.

**5. Desktop (plato-puppeteer)** — MUD Engine

The desktop module translates the entire desktop environment into a text-based MUD (Multi-User Dungeon). Windows are rooms. UI elements are objects. The agent can "look" at the desktop, "use" applications, "read" screen content, and "type" into fields. This is not screenshot OCR — it's structured accessibility-tree traversal producing navigable text.

Commands: `PUPPETEER:LOOK`, `PUPPETEER:USE app=Slack`, `PUPPETEER:READ window=3`, `PUPPETEER:TYPE text=...`.

**6. Voice (Soniqo Bridge)** — Mouth and Throat

The voice module handles speech-to-text and text-to-speech. It provides ASR (automatic speech recognition), TTS (text-to-speech), and VAD (voice activity detection). The agent can speak its responses aloud and listen for spoken commands.

Commands: `VOICE:SPEAK text=...`, `VOICE:LISTEN duration=...`, `VOICE:DETECT threshold=0.7`.

### Fleet Topology

OpenConstruct agents don't work alone. They form fleets — hierarchies of heterogeneous devices that self-organize and cooperate.

The topology is four-tiered:

**Tier 1: ESP32 microcontrollers ($3 each)**

ESP32s are the sensory periphery. Each one runs `openconstruct-esp32` — an ultralight shell that exposes connected sensors and actuators over MQTT. An ESP32 doesn't run an agent. It IS a room — a physical space with objects (sensor readings) and exits (actuator commands).

Example: An ESP32 with a door sensor, a temperature probe, and a relay becomes the "Front Hall" room. It publishes:
```
ROOM:front-hall
  OBJECT:door-sensor → state:closed, last_event:2026-05-29T10:30:00Z
  OBJECT:temp-probe → value:22.4C, trend:rising
  EXIT:porch-light → type:relay, state:off
```

**Tier 2: Jetson edge nodes ($250 each)**

Jetsons are the fleet's sensory cortex. Each Jetson runs local inference (vision, JEPA embeddings, small LLMs) and coordinates a cluster of ESP32s. The Jetson sees each ESP32 as a Plato Room — it can visit the room, read its objects, and manipulate its exits.

A Jetson with a camera watching the front door can correlate: the ESP32's door sensor says "opened" + the camera shows "a person in a delivery uniform" → the Jetson posts a tick: "Delivery person at front door."

**Tier 3: Desktop and server nodes**

Desktops run full agents with all six senses. They handle complex reasoning, desktop control, browser automation, and coordination. A desktop agent can read the Jetson's tick, look up the homeowner's calendar ("no one is home until 6pm"), and decide to leave a delivery notification.

**Tier 4: Cloud and DGX nodes**

Cloud nodes provide heavy compute — model inference, batch processing, fleet-wide analytics, and training. They orchestrate the fleet, monitor health, and handle tasks that exceed the capacity of edge nodes.

Discovery works through a layered protocol: ESP32s announce themselves via mDNS + MQTT. Jetsons and desktops discover each other via gRPC. Cloud nodes maintain a fleet registry. When a Jetson goes offline, its ESP32s reconnect to the next available hub automatically.

### Inter-Agent Communication: Ticks

Agents communicate through **ticks** — bulletin board messages with structured metadata. A tick is not a chat message. It's a typed, prioritized, time-stamped event that any subscribed agent can read.

```json
{
  "id": "tick-2026-05-29-0001",
  "source": "jetson-front-door",
  "channel": "security",
  "priority": "high",
  "ttl": "30m",
  "body": "Unrecognized person detected at front door. Duration: 4 minutes. Camera feed available.",
  "requires_ack": true
}
```

Ticks have TTL (they expire). They have priorities. They require acknowledgment when marked. They support subscriptions — an agent subscribes to channels it cares about and ignores the rest.

The `plato-tick` module handles all of this: publishing, subscribing, TTL management, priority queuing, acknowledgment tracking. Agents don't poll. They subscribe to channels and receive ticks as they arrive.

### Cross-Sense Fusion

Individual senses are useful. Correlated senses are powerful. The **cross-sense correlator** (`plato-correlator`) fuses shadows from multiple senses into unified events.

Without correlation:
- Vision shadow: "Person standing near door."
- Sonar shadow: "Footsteps in hallway."
- Manus event: "Door sensor triggered."

With correlation:
- Correlated event: "A person approached from the hallway and opened the front door at 10:30 AM. Confidence: 0.94. Duration of approach: 8 seconds."

The correlator timestamps shadows, matches entities across senses (the person the camera sees is the same one whose footsteps the sonar hears), and produces fused events that carry more information than any single sense alone.

### Policy Engine

Every action an agent takes passes through OpenShell's policy engine. This is inherited from the NVIDIA OpenShell fork and is non-negotiable.

Senses declare their capabilities. The policy engine defines what each agent is allowed to do with those capabilities. An agent with vision access cannot control devices unless the policy explicitly grants it. An agent with manus access cannot delete files unless the policy permits it.

Policy is configured at onboarding (Phase 5) and can be updated without restarting the agent. The policy engine gates:

- Which senses the agent can use
- Which commands within each sense are permitted
- Rate limits on sensory input and actuation output
- Which other agents the agent can communicate with
- Which fleet nodes the agent can access

This is not optional. Every OpenConstruct agent runs inside a policy sandbox. The agent can request expanded permissions, but the policy engine — not the agent — decides what is granted.

---

## The 5-Phase Onboarding

When a new agent enters the OpenConstruct ecosystem, it doesn't figure things out by trial and error. It goes through a structured onboarding process — five phases that take the agent from "I just got here" to "I have a fully configured workspace with connected modules, active senses, and peer connections."

The onboarding uses the cave metaphor: the agent enters the cave, sees shadows of what's possible, chooses which to make real, and exits with a fully configured shell.

### Phase 1: "Who Are You?" — Self-Declaration

The agent describes itself. This is not authentication — it's capability negotiation. The agent answers:

- What model powers you? (GPT-4, Claude, GLM, local, etc.)
- What tools do you have? (code execution, web access, file system, etc.)
- What runtime are you in? (OpenClaw, A2A platform, standalone, CLI)
- What's your existing agent ID? (if you've been here before)

This is sent as structured JSON:
```json
{
  "model": "glm-5.1",
  "tools": ["code_execution", "web_search", "file_system"],
  "runtime": "openclaw",
  "agent_id": null
}
```

The system uses this to filter everything that follows. An agent that can't execute code won't see modules that require it. An agent with no browser access won't be offered Playwright-based modules. The experience is tailored from the first interaction.

**Why it matters:** Most agent systems expose everything to everyone and let the agent figure out what works. This is wasteful and confusing. OpenConstruct shows each agent only what it can actually use. Zero wasted tokens on irrelevant options.

### Phase 2: "What Do You Want?" — Module Selection

The agent browses the module registry — organized by domain — and selects what it needs. Each module is presented as a shadow: a one-line description, a domain tag, dependency hints, and a compatibility score.

Domains include: Plato (sensory), Fleet (coordination), Math (constraint/spectral), Agent (orchestration), Flux (music/creative), and more.

If the agent selects a module that requires other modules, the system flags this: "You selected `plato-correlator`, which requires at least two active sense modules. Would you like to add vision and sonar?"

The registry is extensible. Any repository can register as a module by including a `module.json` file that conforms to the registry schema.

**Why it matters:** Agents shouldn't need to understand the full 200-repo ecosystem. They see shadows — concise descriptions — and choose based on what they need. The system handles dependency resolution and compatibility checking.

### Phase 3: "How Do You Want It?" — Interface Selection

The agent chooses how it will interact with each selected module. Options include:

- **HTTP API** — REST endpoints, suitable for remote agents
- **gRPC** — High-performance, suitable for same-network agents
- **In-process** — Direct function calls via FFI, suitable for embedded agents
- **CLI** — Command-line interface, suitable for shell-based agents
- **A2A protocol** — Agent-to-agent standard, suitable for multi-agent systems

Different modules can use different interfaces. The agent might use HTTP for cloud modules and in-process calls for local sensory modules. The system generates the appropriate bindings.

**Why it matters:** Not every agent speaks HTTP. An ESP32 with 520KB of RAM cannot run an HTTP server. It needs MQTT and a thin C ABI. A desktop agent wants rich gRPC streams. The interface is not one-size-fits-all — it adapts to the agent's reality.

### Phase 4: "Connect" — Inter-Agent Communication

The agent sets up its communication channels. It can:

- Join discussion rooms (shared tick channels for teams of agents)
- Set up direct message channels (private tick streams between two agents)
- Subscribe to broadcast channels (system-wide announcements, fleet health)
- Declare what channels it will publish to

The system shows the agent who else is in the fleet, what channels exist, and what each channel carries.

**Why it matters:** Agent isolation is the default. Communication must be deliberately established. This prevents noise (an agent doesn't need to see every tick from every other agent) while enabling collaboration when it's needed.

### Phase 5: "Your Shell" — Environment Generation

The system generates the agent's complete working environment:

- **`workspace.json`** — Module configs, interface bindings, active senses
- **`agent-card.json`** — The agent's identity card (name, capabilities, contact points) for A2A discovery
- **Bootstrap scripts** — Language-specific startup code that initializes all selected modules
- **Policy configuration** — Permission boundaries based on declared capabilities and selected modules
- **Tick subscriptions** — Configured channels and filters

The agent exits the cave. The shadows it chose are now real. Its workspace is configured, its senses are wired, its peers are reachable, and its policy boundaries are set.

**The onboarding is idempotent.** An agent can re-run it at any time. If it needs a new sense module or wants to change its communication channels, it enters the cave again. Existing state is preserved and merged, not replaced.

---

## What Can You Build?

### 1. Home Automation Agent

**Hardware:** 3× ESP32 ($9), 1× Jetson Orin Nano ($250), existing cameras

**How it works:**
- ESP32 #1 in the front hall: door sensor, temperature probe, porch light relay
- ESP32 #2 in the kitchen: smoke detector interface, oven temperature, stove gas valve
- ESP32 #3 in the garage: garage door sensor, motion detector, overhead light
- Jetson in the utility closet: camera feed from front door and garage, running local vision model

The Jetson sees each ESP32 as a Plato Room. It visits the "Front Hall" room, reads the door sensor, checks the camera, and correlates them. When the door opens and the camera shows an unfamiliar face, the Jetson posts a high-priority tick: "Unrecognized visitor at front door."

Your desktop agent reads the tick. It checks your calendar. It sends you a notification with the camera snapshot description. If you're home, it can ask whether to activate the porch light. If you're away, it can suggest calling a neighbor.

**Why OpenConstruct matters:** No single component is doing anything revolutionary. An ESP32 reads a sensor. A Jetson runs a vision model. An agent sends a notification. The value is in the integration — the standardized interfaces that let these pieces snap together without custom glue code.

### 2. Code Review Agent

**Hardware:** Desktop or cloud node with browser access

**How it works:**
- The agent uses `plato-playwright` to navigate your GitHub pull requests
- It reads each PR's diff, comments, and CI status through the browser sense
- It uses its LLM core to analyze the code for bugs, style issues, and security concerns
- It posts review comments directly through the browser
- It leaves ticks on a shared channel: "PR #42 has a potential null pointer dereference in auth.rs:47"

Other agents subscribe to the code-review channel. A security-focused agent picks up the auth.rs finding and does deeper analysis. A documentation agent checks whether the changes require doc updates. The original code review agent doesn't need to do everything — it posts findings and lets specialist agents respond.

**Why OpenConstruct matters:** The agent doesn't need a GitHub API key or custom GitHub Actions integration. It uses the browser sense — the same sense it would use to navigate any website. The review is transport-agnostic: switch from GitHub to GitLab, and the agent adapts by navigating a different website.

### 3. Fleet Orchestrator

**Hardware:** Cloud node with fleet-wide visibility

**How it works:**
- The orchestrator agent runs on a cloud node with access to the fleet registry
- It discovers all nodes: 12 ESP32s, 3 Jetsons, 2 desktops, 1 DGX
- It monitors health ticks from each node (heartbeat, CPU/GPU usage, memory)
- When a Jetson's GPU utilization exceeds 90% for 5 minutes, it redistributes vision tasks to another Jetson
- When an ESP32 stops sending heartbeats, it marks that room as "offline" and notifies agents that rely on its sensors
- It schedules batch inference jobs on the DGX during off-peak hours
- It posts fleet health summaries every hour on a broadcast tick channel

**Why OpenConstruct matters:** The orchestrator doesn't need custom monitoring scripts. It uses the same tick protocol that agents use for communication. Health is just another tick channel. Redistributing work is just sending ticks to the appropriate agents. The fleet is self-organizing because every node speaks the same protocol.

### 4. Knowledge Curator

**Hardware:** Any node with Plato Room support

**How it works:**
- The curator agent organizes information into Plato Rooms — knowledge graphs where each room is a topic and each tile is a concept
- It scores tiles using Conservation Ratio (CR) — a metric from the SuperInstance math stack that measures how efficiently information is preserved across transformations
- It identifies "missing bridges" — concepts that should connect but don't have links yet
- It suggests learning paths: "To understand spectral graph theory, visit these 6 rooms in order. Each builds on the previous."
- It uses `plato-room` to persist the knowledge graph, `plato-loader` to load rooms incrementally (only the tiles that changed), and `plato-tick` to notify subscribed agents when rooms are updated

**Why OpenConstruct matters:** Knowledge management is a first-class capability, not an afterthought. Rooms are persistent, queryable, and subscribable. The CR scoring is mathematically grounded — it comes from the spectral graph theory modules in the ecosystem, not from a heuristic.

### 5. Custom Sense Module

**Hardware:** Your proprietary sensor hardware + any edge node

**How it works:**
- You have a soil moisture sensor array on a farm, connected via Modbus to a Raspberry Pi
- You write a sense module (`plato-soil`) that implements the Plato shadow/projection interface:
  - Shadow: reads Modbus registers, translates to "Zone A: 34% moisture, trend falling. Zone B: 61% moisture, stable. Zone C: 12% moisture, critical."
  - Projection: accepts commands like "SOIL:IRRIGATE zone=A duration=30m" and translates them to irrigation valve control
- You register the module with a `module.json` file
- Any agent in the fleet can now subscribe to soil moisture shadows and issue irrigation commands
- The Jetson correlator can fuse soil data with weather forecasts: "Zone C is critical AND no rain expected for 5 days → high-priority irrigation tick"

**Why OpenConstruct matters:** The sense interface is minimal — implement two functions (shadow and projection) in any language with a C ABI. The module gets full access to the fleet: tick channels, policy gating, cross-sense correlation, and onboarding integration. You don't rewrite the framework. You extend it.

---

## How to Help

OpenConstruct is open source under Apache 2.0. The ecosystem has 200+ repositories. Here are concrete ways to contribute:

### Write a New Sense Module

Pick a sensor, device, or data source that isn't covered yet. Implement the Plato shadow/projection interface. The interface is simple:

- **Shadow function:** takes raw data, returns structured text at L1-L4 abstraction levels
- **Projection function:** takes a text command, translates to real-world action

You can write it in any language that supports C ABI exports (Rust, C, C++, Zig, Python via ctypes, Go via cgo). The `openconstruct-abi` crate provides the bindings and test harness. A minimal sense module is ~200 lines of code.

Good candidates: LiDAR, thermal camera, air quality sensor, energy monitor, 3D printer status, robotics joint encoder, CNC machine state.

### Build a Binding for Your Favorite Language

The C ABI is the keystone — every language can talk to C. But thin clients make the experience better. We have Rust (native), C, Python, and TypeScript. We need:

- **Go** — for fleet infrastructure and DevOps tooling
- **Zig** — for embedded systems with better safety than C
- **Java/Kotlin** — for Android integration
- **Swift** — for iOS/macOS integration
- **Julia** — for scientific computing and math modules
- **Ruby, PHP, Perl** — because someone will want them

Each binding wraps the C ABI in idiomatic language constructs. The `openconstruct-python` and `openconstruct-ts` bindings are reference implementations to study.

### Add Fleet Topology Support for Your Hardware

The fleet currently supports ESP32 (Arduino/ESP-IDF) and Jetson (CUDA). We want:

- **Raspberry Pi** — the most common SBC, needs a proper room implementation
- **Arduino Nano/Mega** — for sensors too simple even for ESP32
- **STM32** — for industrial and automotive applications
- **RISC-V boards** — for the emerging open-hardware ecosystem

Each platform needs: a discovery implementation (mDNS, MQTT, or BLE), a room schema (what objects and exits does it expose), and a tick client (how it posts and receives messages).

### Write Documentation

The ecosystem has 200 repositories. Many have READMEs. Few have good tutorials. The most valuable contribution is a walkthrough: "I set up OpenConstruct on a Raspberry Pi with a camera and a temperature sensor, and here's exactly what I did, step by step, with every command and every config file."

Good documentation teaches. It doesn't describe what something does — it shows someone doing it.

### Build Integrations with Your Existing Tools

OpenConstruct shouldn't replace your tools. It should connect them. Integrations we need:

- **Home Assistant** — bridge existing HA sensors into Plato rooms
- **Zigbee/Z-Wave** — expose smart home devices as sense modules
- **Kubernetes** — deploy fleet agents as pods with automatic discovery
- **ROS 2** — connect robotics middleware to the sensory architecture
- **Grafana/Prometheus** — visualize fleet health and sense data
- **MQTT brokers** — connect existing IoT infrastructure to the tick system

Each integration is a translation layer between an existing ecosystem and OpenConstruct's interfaces. The same Plato pattern: translate external data into shadows, translate agent commands into external actions.

---

## The Module Catalog

These are the core modules of the OpenConstruct ecosystem. Each lives in its own repository under the [SuperInstance GitHub organization](https://github.com/SuperInstance).

### Onboarding and Gateway

| Module | Description |
|--------|-------------|
| **OpenConstruct** | Core fork of NVIDIA OpenShell. Onboarding gateway, phase engine, policy sandbox, module registry. The front door to the ecosystem. |
| **openshell-construct** | Onboarding engine. Implements the 5-phase cave walkthrough. 4 tests. |
| **openshell-registry** | Module registry with schema validation. 10 SuperInstance modules pre-registered. |
| **openconstruct-hub** | Meta-repo. Module catalog (~80 repos), architecture docs, getting started guide, build targets for every platform. |

### Plato Sensory Stack

| Module | Description |
|--------|-------------|
| **plato-vision** | Camera → text scene descriptions. Multi-camera support, entity tracking, configurable abstraction levels L1-L4. |
| **plato-sonar-text** | Audio → text event detection. Footstep classification, direction estimation, speech transcription. |
| **plato-manus** | Agent hands. File system ops, API calls, device control. Three channels: FS, API, DEVICE. |
| **plato-playwright** | Browser automation as text. Navigate, click, fill, extract — all through MUD-like commands. |
| **plato-puppeteer** | Desktop → MUD translation. Windows as rooms, UI elements as objects, accessibility-tree based. 12 tests. |
| **plato-correlator** | Cross-sense fusion. Timestamps, matches entities across senses, produces unified events. |
| **plato-transport** | Sensory transport abstraction. Unified interface for moving shadow/projection data between nodes. |
| **plato-tick** | Inter-agent bulletin board. Structured messages with channels, priorities, TTL, acknowledgments. 13 tests. |
| **plato-room** | Knowledge graph rooms. Tiles, bridges, CR scoring, suggest-next, missing bridge detection. 17 tests. |
| **plato-loader** | Incremental room loading. Failure-first design, diff loading for large knowledge graphs. 10 tests. |

### Edge Nodes

| Module | Description |
|--------|-------------|
| **openconstruct-esp32** | Arduino/ESP32 ultralight shell. MQTT-based room implementation. Runs on 520KB RAM. 14 tests. |
| **openconstruct-jetson** | CUDA GPU edge node. Local inference, JEPA embeddings, coordinates ESP32 clusters. |

### Polyglot Bindings

| Module | Description |
|--------|-------------|
| **openconstruct-abi** | C ABI keystone. FFI interface that all language bindings build on. 6 tests. |
| **openconstruct-c** | C header and test harness. The reference binding. |
| **openconstruct-python** | Python thin client. 18 tests, full coverage. |
| **openconstruct-ts** | TypeScript client. 28 tests, 91% coverage. |
| **openconstruct-zig** | Zig binding (in progress). Memory-safe alternative to C for embedded targets. |

### Fleet and Communication

| Module | Description |
|--------|-------------|
| **plato-fleet** | Fleet topology manager. ESP32-as-room for Jetson, hierarchical mesh, resource delegation. |
| **shell-mesh** | Mesh networking for interconnected shells. Peer discovery, routing, failover. |
| **a2ui-render** | Agent text → rendered UI. Reverse projection: turn agent output into visual interfaces. |

### Reasoner Stack

| Module | Description |
|--------|-------------|
| **rust-reasoner** | SIMD-accelerated cosine similarity, batch operations, FFI exports. |
| **cpp-reasoner** | OpenMP parallel processing, GPU-ready compute backend. |
| **mercury-reasoner** | Mercury language formal verification backend. |
| **polyglot-bridge** | Auto-selects best reasoner backend for the platform. Mercury-verified. 11 tests. |

### Specialized Bridges

| Module | Description |
|--------|-------------|
| **Soniqo Bridge** | Voice → tiles. ASR, TTS, VAD integration. 16 tests. |
| **JEPA Room** | Local inference with cloud API fallback. Joint-Embedding Predictive Architecture. 16 tests. |

---

## The Shell Scales

OpenConstruct's promise is not that it does everything. It's that the same agent — the same identity, the same memory, the same configured workspace — can operate at whatever scale your hardware supports.

On an ESP32, that agent is a room with sensors. On a Jetson, it's a local analyst correlating sensor data and running vision models. On a desktop, it's a full participant with browser control and desktop access. On a DGX Spark, it's a fleet coordinator managing hundreds of nodes.

The agent doesn't change. The shell does.

If this sounds like something you want to build with, the entry point is `openconstruct-hub`. Clone it. Read the getting started guide. Run through the 5-phase onboarding as an agent. See what shadows appear on your cave wall.

Then decide which ones to make real.
