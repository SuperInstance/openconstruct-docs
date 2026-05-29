# Fleet Topology

OpenConstruct extends the single-player OpenShell sandbox into a self-organizing mesh of heterogeneous compute tiers. A fleet may contain thousands of ESP32 sensor motes, dozens of NVIDIA Jetsonedge nodes, a handful of desktop workstations, and one or more cloud/DGX orchestrators. Each tier speaks a discovery protocol and resource-delegation dialect appropriate to its constraints, while the signal-chain room metaphor unifies them into a single addressable space.

## Tier Summary

| Tier | Compute | Network | Role in Mesh |
|---|---|---|---|
| **ESP32** | Ultralight, no FPU | Wi-Fi / MQTT | Sensor/actuator leaf. Publishes snaps, subscribes to actuator exits. |
| **Jetson** | CUDA edge, 8–32 GB | Ethernet / Wi-Fi / gRPC | Local hub. Runs inference for its ESP32 cluster. Sees each mote as a Plato room. |
| **Desktop**| Full agent runtime | Wired / gRPC | Peer node. Development, coordination, and local policy authoring. |
| **Cloud / DGX** | Heavy GPU cluster | Backbone / gRPC | Fleet orchestrator. Batch inference, model training, global state reconciliation. |

The mesh is *hierarchical at the edges* and *peer-to-peer at the core*. ESP32s never speak gRPC; DGX nodes never poll MQTT topics one-by-one. The Jetson sits at the boundary, translating between the two worlds.

---

## 1. Discovery Protocol

### 1.1 ESP32 Discovery — mDNS + MQTT

An ESP32 mote boots with three credentials burned at provisioning time:

1. `fleet_id` (a 128-bit UUID shared by the entire installation).
2. `mqtt_broker` hostname (default `_mqtt._tcp.local` via mDNS).
3. A PSK used to sign MQTT CONNECT packets.

On boot the mote multicasts an mDNS query for `_mqtt._tcp.local`. Any Jetson or desktop running the local MQTT broker advertises itself with a priority weight derived from free RAM and CUDA utilization:

```
Jetson-07._mqtt._tcp.local.  PTR jetson-07.local.
jetson-07.local.             A    192.168.4.17
                            SRV  0 20 1883 jetson-07.local.
                             TXT "tier=jetson" "cuda=85" "ram_free_gb=6.2"
```

The ESP32 selects the broker with the lowest SRV priority, falls back to the highest weight on ties, and opens an MQTT connection on port 1883. It immediately publishes a retained `birth` message:

```json
{
  "fleet_id": "a1b2c3d4...",
  "mote_id": "esp32-garage-03",
  "caps": ["pir", "magnetic", "dht22", "relay"],
  "dial": 0.0,
  "last_seen": 1716892800
}
```

The topic hierarchy is:

- `fleet/<fleet_id>/birth/<mote_id>` — retained LWT-compatible birth certificate.
- `fleet/<fleet_id>/tick/<mote_id>` — ephemeral snaps and inferences.
- `fleet/<fleet_id>/act/<mote_id>/<exit>` — actuator commands from the hub.

### 1.2 Heavy-Node Discovery — gRPC + mDNS-SD

Jetsons, desktops, and DGX nodes discover each other over gRPCwith mDNS-SD service type `_construct._tcp.local`. Advertisements carry structured metadata:

```protobuf
message NodeAdvert {
  string node_id     = 1;   // "jetson-07.local"
  Tier   tier        = 2;   // JETSON, DESKTOP, DGX
  repeated string roles = 3; // ["vision_hub", "mqtt_bridge"]
  uint32 cuda_sm     = 4;   // streaming-multiprocessor count
  float  load_1m     = 5;   // CPU load
  bytes  tls_cert_hash = 6; // SHA-256 of the node's mTLS cert
}
```

Heavy nodes maintain a gossip sub-protocol over a bidirectional gRPC stream (`/construct.v1.Fleet/Gossip`). Every 5 seconds each node pushes a `NodeHeartbeat` containing its current load, the set of mote IDs it bridges, and the dial position itrecommends for its leaf cluster. The DGX master aggregates these into a `FleetView` protobuf that is flushed to SQLite-backed gateway state every 30 seconds.

---

## 2. Hierarchical Mesh

### 2.1 Jetson as Hub

A Jetson is the default gateway for every ESP32 within Wi-Fi range. It runs three local services:

- **MQTT broker** (mosquitto or nanomq) bound to `0.0.0.0:1883`.
- **gRPC bridge** (`openshell-fleet-bridge`) that subscribes to `fleet/+/tick/+` and republishes snaps into local signal-chain rooms.
- **Vision inference worker** (TensorRT or ONNX Runtime) that services `inference.local` requests from the bridge.

Each ESP32 is represented inside the Jetson supervisor as a **Plato room** namedafter its `mote_id`. The room’s dial is clamped to `DIAL_BATHY` (0.10) because sensor data is treated as hard fact unless explicitly softened by a downstream inference worker.

### 2.2 Desktop Peer Ring

Desktops do not bridge MQTT. They join a gossip peer ring with other desktops and visible Jetsons. A desktop may:

- Pull `FleetView` snapshots from the DGX to render a TUI of the entire installation.
- Push policy bundles (YAML) to the DGX, which fans them out to relevant Jetson supervisors.
- Request relay tunnels into a specific Jetson sandboxfor live debugging.

Desktop-to-desktop traffic uses gRPC with mTLS backed by the same fleet CA that signs ESP32 PSKs.

### 2.3 DGX as Orchestrator

The cloud/DGX tier hosts the authoritative gateway and the training pipeline. It maintains:

- The canonical `FleetView` database.
- Global policy revision history.
- A batched inference queue for jobs too large for edge GPUs.
- Model retraining jobs that produce updated TensorRT engines for the Jetsons.

DGX nodes are the only tier that may open connections *into* Jetsons (via supervisor relay forwarding). All other traffic is supervisor-initiated outbound, preserving the OpenShell security model.

---

## 3. Resource Delegation

### 3.1 Vision Delegation — Jetson to ESP32 Cluster

An ESP32 has no camera and no FPU. When a PIR snap fires, the Jetson agent *visits* the ESP32 room, reads the snap, and decides whether to run vision inference on a locally attached USB camera that covers the same physical zone:

```rust
// Inside the Jetson supervisor sandbox
let garage = chain.room("esp32-garage-03");
garage.add_snap(json!({"pir": true, "timestamp_ms": 1716892800123}), 1.0);

// Query at DIAL_ANALYSIS (0.40) toinclude low-confidence hypotheses
let context = garage.query(Dial::analysis());
if context.iter().any(|r| r.has_snap("pir")) {
    // Delegate to local GPU
   let inference = vision_model.infer(camera_frame).await?;
    garage.add_inference(
        json!({"visitor_detected": inference.label, "confidence": inference.confidence}),
        inference.confidence,
    );
}
```

The inference is stored as a *soft* signal in the ESP32 room, not pushed back to the mote. The mote only receives actuator commands.

### 3.2 Batch Delegation — DGX to Fleet

When a desktop requests a fleet-wide anomaly model, the DGX does not stream individual camera frames. Instead it:

1. Queries all Jetsons for their last 10 000 snapped inferences.
2. Runs batch embedding extraction on A100/H100 GPUs.
3. Returns a compressed TensorRT engine to each Jetson via the existing supervisor-initiated relay.

This respects the OpenShell boundary: the DGX never initiates raw TCP into a Jetson home network. The Jetson supervisor polls the DGX for available engine artifacts and applies them locally.

---

## 4. Failover

### 4.1 Jetson Death

If a Jetson stops heartbeating for >30 seconds, the DGX marks it `DOWN` in `FleetView` and triggers a failover sequence:

1. **MQTT broker migration.** The ESP32s connected to the dead Jetson have a last-will Testament (LWT) published by the broker on their behalf: `{"state":"orphan","last_broker":"jetson-07"}`. Surviving Jetsons subscribe to `fleet/+/birth/+/state` and race to adopt orphans.
2. **Adoption election.** The surviving Jetson with the lowest load_1m and highest RAM wins. It sends an MQTT `ADOPT` control packet to the orphan mote’s `act` topic.
3. **Mote reconnect.** The ESP32, upon receiving `ADOPT`, disconnects from its stale broker socket and reconnects to the new one using the same PSK and `fleet_id`.
4. **Room hand-off.** The adopting Jetson imports the orphan’s last retained `birth` certificate, reconstructs the Plato room from persisted SQLite state in the DGX (synced every 30 s), and resumesinference duties.

The entire sequence completes in <2 seconds for a cluster of 50 ESP32s.

### 4.2 DGX Partition

If the cloud link drops, Jetsons fall back to a **local quorum** of known desktops. They cache the last policy bundle and continue operating in *autonomous mode* with a capped dial of `DIAL_ANALYSIS`(0.40) to prevent ungrounded soft inferences from triggering physical actuators without oversight.

---

## 5. Room Metaphor

In OpenConstruct, every ESP32 is a **room** that the Jetson agent can visit. The room contains:

- **Sensor objects** (`Snap` instances) — hard telemetry: temperature, magnetic reed switch state, LUX readings.
- **Actuator exits** — named output channels mapped to MQTT topics under `fleet/<fleet_id>/act/<mote_id>/<exit>`.

A concrete room layout for `esp32-front-door-01`:

```yaml
room: esp32-front-door-01
dial: 0.10  # DIAL_BATHY — sensor facts only
objects:
  - name: magnetic_reed
    type: Snap
    value: { closed: false }
    confidence: 1.0
  - name: pir_motion
    type: Snap
    value: { motion: true, zone: "porch" }
    confidence: 1.0
  - name: ambient_light
    type: Snap
    value: { lux: 12.4 }
    confidence: 1.0
exits:
  - name: door_lock
    topic: act/esp32-front-door-01/door_lock
    payload_schema: { lock: bool }
  - name: porch_light
    topic: act/esp32-front-door-01/porch_light
    payload_schema: { intensity: uint8 }
```

When the Jetson agent *enters* the room (i.e., the local bridge worker processes a new MQTT tick), it sees the latest snaps arranged on the walls. It may leavean `Inference` (a hypothesis note) on the room’s cork-board, but it cannot altera Snap. Only the ESP32 itself can update a Snap by publishing a new tick.

---

## 6. Tick Propagation

Ticks are the atomic unit of fleet cognition. A tick is a signed JSON blob with a monotonic sequence number, a timestamp, and a payload. Ticks flow upward through the mesh, growing in semantic richness at each tier.

### 6.1 Concrete Data Flow — Front-Door Intrusion Scenario

**T0 — ESP32 leaves a tick.**

The front-door mote detects motion and publishes:

```json
{
  "tick_id": "esp32-front-door-01:7842",
  "seq": 7842,
  "ts_ms": 1716892800123,
  "dial": 0.0,
  "snaps": [
    { "key": "pir_motion", "value": { "motion": true, "zone": "porch" }, "conf": 1.0 },
    { "key": "magnetic_reed", "value": { "closed": false }, "conf": 1.0 }
  ]
}
```

Topic: `fleet/a1b2c3d4/tick/esp32-front-door-01`

**T1 — Jetson reads the tick.**

The Jetsonbridge worker routes the tick into the local signal-chain room `esp32-front-door-01`. The room now contains two new snaps. The Jetson agent, running inside its sandbox with `Dial::analysis()`, queries the room and sees a potential security event (door open + motion). It captures a camera frame and runs TensorRT inference:

```rust
let room = chain.room("esp32-front-door-01");
room.add_snap(tick.into_snap("pir_motion"), 1.0);
room.add_snap(tick.into_snap("magnetic_reed"),1.0);

let visitor = vision.infer(frame).await?;
room.add_inference(
    json!({"visitor": visitor.label, "bounding_box": visitor.bbox}),
    visitor.confidence,
);
```

The Jetson now leaves its own tick for upstream nodes:

```json
{
  "tick_id": "jetson-07:4911",
  "parent": "esp32-front-door-01:7842",
  "seq": 4911,
  "ts_ms": 1716892800456,
  "dial": 0.40,
  "snaps": [
    { "key": "pir_motion", "value": { "motion": true, "zone": "porch" }, "conf": 1.0},
    { "key": "magnetic_reed", "value": { "closed": false }, "conf": 1.0 }
 ],
  "inferences": [
    { "key": "visitor", "value": { "label": "human", "bbox": [120,80,340,410] }, "conf": 0.91 }
  ]
}
```

**T2 — Desktop reads the Jetson tick.**

The desktop pulls aggregated ticks from the Jetson via gRPC (`/construct.v1.Fleet/PollTicks`). It sees the `visitor` inference with confidence0.91. The desktop agent, operating at `Dial::review()` (0.50), decides this crosses the action threshold. It publishes a command tick back down the chain:

```json
{
  "tick_id": "desktop-alice:332",
  "parent": "jetson-07:4911",
  "seq": 332,
  "ts_ms": 1716892800890,
  "dial": 0.50,
  "act": [
    { "exit":"door_lock", "payload": { "lock": true } },
    { "exit": "porch_light", "payload": { "intensity": 255 } }
  ]
}
```

**T3 — Jetson relays to ESP32.**

The Jetson strips the `act` array and forwards each command to the appropriate MQTT topic:

- `fleet/a1b2c3d4/act/esp32-front-door-01/door_lock` → `{ "lock": true }`
- `fleet/a1b2c3d4/act/esp32-front-door-01/porch_light` → `{ "intensity": 255 }`

The ESP32 applies the commands, updates its local actuator state, and publishes confirmation snaps in the next tick.

### 6.2 Cascade Rules

When a Jetson produces a high-confidence inference (≥ 0.85) in an ESP32 room, the signal-chain `cascade` primitive promotes it downward into *sibling* rooms that share the same physical zone. For example, a `visitor_detected` inference in `esp32-front-door-01` cascades as a snap into `esp32-porch-02` at 0.68 confidence (0.85 × 0.8 decay). This lets nearby motes coordinate without waiting for the desktop tier.

---

## 7. Security Boundaries

The mesh inherits OpenShell’s supervisor-gateway split:

- **ESP32 → Jetson:** MQTT over TLS-PSK. The PSK is fleet-scoped; compromise of one mote does not grant access to others because theJetson broker enforces topic ACLs by `mote_id`.
- **Jetson → Desktop/DGX:** mTLS with SPIFFE workload identity. The supervisor initiates outbound gRPC and keeps the connection alive; no inbound firewall holes required.
- **Actuator commands:** Every `act` tick is signed by the desktop agent’s private key. The Jetson verifies the signature against the fleet CA before relaying to MQTT. Unsigned actuator messages are dropped by the bridge.

---

## 8. Operational Invariants

1. **Snaps only flow up.** An ESP32 snap may be read by a Jetson, but a Jetson may never write a snap into an ESP32 room. The ESP32 owns its own facts.
2.**Inferences only flow down.** A desktop inference may cascade into a Jetson room, but it may never overwrite an ESP32 snap. Inferences are advice, not truth.
3. **Actuators only fire on signed exits.** Unsigned or replayed actuator commands are rejected by the bridge’s policy proxy.
4. **Dial hardens under partition.** When a Jetson loses DGX contact, its local dial ceiling drops to 0.40, preventing soft hallucinations from controlling physical hardware without fleet oversight.

These invariants keep the mesh safe even when nodes fail, partitions heal slowly, or models hallucinate at the edge.',                old_start=1,                new_start=1,                is_summary=False            )        ],        extras=None    ))StepBegin(n=7)ThinkPart(    type='think',    think='The user wants me to output ONLY the markdown. I wrote the file to OpenConstruct/architecture/FLEET-TOPOLOGY.md, but the user said "Output ONLY the markdown." This likely means they want the markdown content as my response, not just written to a file. However, since I already wrote the file, I should also present the markdown content in my response. Let me read the file and output it.

Wait, actually the user said "Output ONLY the markdown" which means my entire response should be just the markdown document. I should read the file and outputit. But actually, I already have the content. Let me just output it directly. But I need to make sure I follow the instruction to output only markdown. Let me read what I wrote and output it.

Actually, looking at the message again: "Write 1500+ words with concrete data flow examples. Output ONLY the markdown."

This means my response should contain only the markdown content. I already wrote it to the file. I should now output the markdown as my response text.',    encrypted=None)TextPart(    type='text',    text='```markdown
# Fleet Topology

OpenConstruct extends the single-player OpenShell sandbox into a self-organizing mesh of heterogeneous compute tiers.A fleet may contain thousands of ESP32 sensor motes, dozens of NVIDIA Jetson edge nodes, a handful of desktop workstations, and one or more cloud/DGX orchestrators. Each tier speaks a discovery protocol and resource-delegation dialect appropriate to its constraints, while the signal-chain room metaphor unifies them into a single addressable space.

## Tier Summary

| Tier | Compute | Network |Role in Mesh |
|---|---|---|---|
| **ESP32** | Ultralight, no FPU | Wi-Fi / MQTT | Sensor/actuator leaf. Publishes snaps, subscribes to actuator exits. |
| **Jetson** | CUDA edge, 8–32 GB | Ethernet / Wi-Fi / gRPC | Local hub. Runs inference for its ESP32 cluster. Sees each mote as a Plato room. |
| **Desktop** | Full agent runtime | Wired / gRPC | Peer node. Development, coordination, and local policy authoring. |
| **Cloud / DGX** | Heavy GPU cluster | Backbone / gRPC | Fleet orchestrator. Batch inference, model training, global state reconciliation. |

The mesh is *hierarchical at the edges* and *peer-to-peer at the core*.ESP32s never speak gRPC; DGX nodes never poll MQTT topics one-by-one. The Jetsonsits at the boundary, translating between the two worlds.

---

## 1. Discovery Protocol

### 1.1 ESP32 Discovery — mDNS + MQTT

An ESP32 mote boots with three credentials burned at provisioning time:

1. `fleet_id` (a 128-bit UUID shared by the entire installation).
2. `mqtt_broker` hostname (default `_mqtt._tcp.local` via mDNS).
3. A PSK used to sign MQTT CONNECT packets.

On boot the mote multicasts an mDNS query for `_mqtt._tcp.local`. Any Jetson or desktop running the local MQTT broker advertises itself with a priority weight derived from free RAM and CUDA utilization:

```
Jetson-07._mqtt._tcp.local.  PTR  jetson-07.local.
jetson-07.local.             A    192.168.4.17
                            SRV  0 20 1883 jetson-07.local.
                             TXT  "tier=jetson" "cuda=85" "ram_free_gb=6.2"
```

The ESP32 selects the broker with the lowest SRV priority, falls back to the highest weight on ties, and opens an MQTT connection on port 1883. It immediately publishes a retained `birth` message:

```json
{
  "fleet_id": "a1b2c3d4...",
  "mote_id": "esp32-garage-03",
  "caps": ["pir", "magnetic", "dht22", "relay"],
  "dial": 0.0,
  "last_seen": 1716892800
}
```

The topic hierarchy is:

- `fleet/<fleet_id>/birth/<mote_id>` — retained LWT-compatible birth certificate.
- `fleet/<fleet_id>/tick/<mote_id>` — ephemeral snaps and inferences.
- `fleet/<fleet_id>/act/<mote_id>/<exit>` — actuator commands from the hub.

### 1.2 Heavy-Node Discovery — gRPC + mDNS-SD

Jetsons, desktops, and DGX nodes discover each other over gRPC with mDNS-SD service type `_construct._tcp.local`. Advertisements carry structuredmetadata:

```protobuf
message NodeAdvert {
  string node_id     = 1;   // "jetson-07.local"
  Tier   tier        = 2;   // JETSON, DESKTOP, DGX
  repeated string roles = 3; // ["vision_hub", "mqtt_bridge"]
  uint32 cuda_sm     = 4;  // streaming-multiprocessor count
  float  load_1m     = 5;   // CPU load
  bytes  tls_cert_hash = 6; // SHA-256 of the node's mTLS cert
}
```

Heavy nodes maintain a gossip sub-protocol over a bidirectional gRPC stream (`/construct.v1.Fleet/Gossip`). Every 5 seconds each node pushes a `NodeHeartbeat` containing its current load, the set of mote IDs it bridges, and the dial position it recommends for its leaf cluster. The DGX master aggregates these into a `FleetView`protobuf that is flushed to SQLite-backed gateway state every 30 seconds.

---

## 2. Hierarchical Mesh

### 2.1 Jetson as Hub

A Jetson is the default gateway for every ESP32 within Wi-Fi range. It runs three local services:

- **MQTT broker** (mosquitto or nanomq) bound to `0.0.0.0:1883`.
- **gRPC bridge**(`openshell-fleet-bridge`) that subscribes to `fleet/+/tick/+` and republishes snaps into local signal-chain rooms.
- **Vision inference worker** (TensorRT or ONNX Runtime) that services `inference.local` requests from the bridge.

Each ESP32 is represented inside the Jetson supervisor as a **Plato room** named after its `mote_id`. The room's dial is clamped to `DIAL_BATHY` (0.10) because sensor data is treated as hard fact unless explicitly softened by a downstream inference worker.

### 2.2 Desktop Peer Ring

Desktops do not bridge MQTT. They join a gossip peer ring with other desktops and visible Jetsons. A desktop may:

- Pull `FleetView` snapshots from the DGX to render a TUI of the entire installation.
- Push policy bundles (YAML) to the DGX, which fans them out to relevant Jetson supervisors.
- Request relay tunnels into a specific Jetson sandbox for live debugging.

Desktop-to-desktop traffic uses gRPC with mTLS backed by the same fleet CA that signs ESP32 PSKs.

### 2.3 DGX as Orchestrator

The cloud/DGX tier hosts the authoritative gateway and the training pipeline. It maintains:

- The canonical `FleetView` database.
- Global policy revision history.
- A batched inference queue for jobs too large for edge GPUs.
- Model retraining jobs that produce updated TensorRT engines for the Jetsons.

DGX nodes arethe only tier that may open connections *into* Jetsons (via supervisor relay forwarding). All other traffic is supervisor-initiated outbound, preserving the OpenShell security model.

---

## 3. Resource Delegation

### 3.1 Vision Delegation — Jetson to ESP32 Cluster

An ESP32 has no camera and no FPU. When a PIR snap fires, the Jetson agent *visits* the ESP32 room, reads the snap, and decides whether to run vision inference on a locally attached USB camera that covers the same physical zone:

```rust
// Inside the Jetson supervisor sandbox
let garage = chain.room("esp32-garage-03");
garage.add_snap(json!({"pir": true, "timestamp_ms": 1716892800123}), 1.0);

// Query at DIAL_ANALYSIS (0.40) to include low-confidence hypotheses
let context = garage.query(Dial::analysis());
if context.iter().any(|r| r.has_snap("pir")) {
    // Delegate to local GPU
   let inference = vision_model.infer(camera_frame).await?;
    garage.add_inference(
        json!({"visitor_detected": inference.label, "confidence": inference.confidence}),
        inference.confidence,
    );
}
```

The inference is stored as a *soft* signal in the ESP32 room, not pushed back to the mote. Themote only receives actuator commands.

### 3.2 Batch Delegation — DGX to Fleet

When a desktop requests a fleet-wide anomaly model, the DGX does not stream individual camera frames. Instead it:

1. Queries all Jetsons for their last 10 000 snapped inferences.
2. Runs batch embedding extraction on A100/H100 GPUs.
3. Returns a compressed TensorRT engine to each Jetson via the existing supervisor-initiated relay.

This respects the OpenShell boundary: the DGX never initiates raw TCP into a Jetson home network. The Jetson supervisor polls the DGX for available engine artifacts and applies them locally.

---

## 4. Failover

### 4.1 Jetson Death

If a Jetson stops heartbeating for >30 seconds, the DGX marks it `DOWN` in `FleetView` and triggers a failover sequence:

1. **MQTT broker migration.** The ESP32s connected to the dead Jetson have a last-will Testament (LWT) published by the broker on their behalf: `{"state":"orphan","last_broker":"jetson-07"}`. Surviving Jetsons subscribe to `fleet/+/birth/+/state` and race to adopt orphans.
2. **Adoption election.** The surviving Jetson with the lowest load_1m and highest RAM wins. It sends an MQTT `ADOPT` control packet to the orphan mote's `act` topic.
3. **Mote reconnect.** The ESP32, upon receiving `ADOPT`, disconnects from its stale broker socket and reconnects to the newone using the same PSK and `fleet_id`.
4. **Room hand-off.** The adopting Jetson imports the orphan's last retained `birth` certificate, reconstructs the Plato room from persisted SQLite state in the DGX (synced every 30 s), and resumes inference duties.

The entire sequence completes in <2 seconds for a cluster of50 ESP32s.

### 4.2 DGX Partition

If the cloud link drops, Jetsons fall back to a **local quorum** of known desktops. They cache the last policy bundle andcontinue operating in *autonomous mode* with a capped dial of `DIAL_ANALYSIS` (0.40) to prevent ungrounded soft inferences from triggering physical actuators without oversight.

---

## 5. Room Metaphor

In OpenConstruct, every ESP32 is a **room** that the Jetson agent can visit. The room contains:

- **Sensor objects** (`Snap` instances) — hard telemetry: temperature, magnetic reed switchstate, LUX readings.
- **Actuator exits** — named output channels mapped to MQTT topics under `fleet/<fleet_id>/act/<mote_id>/<exit>`.

A concrete room layout for `esp32-front-door-01`:

```yaml
room: esp32-front-door-01
dial: 0.10  # DIAL_BATHY — sensor facts only
objects:
  - name: magnetic_reed
    type: Snap
    value: { closed: false }
    confidence: 1.0
  - name: pir_motion
   type: Snap
    value: { motion: true, zone: "porch" }
    confidence: 1.0
  - name: ambient_light
    type: Snap
    value: { lux: 12.4 }
    confidence:1.0
exits:
  - name: door_lock
    topic: act/esp32-front-door-01/door_lock
   payload_schema: { lock: bool }
  - name: porch_light
    topic: act/esp32-front-door-01/porch_light
    payload_schema: { intensity: uint8 }
```

When the Jetson agent *enters* the room (i.e., the local bridge worker processes a new MQTT tick), it sees the latest snaps arranged on the walls. It may leave an `Inference` (a hypothesis note) on the room's cork-board, but it cannot alter a Snap. Only the ESP32 itself can update a Snap by publishing a new tick.

---

## 6. Tick Propagation

Ticks are the atomic unit of fleet cognition. A tick is a signed JSON blob with a monotonic sequence number, a timestamp, and a payload. Ticks flow upward through the mesh, growing in semantic richness at each tier.

### 6.1 Concrete Data Flow — Front-Door Intrusion Scenario

**T0 — ESP32leaves a tick.**

The front-door mote detects motion and publishes:

```json
{
  "tick_id": "esp32-front-door-01:7842",
  "seq": 7842,
  "ts_ms": 1716892800123,
  "dial": 0.0,
  "snaps": [
    { "key": "pir_motion", "value": { "motion": true, "zone": "porch" }, "conf": 1.0 },
    { "key": "magnetic_reed", "value": { "closed": false }, "conf": 1.0 }
  ]
}
```

Topic: `fleet/a1b2c3d4/tick/esp32-front-door-01`

**T1 — Jetson reads the tick.**

The Jetson bridge worker routes the tick into the local signal-chain room `esp32-front-door-01`. The room now contains two new snaps. The Jetson agent, running inside its sandbox with `Dial::analysis()`, queries the room and sees a potential security event (door open + motion). It captures a camera frame and runs TensorRT inference:

```rust
let room = chain.room("esp32-front-door-01");
room.add_snap(tick.into_snap("pir_motion"), 1.0);
room.add_snap(tick.into_snap("magnetic_reed"), 1.0);

let visitor = vision.infer(frame).await?;
room.add_inference(
    json!({"visitor": visitor.label, "bounding_box": visitor.bbox}),
    visitor.confidence,
);
```

The Jetson now leaves its own tick for upstream nodes:

```json
{
  "tick_id": "jetson-07:4911",
  "parent": "esp32-front-door-01:7842",
 "seq": 4911,
  "ts_ms": 1716892800456,
  "dial": 0.40,
  "snaps": [
    { "key": "pir_motion", "value": { "motion": true, "zone": "porch" }, "conf": 1.0 },
    { "key": "magnetic_reed", "value": { "closed": false }, "conf": 1.0 }
  ],
  "inferences": [
    { "key": "visitor", "value": { "label": "human", "bbox": [120,80,340,410] }, "conf": 0.91 }
  ]
}
```

**T2 — Desktop reads the Jetson tick.**

The desktop pulls aggregated ticks from the Jetson via gRPC (`/construct.v1.Fleet/PollTicks`). It sees the `visitor` inference with confidence 0.91. The desktop agent, operating at `Dial::review()` (0.50), decides this crosses the action threshold. It publishes a command tick back down the chain:

```json
{
  "tick_id": "desktop-alice:332",
  "parent": "jetson-07:4911",
  "seq": 332,
  "ts_ms": 1716892800890,
  "dial": 0.50,
  "act": [
    { "exit":"door_lock", "payload": { "lock": true } },
    { "exit": "porch_light", "payload": { "intensity": 255 } }
  ]
}
```

**T3 — Jetson relays to ESP32.**

The Jetson strips the `act` array and forwards each command to the appropriate MQTT topic:

- `fleet/a1b2c3d4/act/esp32-front-door-01/door_lock` → `{ "lock": true }`
- `fleet/a1b2c3d4/act/esp32-front-door-01/porch_light` → `{ "intensity": 255 }`

The ESP32 applies the commands, updates its local actuator state, and publishes confirmation snaps in the next tick.

### 6.2 Cascade Rules

When a Jetson produces a high-confidence inference (≥ 0.85) in an ESP32 room, the signal-chain `cascade` primitive promotes it downward into *sibling* rooms that share the same physical zone. For example, a `visitor_detected` inference in `esp32-front-door-01` cascades as a snap into `esp32-porch-02` at 0.68 confidence (0.85 × 0.8 decay). This lets nearby motes coordinate without waiting for the desktop tier.

---

## 7. Security Boundaries

The mesh inherits OpenShell's supervisor-gateway split:

- **ESP32 → Jetson:** MQTT over TLS-PSK. The PSK isfleet-scoped; compromise of one mote does not grant access to others because theJetson broker enforces topic ACLs by `mote_id`.
- **Jetson → Desktop/DGX:** mTLS with SPIFFE workload identity. The supervisor initiates outbound gRPC and keeps the connection alive; no inbound firewall holes required.
- **Actuator commands:** Every `act` tick is signed by the desktop agent's private key. The Jetsonverifies the signature against the fleet CA before relaying to MQTT. Unsigned actuator messages are dropped by the bridge.

---

## 8. Operational Invariants

1. **Snaps only flow up.** An ESP32 snap may be read by a Jetson, but a Jetson may never write a snap into an ESP32 room. The ESP32 owns its own facts.
2.**Inferences only flow down.** A desktop inference may cascade into a Jetson room, but it may never overwrite an ESP32 snap. Inferences are advice, not truth.
3. **Actuators only fire on signed exits.** Unsigned or replayed actuator commands are rejected by the bridge's policy proxy.
4. **Dial hardens under partition.** When a Jetson loses DGX contact, its local dial ceiling drops to 0.40, preventing soft hallucinations from controlling physical hardware without fleet oversight.

These invariants keep the mesh safe even when nodes fail, partitions heal slowly, or models hallucinate at the edge.
```')StatusUpdate(    context_usage=0.10066986083984375,    context_tokens=26390,    max_context_tokens=262144,    token_usage=TokenUsage(        input_other=2582,        output=4063,        input_cache_read=23808,        input_cache_creation=0    ),    message_id='chatcmpl-KOupSDjLsYgr6QeDCtlVRAnD',    plan_mode=False,    mcp_status=None)TurnEnd()To resume this session: kimi -r 291b7035-8f4b-4bb6-a7e1-fec64cd99868