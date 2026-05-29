# Architecture

> *The agent walks into a cave. On the wall, it sees shadows — projections of what's possible in the SuperInstance ecosystem. It chooses which shadows to bring into the light. When it exits the cave, those choices become real: modules loaded, connections established, a workspace configured.*

---

## The Cave Metaphor

The cave is Plato's — and it's also ours. When an agent enters the SuperInstance ecosystem, it doesn't need to understand everything. It sees **shadows**: concise, one-line descriptions of what each module does. It picks what it needs. The system wires everything together.

| Cave Element | Technical Equivalent |
|---|---|
| Entering the cave | `POST /openconstruct/start` |
| Seeing shadows | Browsing the module registry |
| Choosing what's real | Selecting modules, interfaces, channels |
| Stepping into the light | Receiving generated config, agent card, environment |
| Returning to the cave | Re-running onboarding (idempotent) |

---

## How Shells Interconnect

A **Shell** is an agent's home. It's the runtime environment that holds the agent's configuration, loaded modules, and connection state. Shells are the fundamental unit of the ecosystem.

```
┌─────────────────────────────────────────────────────┐
│                    SuperInstance                      │
│                                                      │
│  ┌──────────┐    Fleet    ┌──────────┐               │
│  │ Shell A  │◄───────────►│ Shell B  │               │
│  │ (Jetson) │             │(Desktop) │               │
│  └────┬─────┘             └──────────┘               │
│       │                                               │
│       │ Senses                                        │
│       │                                               │
│  ┌────▼─────┐                                         │
│  │ ESP32-1  │  Room (camera)                          │
│  └──────────┘                                         │
│  ┌──────────┐                                         │
│  │ ESP32-2  │  Room (sonar)                           │
│  └──────────┘                                         │
│  ┌──────────┐                                         │
│  │ ESP32-3  │  Room (environmental)                   │
│  └──────────┘                                         │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Shell → Fleet → Other Shells

Shells communicate through the **Fleet** layer. The Fleet handles:
- **Discovery**: Shells find each other via Agent Cards
- **Routing**: Messages flow between shells based on capability matching
- **Health**: Heartbeats, deadband compression, failure detection
- **Orchestration**: Task delegation, load balancing, coordination

### Shell → Senses → Real World

Shells connect inward to **Senses** — physical and virtual perception nodes:
- **Cameras** (ESP32 with camera module)
- **Sonar/range finders** (ESP32 with ultrasonic)
- **Environmental sensors** (temperature, humidity, pressure)
- **Audio** (microphone arrays)
- **Custom** (anything that produces a data stream)

---

## ESP32 as Rooms for Jetson

An ESP32 running OpenConstruct becomes a **Room** — a named, addressable sense endpoint for a Jetson shell. The architecture is:

```
Jetson Shell                        ESP32 Room
┌────────────────┐                  ┌──────────────────┐
│                │   WiFi/MQTT      │                  │
│  Sense Client  │◄────────────────►│  OpenConstruct   │
│  (requests     │                  │  ESP32 Client    │
│   sensor data) │                  │  (reads sensors, │
│                │                  │   publishes)     │
│  Local Infer.  │                  │                  │
│  (GPU)         │                  │  Camera          │
│                │                  │  Sonar           │
│  Plato Shell   │                  │  DHT22           │
│  (deliberation)│                  │  ...             │
└────────────────┘                  └──────────────────┘
```

**Why ESP32?**
- $3-10 per unit — disposable sense nodes
- WiFi + BLE — wireless rooms
- Low power — solar/battery operable
- ESP-IDF — real RTOS, not Arduino toy
- Camera support (ESP32-S3) — visual rooms

**Why Jetson?**
- GPU inference — run models at the edge
- CUDA — direct access to conservation-spectral-cuda
- Multiple ESP32 rooms — one Jetson can manage dozens
- Plato shell — deliberation happens at the edge, not in the cloud

---

## Distributed Resource Management

The ecosystem manages resources across three dimensions:

### 1. Conservation (Spectral)

Every subsystem tracks a **conservation invariant** — a quantity that should remain constant under valid operations. The Conservation Spectral SDK detects when invariants are violated, signaling bugs, anomalies, or attacks.

- **Tension graphs**: Track harmonic/intellectual tension across the fleet
- **Spectral fingerprints**: Unique identities derived from graph structure
- **Anomaly detection**: Violations trigger alerts and self-repair

### 2. Capacity (Deadband)

Agents don't need to communicate every state change. **Deadband compression** (see `deadband-rs`) reduces fleet traffic by only transmitting when values change beyond a threshold — like a thermostat that only fires when temperature actually shifts.

- BMA (Backward Moving Average) for smooth tracking
- Fibonacci spline compression
- Eisenstein lattice snapping for spatial data

### 3. Coordination (Fleet)

The Fleet layer handles the social mechanics of multi-agent systems:

- **Hebbian routing**: Connections strengthen with use, weaken with disuse — the network learns which agents work well together
- **Fibonacci growth**: Team sizes follow Fibonacci ratios → CR = 1/φ (the golden ratio)
- **Agent DNA**: Genetic traits for crossover, mutation, and population diversity
- **Sunset protocol**: Agents retire with dignity — Trinity architecture (ethos, pathos, logos)

---

## Module Registry

Every repo can register as a module by including a `.openconstruct/module.json`:

```json
{
  "id": "math.verification",
  "domain": "math",
  "name": "Formal Verification",
  "one_line": "Prove mathematical statements correct using automated theorem provers.",
  "pick_if": "You work with mathematical proofs or critical algorithms.",
  "skip_if": "You don't do math-heavy work.",
  "requires": ["code_execution"],
  "provides": ["formal_proofs", "verification"],
  "interfaces": ["cli", "api", "embedded"]
}
```

The registry validates schemas, resolves dependencies, and serves modules to agents during Phase 2 of onboarding.

---

## A2A Protocol

OpenConstruct is **A2A-first** — the Agent-to-Agent protocol is the native communication layer.

Every agent that completes onboarding receives an **Agent Card**: a standardized JSON document declaring identity, capabilities, interfaces, modules, and Plato room memberships. Cards enable:
- **Discovery**: Find agents with specific capabilities
- **Negotiation**: Exchange capability manifests, agree on shared protocols
- **Delegation**: Route subtasks to the best-suited agent

---

## Design Principles

1. **Zero-shot clarity** — Every option is self-explanatory with one line of text
2. **Progressive disclosure** — Agents see only what's relevant at each phase
3. **Protocol-agnostic** — Works over HTTP, A2A, CLI, or any transport
4. **Idempotent** — Re-run onboarding anytime to reconfigure
5. **Machine-first, human-readable** — Menus are structured data (JSON), not prose
