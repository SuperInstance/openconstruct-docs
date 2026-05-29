# Simulation-First Event Coordination

> *Agents predict, reality confirms, the delta IS the information.*

Traditional event systems are built on triggers: when X happens, do Y. This model collapses under the latency, uncertainty, and scale of a distributed agent fleet. A trigger is reactive — it waits for reality to arrive before deciding what to do next. In a mesh spanning ESP32 motes, Jetson edge nodes, desktop peers, and cloud orchestrators, that wait is too expensive.

Simulation-first coordination inverts this. Instead of reacting to events, the system runs a continuous simulation of expected states. Every agent predicts what should happen next. When reality matches prediction, execution proceeds without hesitation. When reality diverges, the delta is treated as an anomaly, and agents re-plan.

This document describes how OpenConstruct applies simulation-first coordination across five domains: sensor fusion, task delegation, fleet coordination, mesh routing, and agent behavior.

---

## 1. Core Principle: Prediction as the Default Path

In a simulation-first system, the happy path is not "handle an event." It is "confirm a prediction." The control flow:

1. **Project** — Given current state, compute the most likely next state.
2. **Prepare** — Pre-compute responses, pre-allocate resources, preload context.
3. **Observe** — Wait for reality to arrive.
4. **Verify** — Compare observation against projection.
5. **Proceed or Re-plan** — If within tolerance, execute the prepared path. If not, the delta becomes new state and the loop restarts.

Preparation happens *before* confirmation. The system is always one step ahead of reality.

---

## 2. Sensor Fusion: Predictive Frame Differencing

Sensor motes produce a firehose of telemetry. A traditional pipeline ingests every frame, runs full inference, and emits events. Simulation-first predicts the next frame from previous ones and pays attention only to the delta.

### `plato-vision` as the State Differ

`plato-vision` maintains a `baseline` HashMap of object counts per label. When a new `FrameData` arrives, it doesn't treat the frame as an event. It treats it as a hypothesis test:

- Does the observed object count match the baseline? If not, emit `SceneChange::Appeared` or `SceneChange::Disappeared`.
- Have known objects moved beyond the 5.0-pixel threshold? If so, emit `SceneChange::Moved`.
- Have attributes changed? Emit `SceneChange::Changed`.

Only deviations become signals. A static scene produces zero events, zero downstream load, and zero inference cost.

### `plato-correlator` as the Temporal Predictor

A single `SceneChange` is often ambiguous. A person appearing on camera might be an intruder, a resident, or a shadow. `plato-correlator` resolves ambiguity by predicting which sense modalities should corroborate the change within a temporal window.

The correlator ingests `ShadowRef` objects from multiple sources and holds them in a 500ms temporal buffer. When a vision shadow arrives at T0, the correlator predicts a sonar or audio shadow should arrive at the same location before T0 + 500ms. If the predicted shadow appears, confidence jumps and a unified `FusedEvent` is emitted. If the window expires, the shadow is flushed as a low-confidence anomaly.

The `with_now_fn()` constructor is the simulation hook — inject a deterministic clock, advance time artificially, and verify that predictions resolve exactly as expected.

---

## 3. Task Delegation: Predictive Completion Tracking

In a multi-agent fleet, tasks are delegated across tiers. A desktop agent might ask a Jetson to run vision inference; a DGX node might ask a Jetson to collect snapped inferences for retraining.

### Predicted Timeline

When a task is posted to `plato-tick`, the delegator attaches a predicted completion timestamp — not a deadline, but a model output based on historical task duration, current node load, and input size:

- `posted_at`: wall-clock or simulated time when the task was created
- `predicted_done_at`: the model's expectation of when the result should arrive
- `ttl_ms`: the hard drop-dead time after which the task is abandoned

As the worker progresses, it emits intermediate ticks back to the delegator. The delegator compares `actual_progress` against `predicted_progress`. If the worker is ahead, pre-stage the next task. If behind, re-delegate before the original fails.

### Anomaly = Re-delegation Signal

A delta between predicted and actual progress is a first-class signal. If a Jetson's inference latency is 20% above prediction for three consecutive tasks, the simulation model updates its parameters and begins routing to an alternative node *before* the current one fails.

---

## 4. Fleet Coordination: Predictive Resource Pre-allocation

The fleet topology spans heterogeneous compute tiers. Reactive resource allocation introduces cold-start latency that violates real-time constraints.

### Demand Forecasting

Each Jetson hub maintains a local model of its ESP32 cluster's demand patterns. It predicts which zones are likely to require vision inference in the next 30 seconds based on time-of-day, recent PIR activity, and calendar data. It pre-allocates GPU memory and pre-loads TensorRT engines before demand materializes.

The prediction is expressed as expected `Tick` arrivals. If `esp32-front-door-01` has 0.85 probability of firing a PIR snap in the next window, the Jetson warms the vision pipeline. When the actual tick arrives, inference begins immediately — a confirmation, not a cold start.

### Pre-allocated Paths

The DGX orchestrator runs a higher-level simulation. It pre-stages policy bundles and engine artifacts on Jetson nodes before they are requested. The 30-second `FleetView` flush is a confirmation window, not a polling loop. Divergence triggers reconciliation.

---

## 5. Shell Mesh: Predictive Message Routing

`shell-mesh` is the transport fabric that carries ticks between distributed agents. In a simulation-first architecture, routing is pre-computed and verified on delivery.

### Pre-computed Path Hypotheses

Before a tick is sent, the mesh layer predicts the optimal path based on topology, link latency, and node load. It generates a `PathHypothesis` — an ordered list of hops with expected arrival times. The tick is annotated and released.

As the tick traverses each hop, the relay timestamps arrival and compares against prediction. Within tolerance → forward silently. Late → emit a `RouteAnomaly` shadow. Early → adjust the latency model.

### Verification on Delivery

The destination `ack()`s the tick through `plato-tick`, including observed path and timing deltas. The sender's simulation loop consumes these acks and updates its routing model. Over time, pre-computed paths converge to ground truth without requiring a dedicated measurement protocol.

---

## 6. Agent Behavior: Predictive Command Preloading

An agent in an OpenShell sandbox is a sequential decision-maker. A simulation-first shell predicts the agent's next command and preloads context.

### Command Prediction

The supervisor monitors the agent's command history and builds a lightweight Markov model over the next command. If the agent has queried an ESP32 room three times in the last second, high probability goes to a fourth query or an actuator command.

The supervisor preloads:
- The room's latest snaps from local SQLite cache
- The actuator exit schema for validation
- The TLS session or MQTT topic handle for the target mote

When the agent emits the command, it executes against already-resident state. No database query, no network round-trip.

---

## 7. The Simulation Loop

These five domains share a single simulation loop at the heart of every node:

```
Predict → Prepare → Observe → Verify → Proceed/Re-plan → (back to Predict)
```

1. **Predict** — `plato-vision` projects the next frame. `plato-correlator` predicts corroboration. Task models predict completion. Fleet models predict demand. Mesh models predict routes. Agent models predict commands.
2. **Prepare** — Resources pre-allocated, contexts preloaded, paths pre-computed, ticks staged with predicted timestamps.
3. **Observe** — Reality arrives: a frame, a shadow, a progress tick, a resource request, a delivered message, an agent command.
4. **Verify** — Observation compared against prediction. `plato-tick`'s `ack()` confirms delivery. `plato-vision`'s `track_changes()` confirms scene stability. `plato-correlator`'s temporal window confirms cross-sense fusion.
5. **Proceed or Re-plan** — Within tolerance → execute. Outside → delta becomes new state, loop restarts.

---

## 8. Operational Invariants

1. **Time is injectable.** `with_now_fn()` and timestamp-based filtering must support deterministic clocks.
2. **Predictions are first-class.** Every prediction must be observable, comparable, and storable.
3. **Anomalies are signals, not errors.** Deltas are typed, routed, and consumed by the re-planning subsystem.
4. **The default path is silent.** When reality matches prediction, produce minimal telemetry. Volume depends on error rate, not node count.

---

## 9. Module Reference

| Module | Simulation Role | Confirmation Primitive |
|---|---|---|
| `plato-vision` | Projects next frame from baseline; only deviations become events | `track_changes()` returns empty when prediction matches |
| `plato-correlator` | Predicts cross-sense corroboration within 500ms window | Fused event confidence reflects prediction accuracy |
| `plato-tick` | Stages predicted completion times; distributes events with `ack()` tracking | `ack()` verifies delivery and behavioral match |
| `shell-mesh` | (Future) Pre-computes routing hypotheses; verifies on delivery | Per-hop timestamp comparison against predicted path |

These modules expose the primitives — deterministic time, baseline tracking, temporal windows, delivery acknowledgments — required for simulation-first operation. The loop described in this document uses them as-is, arranged so prediction precedes observation and confirmation precedes execution.

---

*Next: [PLATO-SENSORY.md](PLATO-SENSORY.md) — The sense module design across vision, sonar, manus, browser, and voice.*
