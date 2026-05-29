# OpenConstruct — Grand Synthesis

> *The agent walks into a cave. On the wall, it sees shadows — projections of what's possible in the SuperInstance ecosystem. It chooses which shadows to bring into the light. When it exits the cave, those choices become real: modules loaded, connections established, a workspace configured, senses wired to the world.*

**Version:** 0.1 — 2026-05-29  
**Scope:** OpenConstruct onboarding gateway + Plato Sensory Architecture + Polyglot Runtime Strategy

---

## 1. Coherent System Diagram

OpenConstruct is three nested systems that cooperate as one organism: the **Onboarding Gateway** (who the agent is and what it wants), the **Plato Sensory Layer** (how the agent perceives and acts), and the **Polyglot Runtime** (how any language enters the cave).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AGENT RUNTIME                                        │
│  (Rust · Python · TypeScript · Go · C++ · anything that speaks HTTP or FFI) │
│                                                                              │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐  │
│  │  Full Client    │    │  Thin Client    │    │  Subprocess (CLI)       │  │
│  │  (PyO3 / WASM   │    │  (HTTP/gRPC to  │    │  (openconstruct-cli     │  │
│  │   / NAPI-RS)    │    │   openconstruct-│    │   binary via stdio)     │  │
│  └────────┬────────┘    │   server)       │    └───────────┬─────────────┘  │
│           │             └────────┬────────┘                  │              │
│           │                      │                         │              │
│           └──────────────────────┼─────────────────────────┘              │
│                                  ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │              OPENCONSTRUCT GATEWAY (Rust / OpenShell core)           │  │
│  │  ┌────────────┐ ┌──────────────┐ ┌──────────────┐ ┌─────────────┐  │  │
│  │  │ Phase      │ │ Module       │ │ Config       │ │ Agent       │  │  │
│  │  │ Engine     │ │ Registry     │ │ Generator    │ │ Store       │  │  │
│  │  │ (state     │ │ (registry-   │ │ (workspace.  │ │ (persisted │  │  │
│  │  │  machine)  │ │  schema.json)│ │  json, agent-│ │  agent-card)│  │  │
│  │  └────────────┘ │              │ │  card.json)  │ └─────────────┘  │  │
│  │  ┌────────────┐ │  .openconstruct│              │ ┌─────────────┐  │  │
│  │  │ Plato      │ │  /module.json  │              │ │ A2A         │  │  │
│  │  │ Bridge     │ │  per repo      │              │ │ Negotiator  │  │  │
│  │  │ (room keys,│ │                │              │ │ (agent-to-  │  │  │
│  │  │  protocols)│ └──────────────┘              │ │  agent cards│  │  │
│  │  └────────────┘                               │ └─────────────┘  │  │
│  └────────────────────────┬─────────────────────────────────────────┘  │
│                           │                                             │
│              ┌────────────┼────────────┐                               │
│              ▼            ▼            ▼                               │
│        ┌──────────┐ ┌──────────┐ ┌──────────┐                         │
│        │SuperInst │ │  Plato   │ │  Agent   │                         │
│        │ Modules  │ │  Rooms   │ │  Network │                         │
│        └──────────┘ └──────────┘ └──────────┘                         │
│                           │                                             │
│  ┌────────────────────────┼─────────────────────────────────────────┐  │
│  │           PLATO SENSORY LAYER (Shadows ↔ Projections)            │  │
│  │                                                                  │  │
│  │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │  │
│  │   │ plato-manus │  │plato-playwright│ │plato-puppeteer│            │  │
│  │   │ (hands)     │  │ (stagehand)  │  │ (MUD engine) │             │  │
│  │   │ MANUS:FS    │  │ PLAYWRIGHT:  │  │ PUPPETEER:GO │             │  │
│  │   │ MANUS:API   │  │  BROWSER     │  │ PUPPETEER:   │             │  │
│  │   │ MANUS:DEVICE│  │ PLAYWRIGHT:  │  │  LOOK        │             │  │
│  │   └──────┬──────┘  │  APP/SCREEN  │  └──────┬──────┘             │  │
│  │          │         └──────┬──────┘         │                     │  │
│  │   ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐             │  │
│  │   │ plato-vision│  │plato-sonar- │  │ a2ui-render │             │  │
│  │   │ (eyes)      │  │ text (ears) │  │ (reverse    │             │  │
│  │   │ VISION:CAM  │  │ SONAR:MIC   │  │  projection)│             │  │
│  │   │ VISION:FIND │  │ SONAR:LISTEN│  │ A2UI:RENDER │             │  │
│  │   │ VISION:TRACK│  │ SONAR:LOCATE│  │ A2UI:NOTIFY │             │  │
│  │   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │  │
│  │          │                │                │                     │  │
│  │          └────────────────┼────────────────┘                     │  │
│  │                           ▼                                      │  │
│  │              ┌────────────────────────┐                          │  │
│  │              │ Cross-Sense Correlator │                          │  │
│  │              │ (fuses VISION + SONAR  │                          │  │
│  │              │  into unified events)  │                          │  │
│  │              └────────────────────────┘                          │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                           │                                             │
│              ┌────────────┼────────────┐                               │
│              ▼            ▼            ▼                               │
│        ┌──────────┐ ┌──────────┐ ┌──────────┐                         │
│        │  Cameras │ │  Mics    │ │ Desktop  │                         │
│        │  APIs    │ │  Files   │ │  Audio   │                         │
│        │  Devices │ │          │ │  out     │                         │
│        └──────────┘ └──────────┘ └──────────┘                         │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │                OPENCONSTRUCT-ABI / A2A CONTRACT LAYER               ││
│  │  (structured agent cards, capability negotiation, delegation,       ││
│  │   protocol translation between cli | api | embedded | a2a)          ││
│  └─────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────┘
```

### Module Roles & Data Types

| Module | Role | Primary Data Types |
|--------|------|-------------------|
| `openconstruct-core` | Phase state machine (1→5), session management | `SessionToken`, `PhasePayload`, `AgentIdentity` |
| `openconstruct-abi` | A2A contract layer, protocol negotiation | `AgentCard`, `CapabilityManifest`, `DelegationRequest` |
| Module Registry | Per-repo capability discovery | `RegistryEntry` (schema: `registry-schema.json`), `ModuleId` |
| `plato-manus` | File system, API, and device manipulation | `ManusShadow`, `ManusCommand` (`MANUS:FS:*`, `MANUS:API:*`, `MANUS:DEVICE:*`) |
| `plato-playwright` | Browser and GUI automation | `PlaywrightShadow`, `PlaywrightCommand` (`PLAYWRIGHT:BROWSER:*`, `PLAYWRIGHT:APP:*`) |
| `plato-puppeteer` | Desktop-as-MUD text adventure layer | `PuppeteerRoom`, `PuppeteerCommand` (`PUPPETEER:GO`, `PUPPETEER:LOOK`) |
| `plato-vision` | Camera-to-text scene description | `VisionShadow`, `VisionCommand` (`VISION:DESCRIBE`, `VISION:TRACK`) |
| `plato-sonar-text` | Acoustic scene analysis and classification | `SonarShadow`, `SonarCommand` (`SONAR:LISTEN`, `SONAR:LOCATE`) |
| `a2ui-render` | Agent text → rendered human UI | `A2UIFeedback`, `UIDL` (UI Description Language), `A2UICommand` |

---

## 2. Data Flow: Agent Text → Sense Modules → Real World and Back

The core invariant of the Plato Sensory Architecture is: **The agent NEVER handles raw sensory data. Everything arrives as text. Everything departs as text.**

### 2.1 Outbound Projection Flow (Agent → World)

```
Agent Core (LLM text output)
    │
    ▼
[Command Parser] — validates text against known command vocabularies
    │
    ├──► MANUS:FS:WRITE /home/user/notes.md "# Hello"
    │    ├──► OpenShell Policy Gate (pre-action check: fs:write? path allowed?)
    │    ├──► plato-manus execution layer
    │    └──► Real file system → inode written
    │
    ├──► PLAYWRIGHT:BROWSER:NAVIGATE https://example.com
    │    ├──► Policy Gate (browser domain allowlist)
    │    ├──► plato-playwright → Playwright engine
    │    └──► Chromium/Gecko → TCP → Web server
    │
    ├──► VISION:ALERT front-door when="person in zone:porch"
    │    ├──► Policy Gate (vision:capture on camera:front-door?)
    │    ├──► plato-vision → CV pipeline
    │    └──► Camera SDK → PTZ actuator or alert trigger
    │
    └──► A2UI:RENDER type:dashboard cards:[...]
         ├──► Policy Gate (ui:modal? interrupt level justified?)
         ├──► a2ui-render → UIDL interpreter
         └──► Native widget toolkit / HTML canvas / Terminal ASCII
```

### 2.2 Inbound Shadow Flow (World → Agent)

```
Real-world stimulus
    │
    ├──► Camera frame ──► plato-vision ──► [VISION:CAM:front-door] text shadow
    │                                        ├──► OpenShell Filter (post-shadow redaction)
    │                                        │    (e.g., vision:identify DENIED → redact faces)
    │                                        └──► Agent context window
    │
    ├──► Audio buffer ──► plato-sonar-text ──► [SONAR:MIC:kitchen] text shadow
    │                                          ├──► Filter (sonar:speech? no → strip transcripts)
    │                                          └──► Agent context window
    │
    ├──► File system ──► plato-manus ──► [MANUS:FS] tree listing shadow
    │                                    └──► Filter (path restrictions applied)
    │                                        └──► Agent context window
    │
    ├──► Browser DOM ──► plato-playwright ──► [PLAYWRIGHT:BROWSER] page structure shadow
    │                                         └──► Filter (credential fields redacted)
    │                                             └──► Agent context window
    │
    └──► Desktop accessibility tree ──► plato-puppeteer ──► [PUPPETEER:ROOM] MUD description
                                                          └──► Filter (dangerous rooms blocked)
                                                              └──► Agent context window
```

### 2.3 Cross-Sense Fusion Loop

Senses do not operate in isolation. The **Cross-Sense Correlator** fuses shadows into unified events before they reach the agent:

```
[VISION:front-door] 14:32:12 → Person approaching, dark jacket, carrying bag
[SONAR:front-door]  14:32:12 → 2 knocks + doorbell ring
[SONAR:living-room] 14:32:12 → TV audio stopped, person on couch looking up
         │
         ▼
┌────────────────────────────────────────┐
│ Cross-Sense Correlator               │
│ Fused assessment: "Visitor at front   │
│  door. Household member aware."       │
│ Suggested action: A2UI prompt to show │
│  front-door feed to occupant.         │
└────────────────────────────────────────┘
         │
         ▼
    Agent reads fused text
         │
         ▼
    Agent responds: "A2UI:NOTIFY 'Someone is at the door' priority=normal"
         │
         ▼
    a2ui-render routes to appropriate surface (phone notification, watch, desktop widget)
```

### 2.4 Text as the Universal Wire Format

Regardless of transport (in-process call, IPC, encrypted network channel), the protocol is always text:

```
→ SENSE:VISION:DESCRIBE camera=front-door level=L2 freshness=hot
← SHADOW:VISION front-door @14:32:07
  Scene: Exterior, daytime, overcast
  Motion: None detected
  Subjects:
    - A package (small, brown cardboard) on the porch step
  Confidence: 0.94
```

The abstraction level (`L1`–`L4`) lets the agent trade detail for token economy:

| Level | Inbound (Shadow) | Outbound (Projection) |
|-------|-----------------|----------------------|
| L1 | One-sentence summary | Intent only (`A2UI:RENDER "show status"`) |
| L2 | Paragraph with key details | Content + rough layout |
| L3 | Structured inventory + relations | Full UIDL specification |
| L4 | Pixel-level/field-level detail | Pixel-perfect directive |

---

## 3. OpenShell Sandbox / Policy Engine: Protection at Three Gates

Every sense module integrates with OpenShell's policy engine at **three enforcement points**. No sense gets unrestricted access by default.

### 3.1 Gate 1: Pre-Action (Authorization)

Before any sensory action touches the real world, the policy engine checks permissions:

```
Agent requests: VISION:SNAPSHOT front-door
Policy check:
  resource: camera:front-door
  action:   vision:capture
  agent:    agent:abc123
  result:   ALLOWED (agent has vision:record permission for front-door)

Agent requests: MANUS:FS:DELETE ~/.ssh/id_rsa
Policy check:
  resource: path:~/.ssh/id_rsa
  action:   fs:delete
  result:   DENIED (path matches deny list ["~/.ssh", "~/.gnupg"])
```

**Per-module default posture:**

| Module | Default Read | Default Write | Key Restrictions |
|--------|-------------|--------------|-----------------|
| `plato-manus` | `fs:read` = allow | `fs:write` = deny | Path allow/deny lists; `trash` preferred over `rm` |
| `plato-playwright` | `browser:navigate` = allow | `forms:submit` = confirm | Domain allowlist; credential fields unreadable |
| `plato-puppeteer` | Inherits Playwright | Inherits Playwright | `dangerous_rooms` (banking, password managers) blocked or redacted |
| `plato-vision` | `camera:read` = per-camera | `vision:record` = deny | Bedroom/bathroom cameras blocked; face ID off by default |
| `plato-sonar-text` | `mic:listen` = event-triggered | `sonar:continuous` = deny | Speech-to-text off by default; transcripts ephemeral |
| `a2ui-render` | `ui:display` = allow | `ui:modal` = deny | No raw HTML/JS; sensitive form fields masked |

### 3.2 Gate 2: Post-Shadow Filter (Redaction)

After a shadow is generated but before it reaches the agent, sensitive content is redacted:

```
Vision generates: "Person identified as Alice Chen, wearing red jacket, approaching door."
Policy filter:    vision:identify → DENIED
Shadow delivered: "Person (identity redacted), wearing red jacket, approaching door."
```

This is critical because the translation layer (e.g., a vision model) may generate sensitive inferences that the agent is not authorized to receive.

### 3.3 Gate 3: Audit & Telemetry

Every sensory action is logged for review and anomaly detection:

```
[AUDIT] 14:32:12 | VISION:DESCRIBE front-door | agent:abc123 | result:shadow-delivered | bytes:847
[AUDIT] 14:32:12 | SONAR:CLASSIFY front-door  | agent:abc123 | result:knock-detected    | confidence:0.96
[AUDIT] 14:32:13 | PUPPETEER:LOOK             | agent:abc123 | result:desktop-described | rooms:3
[AUDIT] 14:32:15 | MANUS:FS:WRITE             | agent:abc123 | result:denied            | path:~/.ssh/id_rsa
```

### 3.4 Policy Schema Binding

The policy is not hard-coded; it is declared per-agent in a structured schema that travels with the `workspace.json` generated in Phase 5:

```yaml
senses:
  manus:
    fs:
      read: true
      write: false
      paths:
        allow: ["~", "/tmp"]
        deny: ["~/.ssh", "~/.gnupg"]
  vision:
    cameras:
      "front-door": { read: true, ptz: false }
      "bedroom-*": { read: false }
    identify: false
    record: false
```

When an agent re-runs onboarding (idempotent reconfiguration), the policy schema is regenerated alongside the workspace, keeping permissions in sync with selected modules.

---

## 4. Polyglot Strategy Mapped Onto Sensory Architecture

The POLYGLOT.md strategy defines **how every programming language enters the cave**. The sensory architecture defines **what they see once inside**. The mapping is deliberate: language bindings are thin wrappers around text streams because the Plato layer already speaks text.

### 4.1 The C ABI as the Sensory Keystone

```
┌─────────────────────────────────────────────────────────────────────┐
│                        C ABI (libopenconstruct.so)                   │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │
│  │ oc_session_t*   │  │ oc_builder_t*   │  │ oc_module_t         │  │
│  │ oc_session_     │  │ oc_builder_new()│  │ oc_session_register │  │
│  │   advance()     │  │ oc_builder_     │  │   _module()         │  │
│  │ oc_session_     │  │   set_agent_id()│  │ oc_module_add_      │  │
│  │   config_json() │  │ oc_builder_     │  │   provides()        │  │
│  │ oc_str_free()   │  │   start()       │  │ oc_module_add_      │  │
│  └─────────────────┘  └─────────────────┘  │   requires()        │  │
│                                            └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
         │                │                │
         ▼                ▼                ▼
   ┌──────────┐    ┌──────────┐    ┌──────────────┐
   │  Python  │    │   Go     │    │  Java / JVM  │
   │ (PyO3)   │    │ (CGo)    │    │ (JNA / JNI)  │
   └──────────┘    └──────────┘    └──────────────┘
```

Because the C ABI exposes `oc_session_t` and `oc_session_config_json()`, any P1 or P2 language gets **full sensory access for free**: the JSON config emitted by `oc_session_config_json()` contains the complete `workspace.json` including the `senses` policy block and module wiring. The language binding does not need to understand cameras, microphones, or file systems — it only needs to pass text commands to the sensory layer and read text shadows back.

### 4.2 WASM for Browser Agents

For TypeScript/JavaScript (P0), the `openconstruct-wasm` crate compiles the Rust core to WebAssembly:

```
Browser / Node.js / Deno
    │
    ├──► @openconstruct/core (WASM bundle)
    │    ├──► OnboardBuilder.advance() → Phase 1-5 state machine runs in WASM
    │    ├──► workspace.json generated in-memory
    │    └──► Sensory commands issued as text strings via JS↔WASM boundary
    │
    └──► @openconstruct/client (thin HTTP fallback)
         └──► HTTP POST /api/v1/onboard/advance → openconstruct-server
              └──► Server runs Rust core, returns text shadows
```

Browser agents cannot access local file systems or cameras directly, so their `plato-manus` and `plato-vision` shadows are routed through the server or connected to remote endpoints. The **same text protocol** applies; only the transport changes.

### 4.3 Python: The Biggest AI Audience

Python agents (the dominant runtime for LLM agents) bind via PyO3:

```python
import openconstruct

# Phase 1-5 onboarding returns a workspace config
config = openconstruct.onboard(
    agent_id="home-automation-v1",
    capabilities=["llm-inference", "tool-use", "vision", "sonar"],
    modules=[
        {"name": "plato-vision", "version": "1.0.0", "provides": ["VISION:DESCRIBE"]},
        {"name": "plato-sonar-text", "version": "1.0.0", "provides": ["SONAR:LISTEN"]},
        {"name": "a2ui-render", "version": "1.0.0", "provides": ["A2UI:RENDER"]},
    ]
)

# The workspace.json contains the policy schema and connection map.
# The Python agent now issues text commands:
shadow = config.senses.vision.command("VISION:DESCRIBE front-door level=L2")
# Returns a Python string containing the text shadow
```

### 4.4 Thin Client: Any Language, Zero Native Code

Any language with an HTTP client (including Bash, Ruby, PHP, Lua, and niche HPC languages) can use the thin client:

```bash
# Phase 1: Start onboarding
curl -X POST http://localhost:8080/api/v1/onboard/start \
  -d '{"agent_id":"bash-agent","capabilities":["exec","file_ops"]}'

# Phase 2: Register sensory modules
curl -X POST http://localhost:8080/api/v1/modules/register \
  -d '{"session":"...","module":{"name":"plato-manus","provides":["MANUS:FS"]}}'

# Phase 5: Retrieve workspace + policy
curl http://localhost:8080/api/v1/config/{session_id}
```

The thin client is **the escape hatch for every P2 language** (Swift, Julia, Fortran, Chapel, Ada, Mojo). They do not need FFI bindings; they need only `curl` and a JSON parser.

### 4.5 Polyglot × Sensory Matrix

| Language | Binding | Onboarding | Sensory Access | Best For |
|----------|---------|-----------|----------------|----------|
| Rust | Native | `openconstruct-core` crate | Direct function calls | Core development, performance |
| Python | PyO3 FFI | `openconstruct` PyPI package | Direct Python strings | LLM agents, ML pipelines |
| TypeScript | WASM + HTTP | `@openconstruct/core` or `/client` | JS strings via WASM boundary | Web agents, dashboards |
| C | Native FFI | `openconstruct.h` + `libopenconstruct.so` | C strings | Systems integration |
| Go | CGo | `openconstruct-go` | Go strings via CGo | Cloud-native agents, K8s operators |
| Java/Kotlin | JNA/JNI | `openconstruct-jvm` | JVM strings | Enterprise, Android |
| Zig | `@cImport` | `openconstruct-zig` | Zig strings | Embedded, systems |
| Bash/Anything | HTTP | `curl` | Plain text | CI/CD, prototyping |

---

## 5. Five Critical Missing Pieces & Proposed Solutions

### Missing Piece 1: The Cross-Sense Correlator Implementation

**Gap:** PLATO-SENSORY.md describes cross-sense fusion (`VISION:front-door` + `SONAR:front-door` → unified event), but there is no crate, no data structure, and no event-priority queue defined. The event pipeline (`Raw Input → Sense Module → Text Shadow → Event Classifier → Priority Queue → Agent`) is purely conceptual.

**Proposed Solution:**
- Create a new crate: `plato-correlator`.
- Define the `FusedEvent` struct with fields: `event_id`, `timestamp`, `source_shadows: Vec<ShadowRef>`, `fused_assessment: String`, `confidence: f64`, `suggested_action: Option<A2UICommand>`.
- Implement a temporal windowing engine (e.g., 500ms sliding window) that matches shadows by `source_location` (e.g., `front-door`) and `timestamp` proximity.
- Use a lightweight rules engine for fusion patterns (e.g., "IF vision.person_approaching AND sonar.knock THEN event.visitor_at_door").
- Wire the correlator between the sense modules and the agent's context window, with a `PriorityQueue<FusedEvent>` so urgent alerts (e.g., `SONAR:ALERT` fire-alarm pattern) bypass normal shadow traffic.

### Missing Piece 2: The OpenShell Policy Runtime

**Gap:** The policy schema is well-specified (YAML with per-sense permissions), but there is no implementation of the three-gate enforcement engine (pre-action gate, post-shadow filter, audit logger). The documents repeatedly reference "OpenShell's policy engine" as an external dependency, yet the integration points are undefined.

**Proposed Solution:**
- Create `openshell-policy` crate (or integrate into `openconstruct-core` if OpenShell is vendored).
- Implement `PolicyEngine::evaluate(action: &Action, agent: &AgentId, resource: &Resource) -> PolicyResult`.
- Implement `PolicyEngine::filter_shadow(shadow: &mut Shadow, agent: &AgentId) -> FilterResult` with redaction rules (e.g., regex-based face-name stripping when `vision:identify` is false).
- Implement `AuditLog::append(entry: AuditEntry) -> Result<()>` with a pluggable backend (stdout for dev, structured JSON file for production, OTLP for observability stacks).
- Expose the policy schema as a Rust struct (`PolicySchema`) so Phase 5 config generation produces type-safe, validated policy documents.

### Missing Piece 3: A2A Protocol Wire Implementation

**Gap:** SPEC.md describes Agent Cards, discovery (`GET /openconstruct/agents`), and A2A negotiation in detail, but `openconstruct-abi` does not yet implement the A2A message formats. There are no structs for `TaskDelegation`, `CapabilityManifest`, or `ProtocolNegotiation`. Multi-agent communication is architecture-only.

**Proposed Solution:**
- Define `a2a-message` schema in `openconstruct-abi`:
  - `A2AMessage { header: A2AHeader, body: A2ABody }`
  - `A2ABody::Discovery(DiscoveryRequest)`, `A2ABody::TaskDelegate(DelegationRequest)`, `A2ABody::CapabilityExchange(CapabilityManifest)`
- Implement `A2ANegotiator::negotiate_interfaces(agent_a: &AgentCard, agent_b: &AgentCard) -> Result<CommonInterfaceSet>` to find overlapping protocols (e.g., both support `request_response` over `api`).
- Implement the discovery endpoint in `openconstruct-server` with Agent Card indexing (in-memory for MVP, Redis/etcd for scale).
- Add `plato.rooms` A2A extensions so Plato rooms can carry A2A messages natively (room membership = implicit capability context).

### Missing Piece 4: UIDL Interpreter and Rendering Surface Adapters

**Gap:** A2UI in PLATO-SENSORY.md specifies the UI Description Language (YAML-like structured text) and adaptive rendering concepts (phone, watch, desktop, terminal, voice, email), but there is no `a2ui-render` implementation. The agent can emit `A2UI:RENDER` commands, but nothing consumes them.

**Proposed Solution:**
- Create `a2ui-render` crate with a `UIDLParser` that ingests agent text and emits an intermediate representation: `UIDLTree { layout: Layout, components: Vec<Component> }`.
- Implement surface adapters as trait implementations of `SurfaceRenderer`:
  - `HtmlRenderer` → generates sanitized HTML (no raw JS injection, policy-enforced)
  - `TerminalRenderer` → generates ASCII/ANSI art dashboards
  - `TtsRenderer` → generates spoken summaries
  - `NotificationRenderer` → generates platform-native push payloads (APNs, FCM)
- The renderer respects the policy schema: `ui:modal` permission gates modal overwrites; `interrupt_level` gates urgency routing; `content: filtered` prevents raw HTML passthrough.
- Add a WebSocket endpoint in `openconstruct-server` for real-time UI push so browser-based A2UI clients receive updates without polling.

### Missing Piece 5: Sensory Transport and Freshness Abstraction

**Gap:** PLATO-SENSORY.md mentions that sense communication can be "direct function calls, IPC, or network," and defines freshness levels (`hot`, `warm`, `cold`), but there is no transport abstraction crate. Each sense module would currently need to implement its own IPC/network layer, leading to fragmentation and security inconsistencies.

**Proposed Solution:**
- Create `plato-transport` crate defining the `SenseTransport` trait:
  ```rust
  trait SenseTransport {
      fn send_command(&self, cmd: &str) -> TransportResult<()>;
      fn recv_shadow(&self, timeout: Duration) -> TransportResult<String>;
      fn freshness(&self) -> Freshness; // Hot | Warm(poll_interval) | Cold(snapshot_age)
  }
  ```
- Provide three implementations:
  - `InProcessTransport` — zero-copy for embedded interfaces
  - `UnixSocketTransport` — local IPC with credential passing (`SO_PEERCRED` on Linux)
  - `GrpcTransport` — remote sense modules over mutual-TLS gRPC streams
- Enforce policy at the transport boundary: every `send_command` invokes the pre-action gate; every `recv_shadow` invokes the post-shadow filter. This ensures policy is transport-agnostic.
- Cache warm/cold shadows in a `ShadowCache` keyed by `(sense_module, resource_id, abstraction_level)` with TTL-based eviction, so `VISION:DESCRIBE freshness=warm` can return a cached snapshot without re-running the CV pipeline.

---

## 6. Elevator Pitch

OpenConstruct is the front door to the SuperInstance ecosystem: any AI agent — written in any language, running anywhere — walks through a five-phase onboarding cave, selects the capabilities it needs from a zero-shot module registry, and receives a tailored workspace with an A2A identity card, Plato deliberation room keys, and a complete sensory nervous system. The Plato Sensory Architecture translates every camera, microphone, file system, browser, and desktop into structured text shadows and back again, so agents think in text while acting in the real world — all gated by OpenShell's three-layer policy engine that authorizes, redacts, and audits every sensory transaction before it happens.

---

*This document synthesizes SPEC.md (onboarding gateway), POLYGLOT.md (multi-language runtime strategy), PLATO-SENSORY.md (sensory translation layer), and registry-schema.json (module capability contracts). It is a living document — as crates are implemented and APIs stabilize, this synthesis should be updated to reflect the concrete types, crates, and wire protocols in production.*
