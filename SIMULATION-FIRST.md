# Simulation-First Event Coordination

Traditional event systems arebuilt on triggers: when X happens, do Y. This model works for simple pipelines, but it collapses under the latency, uncertainty, and scale of a distributed agent fleet. A trigger is reactive; it waits for reality to arrive before deciding what to do next. In a mesh spanning ESP32 motes, Jetson edge nodes, desktop peers, and cloud orchestrators, that wait is too expensive. By the time a trigger fires, the moment to act has already passed.

Simulation-first coordination inverts this. Instead of reacting to events, the system runs a continuous simulation of expected states. Every agent predicts what should happen next. When reality matches prediction, execution proceeds without hesitation. When reality diverges,the delta is treated as an anomaly, and agents re-plan. There is no explicit "when X happens" clause. There is only the gap between expected and observed, and the machinery to close it.

This document describes how OpenConstruct applies simulation-first coordination across five domains: sensor fusion, task delegation, fleet coordination, shell mesh routing, and agent behavior. It connects the concept to four concrete modules: `plato-correlator`, `plato-vision`, `shell-mesh`, and `plato-tick`.

---

## 1. Core Principle: Prediction as the Default Path

In a simulation-first system, the happy path is not "handle an event." It is "confirm a prediction." The control flow looks like this:

1. **Project** —Given current state, compute the most likely next state.
2. **Prepare** — Pre-compute responses, pre-allocate resources, preload context.
3. **Observe** — Wait for reality to arrive.
4. **Verify** — Compare observation against projection.
5. **Proceed or Re-plan** — If the error is within tolerance, execute the prepared path. If not, treat the delta as a new state and return to step 1.

The critical shift is that preparation happens *before* confirmation. The system is always one step ahead of reality, and its busyness is proportional to the accuracy of its model, not the volume of raw events.

---

## 2. Sensor Fusion: Predictive Frame Differencing

Sensor motes produce a firehose of low-level telemetry. A traditional pipeline would ingest every frame, run full inference, and emit events for downstream consumers. In simulation-first mode, the system predicts the next frame from the previous ones and only pays attention to the delta.

### 2.1 Plato-Vision as the State Differ

`plato-vision` maintains a `baseline` HashMap of object counts per label across the scene. When a new `FrameData`arrives, it does not treat the frame as an event. It treats it as a hypothesis test:

- Does the observed object count match the baseline? If not, emit `SceneChange::Appeared` or `SceneChange::Disappeared`.
- Have known objects moved beyond the 5.0-pixel threshold? If so, emit `SceneChange::Moved`.
- Have object attributes changed? Emit `SceneChange::Changed`.

Only deviations from expectation become signals. A static scene produces zero events, zero downstream load, and zero inference cost. The vision module becomes a compression layer that discards predictable sensor noise.

### 2.2 Plato-Correlator as the Temporal Predictor

A single `SceneChange` is often ambiguous. A person appearing on a camera might be an intruder, a resident, or a shadow. `plato-correlator` resolves ambiguity by predicting which sense modalities should corroborate the change within atemporal window.

The correlator ingests `ShadowRef` objects from multiple sources—vision, sonar, radar, audio—and holds them in a 500 ms temporal buffer. Itscore assumption is that correlated events from different sources are not independent arrivals; they are confirmations of a single underlying state transition that the system has already predicted.

When a vision shadow arrives at T0, the correlator does not immediately emit a fused event. It predicts that a sonar or audio shadow should arrive at the same location before T0 + 500 ms. If the predicted shadow appears, confidence jumps and a unified `FusedEvent` is emitted. If the window expires without confirmation, the shadow is flushed as a low-confidence anomaly or dropped entirely.

The `with_now_fn` constructor is the simulation hook. A test or orchestrator can inject a deterministic clock, advance time artificially, and verify that the correlator's predictions resolve exactly as expected. The module is designed to run inside a simulation loop as naturally as it runs on wall-clock time.

---

## 3. Task Delegation: Predictive Completion Tracking

In a multi-agent fleet, tasks are delegated across tiers. A desktop agent might ask a Jetson to run a vision inference; a DGX node might ask a Jetson to collect a batch of snapped inferences for retraining. A trigger-based system would fire a "task complete" event and react. A simulation-first system predicts completion time and compares it against actual progress.

### 3.1 Predicted Timeline

When a task is posted to `plato-tick`, the delegator attaches a predicted completion timestamp. This is not a deadline; it is a model output based on historical task duration, current node load, and input size. The `Tick` carries:

- `posted_at`: wall-clock or simulated time when the task was created.
- `predicted_done_at`: the model's expectation of when the result should arrive.
- `ttl_ms`: the hard drop-dead time after which the task is abandoned.

Asthe worker progresses, it emits intermediate ticks—progress shadows—back to the delegator. The delegator compares `actual_progress` against `predicted_progress`. If the worker is ahead of schedule, the delegator may pre-stage the next task.If it is behind, the delegator may re-delegate to another node before the original task fails.

### 3.2 Anomaly = Re-delegation Signal

A delta between predicted and actual progress is not a log line. It is a first-class signal. If the Jetson's inference latency is 20% above prediction for three consecutive tasks,the desktop agent's simulation model updates its parameters and begins routing similar tasks to an alternative Jetson *before* the current one fails. The anomaly becomes the trigger for re-planning, but the re-planning itself was prepared in advance because the system was already running a completion model.

---

## 4. Fleet Coordination: Predictive Resource Pre-allocation

The fleet topology described in `FLEET-TOPOLOGY.md` spans heterogeneous compute tiers. A DGX orchestrator manages thousands of ESP32 motes through dozens of Jetson hubs. Reactive resource allocation—spinning up inference workers when camera frames arrive—introduces cold-start latency that violates real-time constraints.

### 4.1 Demand Forecasting

Each Jetson hub maintains a local model of its ESP32 cluster's demand patterns. It predicts, based on time-of-day, recent PIR activity, and upstream calendar data, which zones are likely to require vision inference in the next 30 seconds. It pre-allocates GPU memory and pre-loads the corresponding TensorRT engine before the demand materializes.

The prediction is expressed asa set of expected `Tick` arrivals. The Jetson's simulation loop runs every 100 ms and outputs a probability distribution over its mote IDs. If `esp32-front-door-01` has a 0.85 probability of firing a PIR snap in the next window, the Jetsonwarms the vision pipeline for that room. When the actual tick arrives, inferencebegins immediately. The observed tick confirms the prediction rather than initiating a cold start.

### 4.2 Pre-allocated Paths

The DGX orchestrator runs ahigher-level simulation. It predicts aggregate resource needs across the fleet based on seasonal patterns, maintenance schedules, and model update rollouts. It pre-stages policy bundles and engine artifacts on Jetson nodes before they are requested. The 30-second `FleetView` flush interval is not a polling loop; it is a confirmation window. The DGX expects each Jetson to report state that matches its prediction. Divergence triggers a reconciliation pass.

---

## 5. ShellMesh: Predictive Message Routing

`shell-mesh` is the transport fabric that carries ticks between distributed agents. In its current form it is a lightweight placeholder, but its intended role is to provide the mesh networking protocol for interconnected Plato shells. In a simulation-first architecture, routing is not computed on message arrival. It is pre-computed and verified on delivery.

### 5.1 Pre-computed Path Hypotheses

Before a tick is sent, the mesh layer predicts the optimal path from source to destination based on the current topology graph, link latency models, and node load. It generates a `PathHypothesis`: an ordered list of hops with expected arrival times at each step. The tick is annotated with this hypothesis and released into the network.

As the tick traverseseach hop, the relay node timestamps arrival and compares it against the predicted time. If the tick arrives within tolerance, the relay forwards it without ceremony. If it is late, the relay emits a `RouteAnomaly` shadow that feeds back into the topology model. If it is early, the model learns that the latency estimatewas conservative and adjusts.

### 5.2 Verification on Delivery

The destination node does not treat receipt as success. It treats receipt as confirmation of the final hop in the path hypothesis. It `ack()`s the tick through `plato-tick`, but the ack payload includes the observed path and timing deltas. The sender's simulation loop consumes these acks and updates its routing model. Over time,the pre-computed paths converge to ground truth without ever requiring a dedicated network measurement protocol.

---

## 6. Agent Behavior: Predictive Command Preloading

An agent running inside an OpenShell sandbox is a sequential decision-maker. It reads state, plans, and emits commands. A trigger-based shell would execute each command as it is generated. A simulation-first shell predictsthe agent's next command and preloads the context needed to execute it instantly.

### 6.1 Command Prediction

The sandbox supervisor monitors the agent'srecent command history and current query context. It builds a lightweight Markovmodel (or delegates to a small neural predictor) that outputs a probability distribution over the next command. If the agent has queried an ESP32 room three times in the last second, the predictor assigns high probability to a fourth query or an actuator command on that room's exits.

The supervisor preloads:
- Theroom's latest snaps from local SQLite cache.
- The actuator exit schema for validation.
- The TLS session or MQTT topic handle for the target mote.

When the agent actually emits the command, it executes against already-resident state.The command does not wait for a database query or a network round-trip. The delta between predicted and actual command is usually zero. When it is not, the preloaded context is discarded and the correct path is fetched reactively—a fallbackthat is acceptable because it is rare.

### 6.2 Context Verification

The agent's `ack()` of a tick is not just a delivery receipt. It is a verification that the agent's behavior matched the supervisor's prediction. If the supervisorpredicted an actuator command and the agent instead emitted a policy query, the delta is logged as a behavioral anomaly. Fleet-wide, these anomalies feed a model that learns each agent's decision boundaries, improving prediction accuracy for the entire mesh.

---

## 7. Integration: The Simulation Loop

These five domains are not independent. They share a single simulation loop that runs atthe heart of every node:

```text
┌─────────────────────────────────────────────────────────────┐
│                      Simulation Loop                        │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐  │
│  │  Predict    │───>│   Prepare   │───>│    Observe      │  │
│  └─────────────┘    └─────────────┘    └─────────────────┘  │
│         ^                                        │          │
│         └─────────────────────────────────────────┘         │
│                            Verify                            │
└─────────────────────────────────────────────────────────────┘
```

1. **Predict** — `plato-vision` projects the next frame. `plato-correlator` predicts whichshadows will corroborate. The task model predicts completion. The fleet model predicts demand. The mesh model predicts routes. The agent model predicts commands.

2. **Prepare** — Resources are pre-allocated, contexts are preloaded, pathsare pre-computed, and ticks are staged in `plato-tick`'s board with `predicted_done_at` timestamps.

3. **Observe** — Reality arrives: a frame, a shadow, a progress tick, a resource request, a delivered message, an agent command.

4. **Verify** — The observation is compared against the prediction. `plato-tick`'s `ack()` mechanism confirms delivery. `plato-vision`'s `track_changes()` confirms scene stability. `plato-correlator`'s temporal window confirms cross-sense fusion.

5. **Proceed or Re-plan** — If the error is within tolerance, the prepared path executes. If not, the delta becomes the new state and the loop restarts.

---

## 8. Operational Invariants

Simulation-first coordination imposes strict invariants on the modules:

1. **Time is injectable.** `plato-correlator`'s `with_now_fn` and `plato-tick`'s timestamp-based filtering must supportdeterministic clocks. A simulation that depends on `std::time::Instant::now()` is not a simulation; it is a replay.

2. **Predictions are first-class.** Everyprediction must be observable, comparable, and storable. A prediction that livesonly in a local variable cannot be verified. `plato-tick` stores predicted completion times as fields on `Tick` objects. `shell-mesh` will store path hypothesesas message metadata.

3. **Anomalies are signals, not errors.** A delta between prediction and observation is the primary control signal. It must be typed, routed, and consumed by the re-planning subsystem. It is not a log line for human operators.

4. **The default path is silent.** When reality matches prediction, the system should produce minimal telemetry. Telemetry is emitted for anomalies, not for confirmations. This keeps the event volume constant even as the fleetscales, because the volume depends on the error rate, not the node count.

---

## 9. Relationship to Existing Modules

| Module | Simulation Role | Confirmation Primitive |
|---|---|---|
| `plato-vision` | Projects next frame from baseline; only deviations become events. | `track_changes()` returns empty when prediction matches. |
| `plato-correlator` | Predicts cross-sense corroborationwithin 500 ms window. | Fused event confidence reflects prediction accuracy. |
| `plato-tick` | Stages predicted completion times; distributes events with `ack()` tracking. | `ack()` verifies delivery and behavioral match. |
| `shell-mesh` | (Future) Pre-computes routing hypotheses; verifies on delivery. | Per-hop timestamp comparison against predicted path. |

These modules do not need to be wrapped in a new coordination framework. They already expose the primitives—deterministic time, baseline tracking, temporal windows, delivery acknowledgments—required for simulation-first operation. The architecture described in this document uses them as-is, arranged into a loop where prediction precedes observation and confirmation precedes execution.

---

## 10. From Trigger to Expectation

The shift from trigger-based to simulation-first coordination is a shift in ontology. A trigger asks, "What happened?" An expectation asks, "What should be happening?" The former produces event-driven systems that scale with sensory volume. The latter produces model-driven systems that scale with prediction accuracy.

In OpenConstruct, accuracy is improved by grounding predictions in the actual modules that observe and transmit state. `plato-vision` grounds visual predictions in pixel deltas. `plato-correlator` grounds temporal predictions in cross-sense fusion. `plato-tick` grounds delivery predictions in explicit acknowledgments. `shell-mesh` will ground routing predictions in path verification. The simulation loop is not a separate layer; it is the emergent behavior of these modules running with prediction as their default posture.

When the front-door PIR fires, the Jetson does not ask, "What should I do?" It already predicted the firing, pre-loaded the vision model, and computed the likely inference outcome. Theactual PIR tick is a confirmation. The camera frame is a confirmation. The visitor inference is a confirmation. Only if a shadow fails to arrive in the 500 ms window—only if reality diverges—does the system wake up and re-plan. Until then, it is already one step ahead, running quietly in the gap between expected and observed, closing it before the rest of the fleet notices it was ever open.')StatusUpdate(    context_usage=0.12475204467773438,    context_tokens=32703,    max_context_tokens=262144,    token_usage=TokenUsage(        input_other=4543,        output=3565,        input_cache_read=28160,        input_cache_creation=0    ),    message_id='chatcmpl-s0X1aLCbdMwJZJqTkZoLtxgS',    plan_mode=False,    mcp_status=None)TurnEnd()To resume this session: kimi -r 9bd44b7c-6f66-4cf7-9923-96f98deb2cfa