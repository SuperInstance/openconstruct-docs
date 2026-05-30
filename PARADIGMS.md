# OpenConstruct Paradigms

*The document that makes engineers go ah-ha. Not a sales pitch. A paradigm shift.*

---

## Paradigm 1: Text Is the Universal Interface

### The Insight

Agents think in text. Everything else is translation. This is not a limitation — it is the most important architectural decision in the system.

Text composes. You can concatenate two text outputs and get a valid input to a third system. You cannot concatenate two image tensors and get anything meaningful without a schema. Text diffs. You can look at two versions of an agent's output and immediately understand what changed. You cannot diff two embedding vectors. Text stores. Every database, every log system, every version control system in existence handles text natively. Text is the only universal data format that predates computing and will outlast every current framework.

In OpenConstruct, the tick payload is text. The agent's context window is text. The policy rules are text. The contracts are text. The Plato tile fields — question, answer, domain — are text. The workflow DAG node labels are text. This is not coincidence. It is the system's load-bearing assumption.

### The Technical Implementation

The tick schema enforces `payload: serde_json::Value`. JSON is text-encoded. Every sense module converts its native output — camera frames, sonar sweeps, haptic signals — into a text description before emitting a tick. The vision sense does not pass a tensor. It passes: `{"objects": [{"label": "chair", "confidence": 0.94, "bbox": [x,y,w,h]}], "timestamp_ns": 1234567890}`.

This means the correlator never handles tensors. It handles JSON. The policy engine never handles images. It handles descriptions. Agents never see raw sensor data. They see structured text derived from sensor data.

The translation happens once, at the sense boundary. After that, everything is text all the way down.

### Concrete Example

`flux-ast/` contains the FLUX instruction set. Every FLUX instruction is a text string with a defined grammar. A constraint violation is not a bitmask — it is a structured string: `CONSTRAINT_VIOLATED(sensor=temp_01, lo=20.0, hi=80.0, val=97.3)`. An agent receiving this tick can reason about it in natural language. A human reading the log can understand it without a decoder ring.

---

## Paradigm 2: The Hermit Crab Architecture

### The Insight

The shell is not the agent. The agent lives in the shell. Different shells for different environments. Same agent, different shells.

A hermit crab does not have its own shell. It finds one that fits, lives in it, and moves to a bigger one when it outgrows it. The crab is the persistent entity. The shell is infrastructure. You would not say "I am my house." Neither should an agent.

In conventional AI systems, the agent is entangled with its deployment environment. The agent that runs on a desktop is a different agent from the one that runs in a browser extension, which is different from the one on a mobile device. They share a name but not an implementation. This is the wrong inversion.

### The Technical Implementation

In OpenConstruct, the agent is a stateless function: `(context: Text) → (action: Text)`. The shell provides the context and interprets the action. Different shells:

- **Desktop shell**: provides file system access, terminal, browser control
- **ESP32 shell**: provides GPIO, mesh networking, sensor reads
- **Jetson shell**: provides CUDA inference, camera, sonar
- **Git shell**: provides repo state, commit history, diff, branch

The agent receives a text context from whatever shell it is running in. It produces a text action. The shell executes the action. The agent never knows which shell it is in — and this is correct. An agent that knows it is "in the browser" is an agent that has coupled itself to an accident of deployment.

### Concrete Example

`plato-tiles/` defines tiles that are shell-agnostic. A tile describing a software architecture concept is the same tile whether it is being processed by a desktop agent, a Jetson agent, or a cloud agent. The `ForgeRoom` in `plato-forge-bridge` is a shell abstraction: it presents assembled tiles to whatever agent is resident, regardless of where the room runs. The agent just sees tiles.

The hermit crab architecture is why the same Plato agent can run on a desktop, in a WASM sandbox, or on a Jetson — the agent implementation does not change. The shell changes.

---

## Paradigm 3: Simulation-First Coordination

### The Insight

Agents predict. Reality confirms or diverges. No event triggers needed. The system runs continuously and the delta is the information.

Conventional event-driven systems wait for something to happen, then react. This creates a fundamental asymmetry: the system knows the past (events that happened) but not the near future. It cannot pre-position resources, pre-compute decisions, or detect that an expected event did not happen.

Simulation-first inverts this. Every agent maintains a model of what it expects the world to look like at time T+1. At T+1, the actual tick arrives. The agent computes the delta between prediction and reality. If delta is zero, there is no information — nothing happened. If delta is non-zero, the delta is the event. The magnitude of the delta is the urgency.

### The Technical Implementation

The correlator does not wait for events. It runs continuously, emitting prediction ticks that describe expected future sensor states. When actual sensor ticks arrive, the correlator computes `actual - predicted` for each sense. Ticks with large deltas are flagged for the policy engine. Ticks with small deltas are discarded — they contain no new information.

The workflow DAG nodes are not triggers. They are matchers: "when the prediction delta for sensor X exceeds threshold Y, advance to node Z." This means the DAG never sleeps waiting for events. It continuously evaluates the prediction-reality gap.

### Concrete Example

In `fleet-simulation/`, the fleet simulation maintains a predicted state for every ESP32 room. The simulation emits `predicted_tick` messages one cycle ahead of real time. When actual ticks arrive from the mesh, the simulation computes divergence. A room that consistently has near-zero divergence is well-understood. A room with high divergence is anomalous. This is the sensor network's self-monitoring mechanism — and it costs nothing, because the simulation was already running.

---

## Paradigm 4: Conservation Ratio as Invariant

### The Insight

The Laplacian eigenvalue structure that underlies knowledge graphs, sensor networks, fleet topology, and agent collaboration is the same math. One math, many manifestations.

The Conservation Ratio (CR) is defined as the ratio of the first non-trivial Laplacian eigenvalue to the maximum eigenvalue of a graph. It measures how well-connected a graph is — how difficult it is to partition the graph into disconnected components. A CR near 1 means the graph is robust. A CR near 0 means the graph is fragile.

This appears everywhere in OpenConstruct:

- **Knowledge graph**: tiles connected by shared tags and domains. Low CR = fragmented knowledge, poor cross-domain inference.
- **Sensor network**: ESP32 rooms connected by mesh links. Low CR = partitioned sensor coverage, dead zones.
- **Fleet topology**: Jetson nodes connected to cloud. Low CR = single point of failure.
- **Agent collaboration**: agents connected by shared tick subscriptions. Low CR = isolated agents, no emergent coordination.

### The Technical Implementation

`spectral-conservation/` computes CR over any graph represented as an adjacency structure. The Laplacian L = D - A where D is the degree matrix and A is the adjacency matrix. The eigenvalue decomposition of L gives λ₀=0 (always, for connected graphs), λ₁ (algebraic connectivity, also called Fiedler value), and λ_max. CR = λ₁ / λ_max.

The system monitors CR for each of these graph domains continuously. When the sensor network CR drops below 0.2, a mesh partition is forming. When the knowledge graph CR drops, the tile collection has become domain-siloed. When the agent collaboration CR drops, agents are operating independently rather than collectively.

### Concrete Example

In `constraint-theory-core-cuda/`, the CUDA kernel for constraint checking operates over a graph where nodes are sensors and edges are constraint relationships. The kernel computes the Laplacian spectrum to identify which sensors are load-bearing (high degree) versus peripheral (low degree). Sensors with low Fiedler centrality can be removed from the active constraint set without significantly degrading coverage — this is the algebraic justification for sensor pruning.

One math. Four domains. Same algorithm.

---

## Paradigm 5: Modular Modularity

### The Insight

Every module works alone. Compose them and you get emergent behavior. The correlator is useless without senses. The fleet is useless without transport. Together they are a sensor network.

This sounds obvious. It is not. Most systems are designed for composition first, standalone operation second. They have internal dependencies that make isolated testing impossible. A module that requires 6 other modules to produce a testable output is not a module — it is a subsystem with a single interface. A real module does something useful in isolation.

### The Technical Implementation

Each sense module in OpenConstruct has a standalone test binary that generates synthetic sensor data, runs the processing pipeline, and emits ticks to stdout. No mesh networking required. No Jetson required. No correlator required. This is not just for testing — it is the architectural proof that the module is actually modular.

The correlator has a standalone mode where it reads ticks from a file (or stdin) rather than a live bus. You can run the full cross-sense fusion pipeline on recorded tick logs with no hardware. This is how regression testing works: record a session, replay it, compare correlator output.

The policy engine can load a policy file and evaluate it against a tick JSON on the command line. No agents, no sessions, no workflow DAG. This means policy development does not require a running system.

### Concrete Example

`guard-dsl/` implements the policy language as a standalone parser and evaluator. The guard DSL can be used without the policy engine module. The policy engine uses the guard DSL, but the guard DSL does not know about the policy engine. This is not accidental — the dependency arrow goes one way, and the module at the tip of the arrow has no knowledge of what is at the base.

Compose `guard-dsl` + `policy engine` + `sandbox` and you get a sandboxed policy-enforced execution environment. Each piece was already working before the composition.

---

## Paradigm 6: The Cave Is the Architecture

### The Insight

The Plato allegory of the cave is not decorative. Shadows are the interface. The wall is the translation layer. Agents never touch reality directly. Humans see through the same wall from the other side.

In the allegory, prisoners see shadows cast on a cave wall. The shadows are all they ever see. The objects casting the shadows are real, but inaccessible. The prisoners build their model of reality from shadows.

OpenConstruct agents are the prisoners. Sensor ticks are the shadows. The physical world — the camera's photons, the sonar's pressure waves, the user's keystrokes — casts shadows onto the tick bus. Agents build their models from ticks. They never touch the physical world directly.

This is not a limitation. This is the correct architecture for any system that must be safe, testable, and auditable. An agent that can directly actuate hardware without going through the shadow layer (the tick/policy/sandbox stack) is an agent that cannot be audited, cannot be replayed, and cannot be tested safely.

The human operator is also looking at shadows. The fleet dashboard shows tick-derived summaries. The session log shows tick histories. The human sees the same representation as the agent — and this means the human and agent have the same epistemic position, which is the prerequisite for meaningful human oversight.

### The Technical Implementation

`fleet-gateway/` is the wall. It is the boundary between the physical sensor world and the tick world. Nothing crosses the wall without being transformed into a tick. No tick crosses the wall outward without being transformed into a physical action. The gateway is the only place in the system where physical causality can be influenced.

The gateway logs every crossing. Every inbound sensor reading. Every outbound actuation command. This log is the ground truth of what the system did and why.

### Concrete Example

`plato-soul-fingerprint/` stores agent identity not as a key or credential, but as a behavioral fingerprint derived from tick patterns. The fingerprint is a shadow — it is derived from observable behavior, not from claimed identity. Two agents with the same fingerprint behave the same way. An agent whose fingerprint diverges from its baseline is an agent that has changed behavior. The system detects this from the shadow, without ever inspecting the agent's internals.

---

## Paradigm 7: Git-Agent Native

### The Insight

Repos are shells. Commits are actions. Branches are timelines. An agent that understands git understands the fleet.

Git is the most successful distributed coordination system ever built. It handles merging, divergence, conflict detection, and history. It does this without a central coordinator. It does this across heterogeneous nodes (developers). It does this across time (branches are time-lines, not just feature-lines).

These are exactly the properties needed for a multi-agent fleet. Agents should not have a separate coordination protocol when git already solves coordination. The git model — distributed, content-addressed, append-only, mergeable — is the right model for agent coordination.

### The Technical Implementation

In OpenConstruct, each agent has a git repo that is its shell. The agent's current state is the HEAD commit. The agent's history is the commit log. The agent's action is a commit — it changes files, commits, and the commit is the action record. Agent coordination is a merge: two agents that have worked on related problems merge their branches, and git's merge algorithm handles the reconciliation.

The workflow DAG maps naturally to a git DAG. A DAG node is a commit. A DAG edge is a parent relationship. Branching in the workflow is branching in git. Merging workflow branches is a git merge.

### Concrete Example

`forgemaster-shell/` implements the git-shell for Plato agents. An agent running in the forgemaster shell reads its context from the git-tracked file state and writes its actions as commits. The entire session history is a git log. Replaying a session is a git rebase. Rolling back a bad agent action is a git revert. There is no custom history mechanism because git is the history mechanism.

The fleet gateway's actuation log (from Paradigm 6) is a git repo. Every physical action is a signed commit. The physical world's ground truth is version-controlled.

---

## Paradigm 8: Formal Verification as a Layer

### The Insight

Mercury proves what Rust runs. Total functions for total systems. Formal verification is not a gate — it is a layer that sits under everything else.

Formal verification is often treated as an alternative to testing, or as a gold-plating exercise for safety-critical systems. Both framings miss the point. Formal verification proves properties that testing cannot: exhaustive coverage of all inputs, not just the inputs you thought to test. A function that is proven correct is correct. A function that passes 10,000 tests might fail on the 10,001st.

But formal verification has costs: it requires total functions (no partial functions, no exceptions, no runtime errors), it scales poorly with code size, and it requires engineers who can write specifications. These constraints make it impractical as the primary development methodology.

The correct use is as a layer: write the core invariants in Mercury or equivalent, prove them correct, then run Rust (or C, or whatever) implementations against the Mercury specifications as the oracle. The Mercury code is not the production code — it is the specification that the production code must match.

### The Technical Implementation

`flux-verify-api/` exposes the Mercury-verified FLUX invariants as an API. The production Rust implementation in `constraint-theory-core-cuda/` validates against this API during testing. The workflow is:

1. Write the constraint invariant in Mercury (total function, proven correct).
2. Implement the constraint check in Rust (fast, handles real data).
3. In tests, generate random inputs, run both, compare outputs.
4. Any divergence is a bug in the Rust implementation, not the Mercury spec.

Mercury proofs cover: constraint monotonicity (a constraint that is satisfied at time T is still satisfied at T+ε under defined conditions), conservation properties (the CR of the sensor graph cannot decrease by more than a defined epsilon per tick without a topology change), and tick ordering (ticks within a session are totally ordered).

### Concrete Example

`guard-constraints/` contains Mercury specifications for the guard DSL's evaluation semantics. A guard expression has a Mercury-proven evaluation function. The Rust guard evaluator is tested by generating random guard expressions, evaluating them in Mercury (via FFI), and comparing to the Rust result. The Mercury code is 40 lines. It proves that the guard evaluation is deterministic, terminating, and has no implicit undefined behavior for any syntactically valid input. The Rust code is 400 lines. The 40 Mercury lines are doing more safety work than the 400 Rust lines.

This is the correct ratio. Specifications are load-bearing. Implementations are replaceable.

---

## Summary

| Paradigm | Core claim |
|---|---|
| Text is the universal interface | Agents think in text; everything else is translation done once at the boundary |
| Hermit crab architecture | The agent is not its shell; same agent, different shells |
| Simulation-first coordination | Agents predict; the delta between prediction and reality is the information |
| Conservation Ratio as invariant | One algebraic structure underlies knowledge, sensors, fleet, and collaboration |
| Modular modularity | Every module works alone; emergence comes from composition |
| The cave is the architecture | Agents and humans both see shadows; the wall is the only safe place for causality |
| Git-agent native | Repos are shells, commits are actions, branches are timelines |
| Formal verification as a layer | Mercury proves what Rust runs; specifications are load-bearing |

These eight paradigms are not independent. They reinforce each other. Text is universal (1) because the wall (6) translates everything to text before it enters the system. The hermit crab (2) works because the shell is text in and text out. Simulation-first (3) works because predictions and actuals are both text and can be diffed. The Conservation Ratio (4) is computed over text-encoded graph structures. Modular modularity (5) works because text interfaces are composable. Git (7) works because git stores text. Formal verification (8) verifies text-encoded specifications.

Pull any one paradigm out and the others weaken. Together, they describe a coherent system.
