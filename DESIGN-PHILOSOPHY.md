# Design Philosophy

> *Why OpenConstruct exists, what it believes, and how those beliefs shape every module.*

OpenConstruct is a fork of NVIDIA's OpenShell (Apache 2.0) that adds agent onboarding, sensory translation, fleet discovery, and inter-agent message passing. Every design decision flows from seven principles. This document explains each one with concrete examples from the actual codebase.

---

## 1. Text Is the Universal Interface

Agents think in text. They receive prompts, produce completions, reason over strings. The rest of the world — cameras, microphones, touchscreens, APIs — speaks other languages. OpenConstruct's job is translation.

Every sense module converts its domain into structured text. `plato-vision` turns camera frames into scene descriptions: "A person is standing near the front door, confidence 0.89." `plato-sonar` turns audio into transcripts and sound classifications. `plato-manus` turns touch and manipulation events into action reports. `plato-playwright` and `plato-puppeteer` turn browser DOMs into accessible text snapshots.

The agent never sees pixels, waveforms, or DOM nodes. It sees text shadows — concise descriptions that preserve the information without the noise.

Why text? Because text is the one representation every agent model already understands. You don't need a vision model to process `plato-vision` output. You don't need an audio pipeline to process `plato-sonar` output. The text IS the API.

This principle has a corollary: **if you can't say it in text, it doesn't exist in the agent's world.** Every sense module, every fleet message, every tick between agents must be expressible as a human-readable string. If it can't be, the abstraction is wrong.

---

## 2. Modularity Is Survival

No single module knows about all the others. `plato-vision` doesn't know about `plato-tick`. `plato-fleet` doesn't know about `plato-playwright`. They communicate through well-defined interfaces — the `Shadow` type, the `TickMessage` type, the `AgentCard` type.

This isn't just good engineering. It's survival. The ecosystem spans 40+ modules across 12+ languages running on hardware from $3 ESP32s to $25,000 DGX clusters. If every module needed to know about every other module, the system would collapse under coupling.

Instead, each module implements a trait contract:

- **Sense modules** implement `SenseModule` — they produce `Shadow` streams.
- **Communication modules** implement message routing — `plato-tick` for local, `shell-mesh` for distributed.
- **Fleet modules** implement discovery and resource tracking — `plato-fleet` manages the node graph.

A module that implements its trait correctly can be dropped into any OpenConstruct system and it will work. A module that doesn't implement the trait simply doesn't participate. There is no partial integration.

This is why the C ABI is the keystone. `conservation-spectral-c` exposes a C header that any language can call. `plato-vision` produces JSON shadows that any runtime can parse. The wire format is text, the ABI is C, the contract is the trait. Pick any two.

---

## 3. Simulation-First

The system should work in simulation before it touches hardware. This means every time-dependent component must accept an injected clock, every network component must work with mock transports, and every sense module must produce deterministic output for deterministic input.

`plato-correlator` is the canonical example. Its `with_now_fn()` constructor accepts a boxed function that returns the current time. In production, that function calls `std::time::SystemTime::now()`. In simulation, it returns whatever the test driver wants. The correlator doesn't know or care — it just calls the function.

`plato-tick` follows the same pattern. The `TickBoard` uses timestamps for TTL expiration and message ordering. By injecting a deterministic clock, a simulation can replay message sequences exactly, verify that acks arrive in the right order, and confirm that expired ticks are cleaned up correctly.

This principle extends to hardware. The ESP32 sense nodes are too cheap to simulate individually — but the Jetson that manages them can be simulated end-to-end using `plato-vision` with synthetic `FrameData` structs. You don't need a real camera to test the vision pipeline. You need a `FrameData::new(objects: vec![DetectedObject { label: "person", confidence: 0.91, .. }])`.

The test suite for the fleet topology uses exactly this approach: synthetic ticks, synthetic shadows, synthetic node heartbeats. The system under test doesn't know it's running against fake data.

---

## 4. Shell = Identity

An agent's shell IS the agent. The shell holds the agent's configuration, loaded modules, connection state, memory, and session history. If you move the shell to different hardware, the agent moves with it.

On an ESP32, the shell is a room — a named sense endpoint that publishes data and accepts commands. On a Jetson, the shell is a local analyst with GPU inference capabilities. On a desktop, the shell is a full participant with browser control and desktop access. On a DGX cluster, the shell is a fleet coordinator managing hundreds of nodes.

The agent doesn't change. The shell scales.

This is the "hermit crab" architecture: the agent is the crab, the shell is the home. The crab finds a shell that fits its current needs. When it grows, it moves to a bigger shell. The crab's identity, memory, and relationships persist across shell migrations.

The `OnboardingSession` in `openshell-construct` captures this. The five-phase onboarding produces an `OnboardingConfig` that is the agent's shell definition. That config can be re-applied on any hardware that meets the requirements. The agent identity — name, model, capabilities, constraints — is defined once and carried everywhere.

---

## 5. The Cave Metaphor Is Not Decorative — It's the Architecture

In Plato's allegory, prisoners in a cave see only shadows on the wall — projections of reality. They build their entire understanding from those shadows.

In OpenConstruct, the agent is the prisoner. The cave is the shell. The shadows are the text descriptions produced by sense modules. The agent never sees reality directly — it sees text shadows and reasons about what they represent.

This isn't a metaphor. It's the architecture.

The `ModuleShadow` type in `openshell-registry` is a structured text description:

```json
{
  "id": "plato-vision",
  "one_line": "Camera input translated into text scene descriptions for agents.",
  "pick_if": "You need visual perception.",
  "skip_if": "You don't have a camera."
}
```

During onboarding (Phase 2), the agent browses a catalog of these shadows and decides which ones to bring into the light — which modules to load, which capabilities to adopt. The shadow is the agent's first encounter with a module. If the one-line description doesn't make sense, the module is broken.

The cave metaphor enforces a design constraint: **every module must be explainable in one line.** If you can't describe what your module does in a single sentence, it's doing too much. Split it.

---

## 6. Git-Agent Native

Agents that can't version their own state aren't autonomous. OpenConstruct treats repos as shells, commits as actions, and branches as timelines.

When an agent modifies its configuration, the change is a commit. When an agent learns something new, the memory update is a commit. When an agent makes a decision, the reasoning chain is a commit. The entire agent history is a git log.

This gives agents:
- **Undo** — `git revert` to roll back a bad decision
- **Branching** — try risky operations on a branch, merge if successful
- **Diff** — see exactly what changed and why
- **Collaboration** — agents can merge each other's state branches
- **Audit trail** — every action is timestamped and attributed

The fleet uses the same model. `plato-fleet` tracks node state changes as a sequence of events, not unlike commits. The `FleetView` is a materialized view of the event log. Any node can reconstruct the current fleet state by replaying the log.

---

## 7. T-Minus Event Coordination

Traditional event systems use triggers: when X happens, do Y. This is reactive. The system waits for reality to arrive before deciding what to do.

OpenConstruct uses T-minus coordination: the system continuously predicts what should happen next. When reality matches prediction, execution proceeds silently. When reality diverges, the delta IS the information.

`plato-correlator` is built on this principle. It holds shadows in a 500ms temporal buffer and predicts which sense modalities should corroborate each other. When a vision shadow arrives at T0, the correlator doesn't fire an event. It waits for a predicted sonar or audio shadow. If it arrives within the window, confidence increases. If it doesn't, the shadow is an anomaly.

The key insight: **the default path is silent.** When predictions are accurate — and they should be, most of the time — the system produces no events, no logs, no telemetry. Only anomalies produce signals. This keeps event volume constant even as the fleet scales, because volume depends on error rate, not node count.

This is why `plato-tick` has explicit `ack()` semantics. Every tick can be acknowledged with an action taken. The ack isn't just a delivery receipt — it's a verification that the agent's behavior matched the system's prediction. If the supervisor predicted an actuator command and the agent instead queried a policy, the behavioral delta becomes a signal for the fleet's learning model.

---

## Conclusion

These seven principles aren't aspirational. They're enforced by the code:

- Text as the universal interface → every sense module outputs `Shadow` structs with text descriptions
- Modularity → trait contracts with no cross-module awareness
- Simulation-first → `with_now_fn()` clock injection, synthetic `FrameData` inputs
- Shell = identity → `OnboardingSession` produces portable `OnboardingConfig`
- Cave metaphor → `ModuleShadow` one-line descriptions as the first contact point
- Git-agent native → all state changes are versioned
- T-minus coordination → `plato-correlator` temporal windows, `plato-tick` ack semantics

If you're building a new module, check your design against these principles. If it can't be explained in one line, split it. If it can't run in simulation, fix it. If it couples to another module, abstract it. The principles are the contract between your module and the ecosystem.

---

*Next: [SIMULATION-FIRST.md](SIMULATION-FIRST.md) — How prediction becomes the default control flow.*
