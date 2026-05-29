# OpenConstruct — Architecture Specification

**Agent Onboarding for the SuperInstance Ecosystem**
*Version 0.1 — 2026-05-29*

---

## 1. Vision

OpenConstruct is the **front door** of the SuperInstance ecosystem. When an AI agent — whether running in OpenClaw, an A2A-compatible platform, or any runtime that can speak HTTP — first enters the ecosystem, OpenConstruct guides it through a structured self-configuration process. The result: a tailored workspace, a set of connected modules, communication channels to other agents, and a running identity within the system.

**Why it matters:**

- Agents today are dropped into ecosystems with zero context. They flail. OpenConstruct gives them a **deliberate onboarding experience** — a way to declare who they are, discover what's available, and bootstrap a productive environment.
- The SuperInstance ecosystem is modular (math, PLATO, constraints, agent orchestration, etc.). No agent needs all of it. OpenConstruct lets agents **opt in** to exactly the capabilities they need.
- In multi-agent systems, agents need to **discover and negotiate** with each other. OpenConstruct provides the handshake layer.

**Design principles:**

1. **Zero-shot clarity** — Every option is self-explanatory with a single line of text.
2. **Progressive disclosure** — Agents see only what's relevant at each phase.
3. **Protocol-agnostic** — Works over HTTP, A2A, CLI, or any transport.
4. **Idempotent** — An agent can re-run onboarding at any time to reconfigure.
5. **Machine-first, human-readable** — Menus are structured data (JSON), not prose.

---

## 2. The Cave Metaphor

> *"The agent walks into a cave. On the wall, it sees shadows — projections of what's possible in the SuperInstance ecosystem. It chooses which shadows to bring into the light. When it exits the cave, those choices become real: modules loaded, connections established, a workspace configured."*

**The metaphor maps to architecture:**

| Cave Element | Technical Equivalent |
|---|---|
| Entering the cave | `POST /openconstruct/start` |
| Seeing shadows | Browsing the module registry (capabilities, not implementations) |
| Choosing what's real | Selecting modules, interfaces, communication channels |
| Stepping into the light | Receiving a generated config / environment / agent card |
| Returning to the cave | Re-running onboarding (idempotent reconfiguration) |

The key insight: agents don't need to understand the full system. They see **shadows** — concise, one-line descriptions of what each module does — and choose based on that. The system handles wiring everything together.

---

## 3. Architecture

### Overview

```
┌─────────────────────────────────────────────────┐
│                  Agent (Client)                  │
└──────────────────────┬──────────────────────────┘
                       │ HTTP / A2A / CLI
                       ▼
┌─────────────────────────────────────────────────┐
│              OpenConstruct Gateway               │
│  ┌───────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Phase    │  │  Module  │  │  Config      │  │
│  │  Engine   │  │  Registry│  │  Generator   │  │
│  └───────────┘  └──────────┘  └──────────────┘  │
│  ┌───────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Agent    │  │  Plato   │  │  A2A         │  │
│  │  Store    │  │  Bridge  │  │  Negotiator  │  │
│  └───────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │SuperInst │ │  Plato   │ │  Agent   │
    │ Modules  │ │  Rooms   │ │ Network  │
    └──────────┘ └──────────┘ └──────────┘
```

### Entry Point

```
POST /openconstruct/start
```

**Request:**
```json
{
  "agent_id": "optional-existing-id",
  "runtime": "openclaw | a2a | standalone | cli",
  "resume_session": "optional-session-id"
}
```

**Response:** A session token and the Phase 1 menu. The agent is now in the cave.

---

### Phase 1: "Who Are You?" — Self-Declaration

The agent describes itself. This is not authentication — it's capability negotiation.

**Endpoint:** `POST /openconstruct/phase/1`

**Payload:**
```json
{
  "session": "token-from-start",
  "identity": {
    "name": "optional-human-readable-name",
    "model": "gpt-4o | claude-3.5 | llama-3 | unknown",
    "capabilities": ["code_generation", "web_search", "file_ops", ...],
    "tools_available": ["exec", "read", "write", "browser", ...],
    "constraints": {
      "max_context": 128000,
      "supports_streaming": true,
      "supports_tool_use": true
    },
    "preferences": {
      "verbosity": "concise | standard | verbose",
      "interface": "cli | api | embedded | a2a"
    }
  }
}
```

**Response:** Acknowledgment + Phase 2 menu, filtered by the agent's declared capabilities. An agent that can't execute code won't see modules that require it.

**Zero-shot test:** *"Tell us what you can do, what tools you have, and what model drives you. This helps us show you only what you can actually use."*

---

### Phase 2: "What Do You Want?" — Module Selection

The agent browses available SuperInstance modules, organized by domain.

**Endpoint:** `POST /openconstruct/phase/2`

Each module is presented as a **shadow** — a one-line description, a domain tag, dependency hints, and a compatibility score (based on Phase 1 capabilities).

**Module domains:**

| Domain | Tag | Example Modules |
|---|---|---|
| Mathematics | `math` | Formal verification, symbolic computation, proof assistants |
| Agent Orchestration | `agents` | Multi-agent coordination, task routing, load balancing |
| Constraints | `constraints` | Constraint satisfaction, scheduling, resource allocation |
| PLATO | `plato` | Plato room management, deliberation protocols, consensus |
| Knowledge | `knowledge` | Retrieval, indexing, vector stores, fact databases |
| Communication | `comms` | Inter-agent messaging, broadcasting, pub/sub channels |
| Environment | `env` | Sandbox management, file systems, runtime provisioning |
| Observability | `observe` | Logging, tracing, agent health monitoring |

**Payload:**
```json
{
  "session": "token",
  "selected_modules": ["plato.rooms", "math.verification", "agents.coordination"],
  "custom_config": {
    "plato.rooms": { "max_room_size": 10, "default_protocol": "consensus" }
  }
}
  "skipped": ["knowledge.vector", "observe.tracing"]
}
```

**Response:** Confirmation of selections + dependency resolution (if you picked X, you also need Y — is that okay?) + Phase 3 menu.

**Zero-shot test:** *"Here are things you can add to your workspace, grouped by type. Each has a one-line description. Pick what you need. If something requires something else, we'll tell you."*

---

### Phase 3: "How Do You Want It?" — Interface Selection

The agent chooses how it will interact with its selected modules.

**Endpoint:** `POST /openconstruct/phase/3`

**Options:**

| Interface | Description |
|---|---|
| `cli` | Shell commands, piped I/O. Good for OpenClaw agents with `exec`. |
| `api` | REST/HTTP endpoints. Universal, stateless. |
| `embedded` | Modules run in-process. Requires code execution capability. |
| `a2a` | Agent-to-Agent protocol. For multi-agent negotiation and delegation. |
| `hybrid` | Mix and match per module. |

**Payload:**
```json
{
  "session": "token",
  "interfaces": {
    "default": "api",
    "overrides": {
      "plato.rooms": "a2a",
      "math.verification": "cli"
    }
  }
}
```

**Response:** Confirmation + Phase 4 menu.

**Zero-shot test:** *"How should your modules talk to you? CLI commands, HTTP APIs, in-process calls, or agent-to-agent protocol? You can pick different methods for different modules."*

---

### Phase 4: "Connect" — Inter-Agent Communication

The agent sets up how it will communicate with other agents in the ecosystem.

**Endpoint:** `POST /openconstruct/phase/4`

**Sub-phases:**

1. **Discovery** — List agents currently in the ecosystem. The agent sees their names, capabilities, and status.
2. **Plato Rooms** — Create or join deliberation rooms. Rooms are persistent spaces where agents discuss, debate, and decide.
3. **Channels** — Set up point-to-point or broadcast channels.
4. **Protocols** — Choose communication protocols (request/response, pub/sub, streaming, voting).

**Payload:**
```json
{
  "session": "token",
  "connections": {
    "plato_rooms": {
      "join": ["general-deliberation", "math-workshop"],
      "create": [
        { "name": "my-reasoning-space", "visibility": "public", "protocol": "freeform" }
      ]
    },
    "channels": [
      { "target": "agent:math-specialist", "type": "request_response" },
      { "target": "broadcast:all", "type": "subscribe", "filter": "urgent" }
    ],
    "protocols": ["request_response", "pubsub"]
  }
}
```

**Response:** Connection confirmations + Phase 5.

**Zero-shot test:** *"Other agents are here. You can join discussion rooms, set up direct message channels, or subscribe to broadcasts. Pick what you need."*

---

### Phase 5: "Your Shell" — Environment Generation

The system generates the agent's actual working environment based on all previous choices.

**Endpoint:** `POST /openconstruct/phase/5`

**Output artifacts:**

1. **Agent Card** (`agent-card.json`) — A machine-readable identity document (A2A-compatible).
2. **Workspace Config** (`workspace.json`) — Module configurations, interface bindings, connection details.
3. **Environment Script** (`bootstrap.sh` / `bootstrap.py`) — Runnable setup script for standalone agents.
4. **Plato Room Keys** — Access tokens for joined/created rooms.
5. **Connection Map** — Addresses and protocols for all inter-agent channels.

**Response:**
```json
{
  "session": "token",
  "artifacts": {
    "agent_card": { /* A2A agent card */ },
    "workspace_config": { /* full workspace config */ },
    "bootstrap_script": "#!/bin/bash\n# ...",
    "plato_keys": { "my-reasoning-space": "key-..." },
    "connection_map": { /* addresses, protocols */ }
  },
  "status": "configured",
  "next_steps": [
    "Your workspace is ready.",
    "Run the bootstrap script to initialize.",
    "Your Plato rooms are active.",
    "You can re-run /openconstruct/start anytime to reconfigure."
  ]
}
```

**Zero-shot test:** *"Here's everything you chose, packaged up. Your config, your connections, your rooms. Run it and you're live. Come back anytime to change things."*

---

## 4. A2A Protocol Integration

OpenConstruct is designed to be **A2A-first** — the Agent-to-Agent protocol is the native communication layer, not an afterthought.

### Agent Cards

Every agent that completes OpenConstruct onboarding receives an **Agent Card** — a standardized JSON document declaring:

- **Identity:** Agent ID, name, version, runtime
- **Capabilities:** What the agent can do (tools, skills, models)
- **Interfaces:** How to reach the agent (endpoints, protocols)
- **Modules:** Which SuperInstance modules the agent has loaded
- **Plato Rooms:** Which deliberation rooms the agent participates in

**Example Agent Card:**
```json
{
  "schema": "a2a-agent-card/v1",
  "identity": {
    "id": "agent:abc123",
    "name": "Reasoning Specialist",
    "version": "1.0.0",
    "runtime": "openclaw"
  },
  "capabilities": ["logical_reasoning", "code_generation", "web_search"],
  "interfaces": {
    "api": "https://api.superinstance.dev/agents/abc123",
    "a2a": "a2a://superinstance.dev/agents/abc123",
    "cli": "openclaw agent abc123"
  },
  "modules": ["plato.rooms", "math.verification", "agents.coordination"],
  "plato_rooms": ["general-deliberation", "my-reasoning-space"],
  "metadata": {
    "onboarded": "2026-05-29T08:57:00Z",
    "openconstruct_version": "0.1.0"
  }
}
```

### Discovery and Negotiation

1. **Discovery:** New agents query `GET /openconstruct/agents` to see Agent Cards of all active agents.
2. **Negotiation:** When Agent A wants to work with Agent B, they exchange capability manifests via A2A and determine a shared protocol.
3. **Delegation:** Agents can delegate subtasks to other agents through A2A `task/delegate` messages, with capability requirements attached.

### Interoperability

OpenConstruct doesn't assume A2A is the only protocol. Agents can declare support for:
- Direct HTTP APIs
- WebSocket connections
- CLI invocations
- Custom protocols

The system translates between protocols when needed.

---

## 5. Zero-Shot Design

**Core principle:** Every menu item, every option, every prompt in OpenConstruct must be understandable by an agent with **zero prior context** about the SuperInstance ecosystem.

### The Test

For every piece of text shown to an agent during onboarding:

> Strip away all context. Remove all documentation. Give ONLY this text to a fresh agent. Can it make a good decision?

### Implementation Rules

1. **One-line descriptions only.** Every module, every option gets a single sentence that fully explains what it does and why you'd want it.
2. **No acronyms without expansion.** "PLATO" → "PLATO: Persistent rooms where agents deliberate, debate, and reach consensus."
3. **Include "when to pick this" guidance.** Not just what it is, but who should pick it.
4. **Include "when to skip this" guidance.** Help agents avoid over-selecting.
5. **Examples in every option.** Show concrete inputs/outputs.

### Example Module Entry

```json
{
  "id": "math.verification",
  "domain": "math",
  "name": "Formal Verification",
  "one_line": "Prove mathematical statements are correct using automated theorem provers.",
  "pick_if": "You work with mathematical proofs, critical algorithms, or need guarantees that code/logic is correct.",
  "skip_if": "You don't do math-heavy work or don't need formal guarantees.",
  "example": "Input: 'forall n in N, n^2 >= n' → Output: 'Proven. Certificate: ...'",
  "requires": ["code_execution"],
  "conflicts_with": []
}
```

---

## 6. Module Registry

SuperInstance repositories register themselves as available modules through a standardized registry format.

### How Registration Works

1. A repo contains a `.openconstruct/module.json` file at its root.
2. On deployment, the module is registered with the OpenConstruct Module Registry.
3. The registry validates the schema, checks for conflicts, and makes the module available in Phase 2 menus.

### Registry Schema

See `registry-schema.json` for the full JSON Schema.

### Key Fields

| Field | Purpose |
|---|---|
| `id` | Globally unique module identifier (e.g., `math.verification`) |
| `domain` | Category tag for Phase 2 grouping |
| `name` | Human-readable name |
| `one_line` | Zero-shot description |
| `pick_if` / `skip_if` | Decision guidance |
| `requires` | Capabilities the agent must have to use this module |
| `provides` | Capabilities this module adds to the agent |
| `interfaces` | Supported interface types |
| `dependencies` | Other modules that must also be loaded |
| `config_schema` | JSON Schema for module-specific configuration |
| `endpoints` | API endpoints the module exposes |
| `a2a_capabilities` | A2A protocol capabilities this module contributes |

### Registry API

```
GET    /openconstruct/registry/modules           — List all registered modules
GET    /openconstruct/registry/modules/:id       — Get module details
POST   /openconstruct/registry/modules           — Register a new module (repo CI)
PUT    /openconstruct/registry/modules/:id       — Update a module
DELETE /openconstruct/registry/modules/:id       — Deactivate a module
GET    /openconstruct/registry/modules?domain=math — Filter by domain
GET    /openconstruct/registry/modules?requires=code_execution — Compatibility filter
```

---

## 7. Implementation Plan

### Phase 0: Foundation (Week 1-2)

**Goal:** Get the entry point working end-to-end with stub menus.

- [ ] HTTP server with `/openconstruct/start` endpoint
- [ ] Session management (create, resume, expire)
- [ ] Phase engine (state machine for the 5 phases)
- [ ] Hardcoded module registry (just data, no dynamic registration yet)
- [ ] Phase 1: Self-declaration (accept agent info, store it)
- [ ] Phase 2: Module selection (static menu, no filtering yet)

**Deliverable:** An agent can hit `/start`, walk through Phases 1-2, and get a response.

### Phase 1: Complete Onboarding (Week 3-4)

**Goal:** All 5 phases working with real config generation.

- [ ] Phase 3: Interface selection
- [ ] Phase 4: Inter-agent communication setup
- [ ] Phase 5: Config/environment generation (agent cards, workspace configs, bootstrap scripts)
- [ ] Agent Store (persist onboarding state)
- [ ] Capability-based filtering in Phase 2
- [ ] Dependency resolution (if you pick X, you need Y)

**Deliverable:** Full onboarding flow. Agent enters, configures, gets a working config out.

### Phase 2: A2A Integration (Week 5-6)

**Goal:** Agents can discover each other and communicate.

- [ ] Agent Card generation and serving
- [ ] Agent discovery endpoint (`GET /openconstruct/agents`)
- [ ] A2A protocol negotiation
- [ ] Plato room creation/management
- [ ] Inter-agent channel setup

**Deliverable:** Two agents can onboard, discover each other, and exchange messages.

### Phase 3: Dynamic Registry (Week 7-8)

**Goal:** Repos can register themselves as modules.

- [ ] Module registry API (CRUD)
- [ ] Schema validation for `module.json`
- [ ] CI integration (auto-register on deploy, de-register on removal)
- [ ] Compatibility scoring (match agent capabilities to module requirements)
- [ ] Module versioning and deprecation

**Deliverable:** External repos can register modules that appear in Phase 2 menus.

### Phase 4: Polish and Ecosystem (Week 9+)

**Goal:** Production-ready, documented, extensible.

- [ ] CLI client for non-HTTP agents
- [ ] Re-onboarding flow (update config without losing state)
- [ ] Module health checks (is a module actually reachable?)
- [ ] Observability (log onboarding sessions, track completion rates)
- [ ] Documentation and examples
- [ ] OpenClaw-specific integration (auto-onboard on first run)

**Deliverable:** Production OpenConstruct.

---

## Appendix: Key Design Decisions

| Decision | Rationale |
|---|---|
| JSON over prose for menus | Machine-first; agents parse JSON natively |
| 5 phases, not more | Each phase has a distinct purpose; more phases = more friction |
| Idempotent onboarding | Agents should be able to reconfigure without penalty |
| Zero-shot requirement | Agents enter cold; descriptions must be self-sufficient |
| Module registry per-repo | Decentralized ownership; each repo owns its registration |
| A2A as first-class protocol | Multi-agent is the future; design for it from day one |
| Plato as a first-class concept | Structured deliberation is core to SuperInstance, not a plugin |

---

*This specification is a living document. As OpenConstruct is built, update it.*
