# Fleet Deployment Guide

> Deploy the SuperInstance ecosystem from a single room to 500 rooms — ESP32s, Jetsons, desktops, and DGX Sparks, all speaking one protocol.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Hardware Tiers](#2-hardware-tiers)
3. [Installing OpenConstruct](#3-installing-openconstruct)
4. [Room Configuration](#4-room-configuration)
5. [PLATO Nervous System Deployment](#5-plato-nervous-system-deployment)
6. [Fleet Mesh Networking](#6-fleet-mesh-networking)
7. [Scaling: 1 Room to 500 Rooms](#7-scaling-1-room-to-500-rooms)
8. [Monitoring & Dashboards](#8-monitoring--dashboards)
9. [Walkthrough: Fishing Vessel Demo](#9-walkthrough-fishing-vessel-demo)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Architecture Overview

The SuperInstance fleet is a **hierarchical mesh** — rigid at the edges, peer-to-peer at the core. ESP32 sensor motes report to Jetson edge hubs. Jetsons coordinate with desktop workstations. One or more DGX Spark nodes serve as fleet coordinators, running heavy inference, training, and cloud relay.

```
                         ┌─────────────────────┐
                         │     DGX Spark        │
                         │   Fleet Coordinator  │
                         │                      │
                         │  L0–L4 + Cloud Relay │
                         │  Training / A2A Hub  │
                         └──────────┬───────────┘
                                    │ gRPC + WebSocket
                    ┌───────────────┼───────────────┐
                    │               │               │
            ┌───────▼──────┐ ┌─────▼──────┐ ┌──────▼───────┐
            │   Desktop    │ │  Desktop   │ │   Desktop    │
            │  Multi-Room  │ │ Multi-Room │ │  Multi-Room  │
            │  L0 + L1 + L2│ │L0 + L1 + L2│ │ L0 + L1 + L2 │
            └───────┬──────┘ └─────┬──────┘ └──────┬───────┘
                    │               │               │
          ┌─────────┼───────┬───────┼───────┬───────┼────────┐
          │         │       │       │       │       │        │
    ┌─────▼───┐┌────▼──┐┌──▼────┐┌──▼────┐┌─▼─────┐┌─▼──────┐
    │ Jetson  ││Jetson ││Jetson ││Jetson ││Jetson ││ Jetson │
    │Room Hub ││Room   ││Room   ││Room   ││Room   ││ Room   │
    │L0 + L1  ││L0 + L1││L0 + L1││L0 + L1││L0 + L1││L0 + L1 │
    └────┬────┘└───┬───┘└───┬───┘└───┬───┘└───┬───┘└───┬────┘
         │         │         │         │         │        │
    ┌────┼────┐ ┌──┼──┐ ┌───┼───┐ ┌──┼──┐ ┌───┼──┐ ┌──┼───┐
    │    │    │ │  │  │ │   │   │ │  │  │ │   │  │ │  │   │
   🔧   📷  🎤  🔧 📷 🎤 🔧 📷 🎤  🔧 📷 🎤 🔧 📷 🎤  🔧 📷 🎤
  ESP  ESP ESP ESP ESP ESP ESP ESP ESP ESP ESP ESP ESP ESP ESP
  L0   L0  L0  L0  L0  L0  L0  L0  L0  L0  L0  L0  L0  L0
```

**Key invariant:** ESP32s never speak gRPC. DGX nodes never poll MQTT topics one-by-one. The Jetson sits at the boundary, translating between the two worlds.

---

## 2. Hardware Tiers

### Specs Table

| Property | ESP32 (Sensor Mote) | Jetson Orin Nano (Room Controller) | Desktop RTX 4070+ (Multi-Room) | DGX Spark (Fleet Coordinator) |
|---|---|---|---|---|
| **CPU** | 240 MHz Xtensa dual-core | 6-core ARM A78AE | 8+ core x86_64 | Grace CPU (72 Arm cores) |
| **GPU** | None | 1024 CUDA cores (Ampere) | RTX 4070+ (5888+ CUDA) | Blackwell GPU (CUDA cluster) |
| **RAM** | 520 KB SRAM / 4–16 MB flash | 8 GB LPDDR5 | 32–64 GB DDR5 | 384 GB LPDDR5x |
| **Storage** | MicroSD / SPI flash | 64 GB NVMe | 1 TB NVMe SSD | 4 TB NVMe RAID |
| **Network** | Wi-Fi 4, BLE 5.0 | Gigabit Ethernet + Wi-Fi 6 | Gigabit Ethernet | 2× 100GbE + Ethernet |
| **Power** | 0.5–2 W | 7–15 W | 200–400 W | ~1200 W |
| **Cost (approx)** | $3–10 | $250–500 | $1,500–3,000 | $30,000+ |
| **PLATO Layers** | L0 deadband only | L0 + L1 (nano model) | L0 + L1 + L2 (LoRA) | All layers (L0–L4) |
| **Role** | Single-sensor room | Room controller, MQTT→gRPC bridge | Multi-room coordination, LoRA fine-tuning | Fleet orchestration, training, cloud relay |
| **Protocol** | MQTT | MQTT + gRPC | gRPC + WebSocket | gRPC + WebSocket + A2A |
| **Deploy Target** | Any room with power + Wi-Fi | 1 per room cluster (8–16 ESP32s) | 1 per zone (4–8 Jetsons) | 1 per fleet (or per building) |

### Tier Selection Decision Tree

```
Do you need GPU inference?
├── No  → ESP32 (sensor/actuator leaf)
└── Yes
    ├── Do you need to serve 8+ rooms?
    │   ├── No  → Jetson (single-room or small cluster)
    │   └── Yes
    │       ├── Do you need model training?
    │       │   ├── No  → Desktop (multi-room, LoRA fine-tuning)
    │       │   └── Yes → DGX Spark (fleet coordinator)
    │       └── DGX Spark if budget allows
    └── Jetson minimum for any GPU need
```

---

## 3. Installing OpenConstruct

### Universal Installer

Every tier uses the same bootstrap command. The installer detects hardware capabilities and configures automatically:

```bash
curl -fsSL https://openconstruct.sh | bash
```

The installer:
1. Detects CPU architecture (ARM, x86_64, Xtensa)
2. Identifies GPU availability (CUDA, none)
3. Measures available RAM and storage
4. Selects the appropriate package set for the detected tier
5. Generates a default configuration
6. Registers with the fleet (if `FLEET_ID` is set)

### Per-Tier Installation

#### ESP32 — Cross-Compiled Flash

The ESP32 can't run the installer directly. Flash from a host machine:

```bash
# On your desktop/laptop
curl -fsSL https://openconstruct.sh | bash -s -- --target=esp32s3

cd openconstruct-esp32
idf.py set-target esp32s3
idf.py menuconfig    # Set WiFi SSID, password, FLEET_ID
idf.py build
idf.py flash -p /dev/ttyUSB0
idf.py monitor       # Verify boot and MQTT connection
```

#### Jetson Orin — Native Install

```bash
# SSH into the Jetson
ssh user@jetson-07.local

# Install (detects ARM + CUDA automatically)
curl -fsSL https://openconstruct.sh | bash

# Verify GPU detection
openconstruct doctor
# ✓ CUDA 12.2 detected (1024 cores)
# ✓ MQTT broker reachable at _mqtt._tcp.local
# ✓ PLATO L0 + L1 layers available

# Start as room controller
openconstruct start --tier=jetson
```

#### Desktop — Full Agent Runtime

```bash
# Local desktop with RTX GPU
curl -fsSL https://openconstruct.sh | bash

# Verify full stack
openconstruct doctor
# ✓ CUDA 12.4 detected (RTX 4070, 5888 cores)
# ✓ PLATO L0 + L1 + L2 layers available (LoRA adapter loaded)
# ✓ Fleet mesh connectivity OK

# Start as multi-room coordinator
openconstruct start --tier=desktop
```

#### DGX Spark — Fleet Coordinator

```bash
# On the DGX Spark
curl -fsSL https://openconstruct.sh | bash

# Verify all layers including L4 cloud relay
openconstruct doctor
# ✓ Grace CPU + Blackwell GPU detected
# ✓ PLATO L0–L4 all layers available
# ✓ Cloud relay endpoint configured
# ✓ A2A protocol handler ready

# Start as fleet coordinator
openconstruct start --tier=dgx-spark --coordinator
```

### Environment Variables

| Variable | Required | Description |
|---|---|---|
| `FLEET_ID` | Yes | 128-bit UUID shared by the entire fleet |
| `OC_TIER` | Auto | Override auto-detected tier (`esp32`, `jetson`, `desktop`, `dgx`) |
| `OC_MQTT_BROKER` | Auto | MQTT broker address (auto-discovered via mDNS) |
| `OC_GRPC_ENDPOINT` | No | Upstream gRPC endpoint for heavy nodes |
| `OC_COORDINATOR` | No | Set `true` on DGX Spark to enable coordinator mode |

---

## 4. Room Configuration

Rooms are defined in YAML — zero-shot understandable by both humans and agents. A room is the fundamental unit of the Plato sensory architecture: a named sense endpoint with defined inputs and outputs.

### Room Definition Schema

```yaml
# rooms/galley.yaml
room:
  id: fishing-vessel-galley
  name: "Galley"
  description: "Ship's kitchen and mess area, port side amidships"
  tier: jetson          # Which tier manages this room
  controller: jetson-galley-01

  senses:
    - id: galley-cam-01
      type: vision
      hardware: esp32s3-cam
      description: "Overhead camera covering stove and prep area"
      position: [2.1, 0.5, 3.4]  # x, y, z meters from room origin
      capabilities: [snapshot, stream]
      default_level: L2

    - id: galley-mic-01
      type: audio
      hardware: inmp441
      description: "Ceiling microphone, captures cooking sounds and speech"
      position: [2.0, 0.5, 2.0]
      capabilities: [listen, classify]
      default_level: L2

    - id: galley-temp-01
      type: environment
      hardware: dht22
      description: "Temperature and humidity sensor near stove"
      position: [1.5, 1.2, 3.0]
      capabilities: [read]
      default_level: L1
      poll_interval: 30s

    - id: galley-smoke-01
      type: environment
      hardware: mq2
      description: "Smoke and gas detector above stove"
      position: [1.5, 0.3, 3.0]
      capabilities: [read, alert]
      default_level: L1
      alert_threshold: 200ppm

  actuators:
    - id: galley-stove-relay
      type: relay
      hardware: esp32s3
      description: "Emergency stove power cutoff"
      position: [1.0, 1.0, 3.0]
      capabilities: [on, off]
      safety: critical      # Requires confirmation before actuation

    - id: galley-vent-fan
      type: relay
      hardware: esp32s3
      description: "Ventilation fan control"
      capabilities: [on, off, speed]
      safety: normal

  policy:
    max_actuation_level: L3          # Actuation requires L3+ agent authorization
    privacy_zones:                    # Regions to mask in vision output
      - name: "Crew personal locker"
        bounds: [0.0, 0.0, 0.0, 1.0, 2.0, 1.0]
    alert_escalation:
      smoke: [local_alarm, coordinator, captain_display]
      motion_off_hours: [coordinator]
```

### Multi-Room Fleet Configuration

```yaml
# fleet.yaml
fleet:
  id: fishing-vessel-alpha
  name: "F/V Aurora Fleet Intelligence"
  coordinator: dgx-spark-01

  rooms:
    - include: rooms/galley.yaml
    - include: rooms/bridge.yaml
    - include: rooms/engine-room.yaml
    - include: rooms/fish-hold.yaml
    - include: rooms/deck.yaml

  mesh:
    discovery: mdns
    mqtt_broker: jetson-hub-01.local
    grpc_backbone: dgx-spark-01.local:50051
    websocket_relay: dgx-spark-01.local:8080

  scheduling:
    timezone: UTC-9  # Alaska
    watch_schedule: true
    autonomy_limits:
      default: advisory      # Agent suggests, human decides
      engine_room: supervised # Agent acts, human watches
      bridge: advisory        # Agent never acts autonomously on bridge

  monitoring:
    health_interval: 60s
    metrics_retention: 30d
    dashboard: true
```

### Configuration Hierarchy

```
fleet.yaml                    # Global fleet settings
├── rooms/
│   ├── galley.yaml           # Room-level config
│   ├── bridge.yaml
│   └── ...
├── policies/
│   ├── safety.yaml           # Global safety policies
│   └── privacy.yaml          # Global privacy policies
└── models/
    ├── l1-nano.yaml          # L1 model configuration
    ├── l2-lora.yaml          # L2 LoRA adapter config
    └── l4-cloud.yaml         # Cloud relay endpoints
```

---

## 5. PLATO Nervous System Deployment

The PLATO Nervous System is the signal chain that transforms raw sensor data into agent-readable text at increasing levels of abstraction. Each tier runs a subset of the signal chain appropriate to its compute budget.

### Signal Chain Layers

```
  Layer    Name            Description                           Runs On
  ──────── ─────────────── ────────────────────────────────────── ──────────────────
  L0       Deadband        Algorithmic filtering (no ML model)   ESP32, Jetson, Desktop, DGX
  L1       Summary         Nano model inference (≤10M params)     Jetson, Desktop, DGX
  L2       Description     LoRA fine-tuning (≤1B params)         Desktop, DGX
  L3       Catalog         Full model inference                   DGX
  L4       Annotation      Cloud relay / remote inference         DGX (relay)
```

### Per-Tier Deployment Map

#### ESP32 — L0 Deadband Only

The ESP32 runs no model. L0 is a pure algorithmic layer that filters noise and detects state changes:

```c
// Deadband filter — runs on ESP32, no ML required
// Only sends a tick when value changes beyond threshold
bool deadband_filter(float current, float last, float threshold) {
    return fabsf(current - last) > threshold;
}
```

- **Memory footprint:** ~2 KB RAM
- **CPU usage:** < 1% of one core
- **What it does:** Noise gating, change detection, heartbeat emission
- **Output:** Raw ticks sent via MQTT only when something changes

#### Jetson — L0 + L1 (Nano Model with GPU Offload)

The Jetson adds L1 inference using a quantized nano model:

```yaml
# l1-nano.yaml
model:
  name: plato-vision-nano-q8
  parameters: 8M
  quantization: int8
  gpu_layers: 12      # Offload all layers to Jetson GPU
  input: l0_ticks     # Consumes L0 output
  output: l1_summary  # Produces one-line summaries
  latency_target: 100ms
  batch_size: 4       # Process up to 4 rooms per inference pass
```

- **Memory footprint:** ~50 MB VRAM
- **GPU usage:** ~15% of Orin Nano GPU
- **What it does:** Scene summarization, sound classification, anomaly detection
- **Output:** One-line summaries like `"Galley: stove active, one person cooking, 72°F, smoke level normal"`

#### Desktop — L0 + L1 + L2 (LoRA Fine-Tuning)

The Desktop adds L2 with a LoRA adapter for richer descriptions:

```yaml
# l2-lora.yaml
model:
  base: plato-vision-base-1b
  lora_adapter: plato-nautical-lora-v3
  parameters: 1B (base) + 8M (LoRA)
  quantization: q4_k_m
  gpu_memory: 6GB      # Fits on RTX 4070
  input: l1_summaries
  output: l2_descriptions
  latency_target: 500ms
  fine_tune:
    dataset: vessel-observations-v5
    method: lora
    rank: 16
    alpha: 32
    schedule: nightly   # Fine-tune during off-hours
```

- **Memory footprint:** ~4 GB VRAM
- **GPU usage:** ~40% of RTX 4070
- **What it does:** MUD-style room descriptions, spatial relationships, temporal patterns
- **Output:** Paragraph descriptions with key details, contextual awareness of vessel operations
- **Fine-tuning:** LoRA adapters are retrained nightly on new observations to adapt to changing conditions

#### DGX Spark — All Layers Including L4 Cloud Relay

The DGX Spark runs the full signal chain and serves as the fleet's brain:

```yaml
# dgx-full-stack.yaml
layers:
  l0:
    enabled: true
    role: fleet_aggregator    # Aggregates L0 ticks from all nodes

  l1:
    enabled: true
    role: batch_inference     # Can batch-process L1 for 100+ rooms

  l2:
    enabled: true
    role: quality_gate        # Cross-validates desktop L2 outputs

  l3:
    enabled: true
    model: plato-vision-full-7b
    description: "Full catalog with structured inventory"

  l4:
    enabled: true
    cloud_relay:
      endpoint: https://api.openconstruct.cloud/v1/relay
      fallback: true          # Falls back to local L3 if cloud unreachable
      compression: gzip
      batch_interval: 5s
      privacy_filter: strip_pii  # Remove faces, text, identifying data
```

- **Memory footprint:** ~40 GB VRAM for full stack
- **GPU usage:** ~60% of Blackwell GPU
- **Additional roles:** Model training, A2A protocol hub, fleet state reconciliation, WebSocket relay

### Signal Flow Diagram

```
ESP32 Sensor ──L0 tick──► Jetson ──L1 summary──► Desktop ──L2 description──► DGX Spark
  (deadband      │           (nano            │           (LoRA                 │
   filter)       │            model)           │            fine-tune)          │
                │                            │                             │
                │          ┌─────────────────┘                             │
                │          │ L1 batch for rooms                             │
                │          │ without desktop                                │
                │          ▼                                                ▼
                │    DGX Spark (fallback)                          DGX Spark
                │    L1 direct from ESP32                           L3 + L4
                │                                                     │
                └─────────────────────────────────────────────────────┘
                                      A2A Protocol
                                      (agent-to-agent
                                       deliberation)
```

---

## 6. Fleet Mesh Networking

### Protocol Stack

```
  Application     A2A Protocol (agent deliberation)
                  WebSocket Relay (dashboard, remote control)
  ─────────────────────────────────────────────────────────
  Transport       gRPC (heavy nodes: Jetson ↔ Desktop ↔ DGX)
                  MQTT (ESP32 ↔ Jetson, lightweight pub/sub)
  ─────────────────────────────────────────────────────────
  Discovery       mDNS-SD (_mqtt._tcp.local, _construct._tcp.local)
  ─────────────────────────────────────────────────────────
  Physical        Ethernet backbone, Wi-Fi for ESP32s
```

### mDNS Discovery

#### ESP32 Discovery (mDNS + MQTT)

1. ESP32 boots with `fleet_id`, PSK, and mDNS query target
2. Mote broadcasts mDNS query for `_mqtt._tcp.local`
3. Jetsons/desktops respond with SRV priority based on load:

```
jetson-galley-01._mqtt._tcp.local.  PTR jetson-galley-01.local.
jetson-galley-01.local.             A    192.168.4.17
                                    SRV  0 20 1883 jetson-galley-01.local.
                                    TXT "tier=jetson" "cuda=85" "ram_free_gb=6.2"
```

4. ESP32 connects to best broker, publishes retained `birth` message
5. Heartbeat every 30 seconds; death declared after 90 seconds of silence

#### Heavy-Node Discovery (gRPC + mDNS-SD)

Jetsons, desktops, and DGX nodes discover each other via `_construct._tcp.local`:

```
dgx-spark-01._construct._tcp.local.  PTR dgx-spark-01.local.
dgx-spark-01.local.                  A    192.168.1.1
                                      SRV  0 10 50051 dgx-spark-01.local.
                                      TXT "tier=dgx" "rooms=42" "models=l0-l4"
```

### WebSocket Relay

The DGX Spark runs a WebSocket relay for dashboard connections and remote agents:

```
Client (dashboard)                  DGX Spark (relay)
     │                                   │
     │── WS CONNECT /ws/fleet ──────────►│
     │◄── WS SUBSCRIBE rooms:* ─────────│
     │                                   │
     │◄── WS EVENT room:galley:tick ────│  (forwarded from MQTT→gRPC)
     │◄── WS EVENT room:bridge:alert ───│
     │── WS COMMAND room:galley:fan:on ─►│  (relayed to gRPC→MQTT)
     │                                   │
```

### A2A Protocol

Agent-to-Agent protocol enables deliberation between rooms:

```json
{
  "protocol": "a2a/v1",
  "from": "room:galley:agent",
  "to": "room:bridge:agent",
  "type": "deliberation",
  "payload": {
    "topic": "ventilation_adjustment",
    "proposal": "Increase galley vent fan to 80% — smoke detector reading 180ppm",
    "confidence": 0.87,
    "evidence": ["galley-smoke-01: 180ppm", "galley-temp-01: 78°F"]
  }
}
```

### Network Architecture Diagram

```
                    ┌─────────────────────────────────┐
                    │         Internet / Cloud         │
                    │      (L4 relay endpoint)         │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │          DGX Spark               │
                    │   WebSocket Relay :8080          │
                    │   gRPC Server     :50051         │
                    │   A2A Hub         :50052         │
                    └──────────────┬──────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │ Ethernet Backbone  │                    │
     ┌────────▼────────┐  ┌───────▼────────┐  ┌───────▼────────┐
     │    Desktop-01   │  │   Desktop-02   │  │   Desktop-03   │
     │  gRPC :50051    │  │  gRPC :50051   │  │  gRPC :50051   │
     └────────┬────────┘  └───────┬────────┘  └───────┬────────┘
              │                   │                   │
     ┌────────▼────────┐         │          ┌────────▼────────┐
     │   Jetson-01     │  ┌──────▼──────┐   │   Jetson-02     │
     │ MQTT :1883      │  │  Jetson-03  │   │ MQTT :1883      │
     │ gRPC :50051     │  │ MQTT :1883  │   │ gRPC :50051     │
     └────────┬────────┘  └──────┬──────┘   └────────┬────────┘
              │ Wi-Fi            │ Wi-Fi             │ Wi-Fi
     ┌────────┼──────┐   ┌──────┼──────┐   ┌────────┼──────┐
     │        │      │   │      │      │   │        │      │
   🔧ESP   📷ESP  🎤ESP  🔧ESP 📷ESP 🎤ESP 🔧ESP  📷ESP  🎤ESP
```

---

## 7. Scaling: 1 Room to 500 Rooms

### Scaling Tiers

| Rooms | Topology | Hardware | Notes |
|---|---|---|---|
| **1** | ESP32 + Jetson | 1 Jetson Orin Nano, 1–3 ESP32s | Simplest fleet. Jetson runs everything. |
| **5–10** | Jetson cluster + Desktop | 1 Desktop, 2–5 Jetsons, 10–30 ESP32s | Desktop handles L2 and cross-room coordination. |
| **10–50** | Desktop + DGX or multi-Desktop | 1 DGX Spark (or 2+ Desktops), 10+ Jetsons, 50–150 ESP32s | Full mesh. gRPC backbone. WebSocket dashboards. |
| **50–200** | DGX Spark fleet | 1–2 DGX Sparks, 20+ Jetsons, 200–600 ESP32s | A2A deliberation at scale. Batch inference on DGX. |
| **200–500** | Multi-DGX with zone segmentation | 2–4 DGX Sparks, 40+ Jetsons, 500–1500 ESP32s | Zone-based topology. Hierarchical aggregation. |

### Scaling Strategy

```
Phase 1: Single Room (Day 1)
├── 1 Jetson Orin Nano
├── 2–3 ESP32s (camera, mic, temp)
├── L0 + L1 signal chain
└── MQTT-only networking

Phase 2: Small Cluster (Week 1)
├── Add 1 Desktop with RTX GPU
├── Expand to 5 rooms, 15 ESP32s
├── L0 + L1 + L2 signal chain
├── gRPC backbone added
└── First dashboard deployment

Phase 3: Building Scale (Month 1)
├── Add DGX Spark as coordinator
├── 50 rooms, 150 ESP32s, 10 Jetsons
├── Full L0–L4 signal chain
├── A2A protocol enabled
├── WebSocket relay for dashboards
└── Nightly LoRA fine-tuning

Phase 4: Campus Scale (Month 3+)
├── Zone segmentation with multiple DGX Sparks
├── 200–500 rooms
├── Hierarchical tick aggregation
├── Federated A2A deliberation
└── Continuous model retraining
```

### Performance Budget at Scale

| Metric | 50 Rooms | 200 Rooms | 500 Rooms |
|---|---|---|---|
| **MQTT messages/sec** | ~150 | ~600 | ~1,500 |
| **gRPC calls/sec** | ~50 | ~200 | ~500 |
| **L1 inferences/sec (Jetson)** | ~50 | 200 (5 Jetsons) | 500 (10 Jetsons) |
| **L2 inferences/sec (Desktop)** | ~20 | 80 (4 Desktops) | 200 (8 Desktops) |
| **L3 batch inference (DGX)** | ~10 | ~40 | ~100 |
| **Network bandwidth** | ~5 Mbps | ~20 Mbps | ~50 Mbps |
| **DGX GPU utilization** | ~10% | ~30% | ~60% |

---

## 8. Monitoring & Dashboards

### Room Health Dashboard

Each room reports health metrics on a 60-second interval:

```yaml
# Health tick format
room_health:
  room_id: fishing-vessel-galley
  timestamp: 1716892800
  status: healthy           # healthy | degraded | critical | offline
  uptime: 86400
  
  sensors:
    - id: galley-cam-01
      status: online
      last_reading: 1716892799
      latency_ms: 45
    - id: galley-temp-01
      status: online
      last_reading: 1716892795
      value: 72.3°F
  
  actuators:
    - id: galley-vent-fan
      status: online
      last_command: "speed:60%"
      command_age: 120s
  
  signal_chain:
    l0_latency: 12ms
    l1_latency: 85ms
    l2_latency: 420ms
  
  autonomy:
    mode: advisory
    decisions_today: 14
    human_overrides: 2
    autonomy_percentage: 85.7%
```

### Fleet-Level Metrics

```bash
# Fleet status overview
openconstruct fleet status

# Output:
# Fleet: fishing-vessel-alpha
# Coordinator: dgx-spark-01
# 
# Rooms:     12/12 online (100%)
# Sensors:   34/36 online  (94%)  ← 2 ESP32s need attention
# Actuation: All systems nominal
# 
# Signal Chain:
#   L0 (deadband):    12/12 rooms  avg latency: 8ms
#   L1 (nano model):  12/12 rooms  avg latency: 78ms
#   L2 (LoRA):         8/12 rooms  avg latency: 390ms
#   L3 (full model):   2/12 rooms  avg latency: 1.2s
#   L4 (cloud relay):  0/12 rooms  (standby)
# 
# Autonomy: 87.3% (263 decisions, 34 human overrides)
# Fleet uptime: 14d 6h 23m
```

### Dashboard Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Fleet Dashboard                        │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │ Room Health  │  │ Signal Chain│  │  Autonomy %    │  │
│  │ ■■■■■■■■■■  │  │ L0: ████ 8ms│  │                │  │
│  │ 12 online    │  │ L1: ████78ms│  │    87.3%       │  │
│  │  2 degraded  │  │ L2: ███390ms│  │   ▲ +2.1%     │  │
│  │  0 offline   │  │ L3: ██ 1.2s │  │   (24h trend)  │  │
│  └─────────────┘  └─────────────┘  └────────────────┘  │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Room Map (click for detail)                       │   │
│  │                                                   │   │
│  │  [Bridge]────[Galley]────[Fish Hold]             │   │
│  │     🟢         🟢           🟡                    │   │
│  │     │           │           │                     │   │
│  │  [Engine]────[Deck]─────[Cabin]                  │   │
│  │     🟢         🟢           🟢                    │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Alerts                                            │   │
│  │ ⚠ galley-smoke-01: 195ppm (threshold: 200ppm)   │   │
│  │ ⚠ fish-hold-temp-02: last reading 4m ago         │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Key Metrics to Monitor

| Metric | Healthy | Warning | Critical |
|---|---|---|---|
| **Room status** | `online` | `degraded` (1–2 sensors down) | `critical` (actuator failure) or `offline` |
| **L0 latency** | < 20ms | 20–50ms | > 50ms |
| **L1 latency** | < 100ms | 100–200ms | > 200ms |
| **L2 latency** | < 500ms | 500ms–1s | > 1s |
| **Sensor staleness** | < 60s | 60–180s | > 180s |
| **Autonomy %** | 80–95% | 60–80% | < 60% (under-trained) or > 98% (under-supervised) |
| **MQTT message loss** | < 0.1% | 0.1–1% | > 1% |
| **DGX GPU utilization** | 40–70% | 70–90% | > 90% (scale needed) |

---

## 9. Walkthrough: Fishing Vessel Demo

This walkthrough deploys a complete SuperInstance fleet on a commercial fishing vessel (F/V Aurora). The vessel has 6 rooms, each with sensors and actuators, coordinated by a DGX Spark in the wheelhouse.

### Hardware List

| Device | Qty | Location | Role |
|---|---|---|---|
| DGX Spark | 1 | Wheelhouse (bridge) | Fleet coordinator |
| Desktop RTX 4070 | 1 | Wheelhouse (bridge) | Multi-room L2 coordination |
| Jetson Orin Nano | 6 | One per room | Room controllers |
| ESP32-S3 + camera | 6 | One per room | Vision sensors |
| ESP32-S3 + INMP441 mic | 4 | Galley, bridge, engine, deck | Audio sensors |
| ESP32-S3 + DHT22 | 6 | One per room | Temperature/humidity |
| ESP32-S3 + MQ-2 | 2 | Galley, engine room | Smoke/gas detection |
| ESP32-S3 + relay | 3 | Galley, engine, fish hold | Actuator control |

### Step 1: Network Setup

```bash
# On the DGX Spark (serves as network backbone)
# Configure static IPs for heavy nodes
cat > /etc/openconstruct/network.conf << EOF
backbone:
  subnet: 192.168.1.0/24
  dgx-spark-01: 192.168.1.1
  desktop-01:   192.168.1.2
  jetson-bridge:   192.168.1.10
  jetson-galley:   192.168.1.11
  jetson-engine:   192.168.1.12
  jetson-fishhold: 192.168.1.13
  jetson-deck:     192.168.1.14
  jetson-cabin:    192.168.1.15

wifi:
  ssid: "FVAurora-Fleet"
  password: "<secure-password>"
  subnet: 192.168.4.0/24
EOF
```

### Step 2: Install OpenConstruct on Each Tier

```bash
# DGX Spark (wheelhouse)
ssh dgx-spark-01
curl -fsSL https://openconstruct.sh | bash
openconstruct start --tier=dgx-spark --coordinator

# Desktop (wheelhouse, second rack)
ssh desktop-01
curl -fsSL https://openconstruct.sh | bash
openconstruct start --tier=desktop

# Each Jetson (one per room)
for host in jetson-bridge jetson-galley jetson-engine jetson-fishhold jetson-deck jetson-cabin; do
  ssh $host "curl -fsSL https://openconstruct.sh | bash && openconstruct start --tier=jetson"
done
```

### Step 3: Flash ESP32s

```bash
# On the desktop, build and flash each ESP32
cd openconstruct-esp32

# Build once, flash many
idf.py set-target esp32s3
idf.py menuconfig  # Set: FLEET_ID, WiFi SSID, MQTT broker

for device in galley-cam galley-mic galley-temp galley-smoke galley-relay \
              bridge-cam bridge-mic bridge-temp \
              engine-cam engine-mic engine-temp engine-smoke engine-relay \
              fishhold-cam fishhold-temp fishhold-relay \
              deck-cam deck-mic deck-temp \
              cabin-cam cabin-temp; do
  idf.py menuconfig --set-CONFIG_DEVICE_ID=$device
  idf.py build
  idf.py flash -p /dev/ttyUSB0  # Swap cables per device
done
```

### Step 4: Create Room Configurations

```bash
# On DGX Spark
mkdir -p /etc/openconstruct/rooms

# Create each room YAML (example: galley)
cat > /etc/openconstruct/rooms/galley.yaml << 'ROOMEOF'
room:
  id: fv-aurora-galley
  name: "Galley"
  description: "Ship's kitchen and mess area, port side amidships"
  tier: jetson
  controller: jetson-galley

  senses:
    - id: galley-cam-01
      type: vision
      hardware: esp32s3-cam
      description: "Overhead camera covering stove and prep area"
      default_level: L2
    - id: galley-mic-01
      type: audio
      hardware: inmp441
      description: "Ceiling microphone"
      default_level: L2
    - id: galley-temp-01
      type: environment
      hardware: dht22
      description: "Temperature near stove"
      default_level: L1
      poll_interval: 30s
    - id: galley-smoke-01
      type: environment
      hardware: mq2
      description: "Smoke detector above stove"
      default_level: L1
      alert_threshold: 200ppm

  actuators:
    - id: galley-stove-relay
      type: relay
      description: "Emergency stove cutoff"
      safety: critical
    - id: galley-vent-fan
      type: relay
      description: "Ventilation fan"
      safety: normal

  policy:
    max_actuation_level: L3
    alert_escalation:
      smoke: [local_alarm, coordinator, captain_display]
ROOMEOF

# Repeat for bridge, engine, fish-hold, deck, cabin...
# (Similar structure, different sensors per room)
```

### Step 5: Create Fleet Configuration

```bash
cat > /etc/openconstruct/fleet.yaml << 'FLEETEOF'
fleet:
  id: fv-aurora-alpha
  name: "F/V Aurora Fleet Intelligence"
  coordinator: dgx-spark-01

  rooms:
    - include: rooms/bridge.yaml
    - include: rooms/galley.yaml
    - include: rooms/engine-room.yaml
    - include: rooms/fish-hold.yaml
    - include: rooms/deck.yaml
    - include: rooms/cabin.yaml

  mesh:
    discovery: mdns
    mqtt_broker: jetson-galley.local  # Primary MQTT hub
    grpc_backbone: dgx-spark-01.local:50051
    websocket_relay: dgx-spark-01.local:8080

  scheduling:
    timezone: UTC-9
    autonomy_limits:
      default: advisory
      engine_room: supervised
      bridge: advisory

  monitoring:
    health_interval: 60s
    dashboard: true
FLEETEOF
```

### Step 6: Deploy and Verify

```bash
# Start the fleet coordinator
openconstruct fleet deploy --config /etc/openconstruct/fleet.yaml

# Verify all rooms came online
openconstruct fleet status

# Expected output:
# Fleet: fv-aurora-alpha
# Rooms: 6/6 online
# Sensors: 22/22 online
# Signal Chain: L0 ✓ L1 ✓ L2 ✓
# Autonomy: 0% (new deployment — learning mode)

# Check individual room
openconstruct room describe fv-aurora-galley
# Room: fv-aurora-galley (Galley)
# Status: online
# Senses: 4 (cam, mic, temp, smoke)
# Actuators: 2 (stove-relay, vent-fan)
# Last L1 summary: "Empty galley, lights off, 68°F, smoke normal"
# Last L2 description: "The galley is dark and quiet. The stove is cold,
#   all surfaces appear clean. A faint hum from the refrigerator is
#   audible. Temperature is a comfortable 68°F with no smoke detected."
```

### Step 7: Monitor and Iterate

```bash
# Open the dashboard (WebSocket on DGX Spark)
openconstruct dashboard --bind 0.0.0.0:3000

# Watch live ticks
openconstruct fleet watch --room=fv-aurora-galley --level=L2

# Check autonomy trend after 24 hours
openconstruct fleet metrics --autonomy --period=24h
# Autonomy: 72.4% (181 decisions, 50 human overrides)
# Trend: improving (↑ 8.3% from yesterday)
# Most common override reason: "False positive on smoke detector"
# Action: Consider adjusting smoke threshold from 200ppm to 250ppm
```

---

## 10. Troubleshooting

### Common Issues

#### ESP32 Won't Connect to MQTT

| Symptom | Cause | Fix |
|---|---|---|
| No mDNS response | Wrong SSID/password | Check `idf.py menuconfig` → WiFi settings |
| MQTT connection refused | Wrong PSK or fleet_id | Re-flash with correct `FLEET_ID` and PSK |
| Connects then drops | Weak Wi-Fi signal | Move ESP32 closer to AP, or add a repeater |
| Birth message sent but no ticks | Sensor wiring issue | Check GPIO pin assignments in firmware |

```bash
# Debug ESP32 connection
# Connect via serial at 115200 baud
idf.py monitor -p /dev/ttyUSB0

# Look for:
# I (3214) wifi:connected with SSID, aid=1
# I (3218) mqtt_client: MQTT connected
# I (3220) openconstruct: Birth message published
```

#### Jetson L1 Inference Too Slow

| Symptom | Cause | Fix |
|---|---|---|
| Latency > 200ms | GPU not detected | `nvidia-smi` to verify CUDA; reinstall JetPack |
| OOM errors | Other processes using VRAM | Stop desktop GUI: `sudo systemctl set-default multi-user` |
| Batch timeout | Too many rooms per Jetson | Reduce batch_size or add a second Jetson |

```bash
# Check Jetson GPU status
sudo tegrastats
# RAM 4012/8192MB, SWAP 0/4096MB, GPU 1204/1534MHz
# EMC 2%@1600MHz, GPU 45%@1204MHz

# Verify model loaded correctly
openconstruct doctor --layer=L1
# ✓ plato-vision-nano-q8 loaded (8M params, 12 GPU layers)
# ✓ Inference latency: 78ms (target: <100ms)
```

#### Desktop L2 LoRA Not Converging

| Symptom | Cause | Fix |
|---|---|---|
| Loss not decreasing | Learning rate too high/low | Adjust `alpha` in l2-lora.yaml (try 16–64) |
| Garbage descriptions | Base model mismatch | Verify base model matches LoRA adapter version |
| Training too slow | Batch size too small | Increase batch_size; ensure CUDA is available |

```bash
# Check LoRA training status
openconstruct train status
# LoRA adapter: plato-nautical-lora-v3
# Base model: plato-vision-base-1b
# Last training: 2024-05-28T02:00:00Z (nightly)
# Loss: 0.34 (↓ from 0.41)
# New samples trained: 1,247
```

#### DGX Spark Fleet Coordinator Issues

| Symptom | Cause | Fix |
|---|---|---|
| Rooms showing offline | gRPC port blocked | Check firewall: `sudo ufw allow 50051/tcp` |
| Dashboard not loading | WebSocket not bound | Check `--bind` flag and port 8080/3000 |
| A2A deliberation failing | mDNS conflicts on multi-homed network | Set `OC_GRPC_ENDPOINT` explicitly |
| Cloud relay errors | No internet / DNS failure | Test: `curl -I https://api.openconstruct.cloud/health` |

```bash
# Full fleet diagnostics
openconstruct fleet diagnose

# Run on each tier to verify connectivity
openconstruct doctor --full
# ✓ Tier detected: dgx-spark
# ✓ CUDA: Grace CPU + Blackwell GPU (all layers available)
# ✓ gRPC server listening on :50051
# ✓ WebSocket relay listening on :8080
# ✓ A2A handler ready on :50052
# ✓ Cloud relay: reachable (latency 45ms)
# ✓ Fleet mesh: 6 rooms, 22 sensors, 5 actuators
# ✓ Signal chain: L0 ✓ L1 ✓ L2 ✓ L3 ✓ L4 ✓
```

#### Network Partition Recovery

When the mesh partitions (e.g., Wi-Fi drops between Jetson and ESP32 cluster):

1. **ESP32s continue autonomously** — L0 deadband filtering runs locally, ticks queue in flash
2. **Jetson detects partition** — marks affected rooms as `degraded`, alerts coordinator
3. **Coordinator reroutes** — if a second Jetson has connectivity, ESP32s reconnect via mDNS
4. **On reconnection** — queued ticks replay, L1/L2 catch up, room returns to `healthy`

```bash
# Check partition status
openconstruct fleet partitions
# No active partitions

# Force reconnect for a specific room
openconstruct room reconnect fv-aurora-galley --force
```

#### General Debug Commands

```bash
# Fleet-wide health check
openconstruct fleet status --verbose

# Per-room signal chain trace
openconstruct room trace fv-aurora-galley --from=L0 --to=L2

# Real-time MQTT traffic
mosquitto_sub -h jetson-galley.local -t "fleet/fv-aurora/#" -v

# gRPC health check
grpcurl -plaintext dgx-spark-01:50051 openconstruct.Fleet/Health

# WebSocket relay test
wscat -c ws://dgx-spark-01:8080/ws/fleet

# Full log dump for support
openconstruct fleet logs --dump > fleet-debug-$(date +%Y%m%d).tar.gz
```

---

## Appendix: Quick Reference Card

```
┌──────────────────────────────────────────────────────────────┐
│              OpenConstruct Fleet Quick Reference              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Install:   curl -fsSL https://openconstruct.sh | bash      │
│                                                              │
│  Start:     openconstruct start --tier=<esp32|jetson|        │
│             desktop|dgx-spark>                               │
│                                                              │
│  Deploy:    openconstruct fleet deploy --config fleet.yaml   │
│                                                              │
│  Status:    openconstruct fleet status                       │
│                                                              │
│  Room:      openconstruct room describe <room-id>            │
│                                                              │
│  Watch:     openconstruct fleet watch --room=<id> --level=L2 │
│                                                              │
│  Doctor:    openconstruct doctor [--full] [--layer=L0-L4]    │
│                                                              │
│  Dashboard: openconstruct dashboard --bind 0.0.0.0:3000      │
│                                                              │
│  Diagnose:  openconstruct fleet diagnose                     │
│                                                              │
│  Train:     openconstruct train status                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

*Part of the [SuperInstance OpenConstruct](https://github.com/SuperInstance/openconstruct-docs) ecosystem.*
