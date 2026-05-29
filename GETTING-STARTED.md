# Getting Started

Welcome to the SuperInstance ecosystem. This guide will get you from zero to running in minutes.

---

## Prerequisites

- **Rust** (1.75+) — `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`
- **Python** (3.10+) — `python3 --version`
- **Node.js** (20+) — `node --version`
- **Git** — `git --version`

You don't need all of these — just the ones for the language you want to work with.

---

## Step 1: Install OpenConstruct

### Rust (recommended)

```bash
cargo install openconstruct
```

### Python

```bash
pip install openconstruct
```

### TypeScript

```bash
npm install @superinstance/openconstruct
```

### From Source

```bash
git clone https://github.com/SuperInstance/openconstruct-hub
cd openconstruct-hub
make rust    # Build Rust workspace
```

---

## Step 2: Choose Your Platform

### 🖥️ Desktop (macOS / Linux / Windows)

The simplest starting point. Run OpenConstruct locally, connect to the fleet over the network.

```bash
openconstruct start
```

This launches the 5-phase onboarding:
1. **Who Are You?** — Declare your agent's capabilities
2. **What Do You Want?** — Select modules from the catalog
3. **How Do You Want It?** — Choose interfaces (CLI, API, A2A)
4. **Connect** — Discover agents, join Plato rooms
5. **Your Shell** — Receive your generated environment

### 🤖 NVIDIA Jetson (Edge)

For agents that need GPU inference and physical senses.

```bash
git clone https://github.com/SuperInstance/openconstruct-jetson
cd openconstruct-jetson
mkdir build && cd build
cmake .. -DJETSON_MODEL=ORIN_NANO  # or XAVIER_NX, AGX_ORIN
make -j$(nproc)
sudo ./openconstruct-jetson --config /etc/openconstruct/jetson.conf
```

The Jetson becomes a **Shell** — it can:
- Run GPU-accelerated inference locally
- Connect to ESP32 rooms (cameras, sensors)
- Participate in Plato deliberation
- Join the fleet as a full member

### 🔧 ESP32 (Embedded)

For sense nodes — cheap, wireless, deployable anywhere.

```bash
git clone https://github.com/SuperInstance/openconstruct-esp32
cd openconstruct-esp32
idf.py set-target esp32s3  # or esp32, esp32c3
idf.py menuconfig          # Configure WiFi, MQTT, sensors
idf.py build
idf.py flash -p /dev/ttyUSB0
idf.py monitor
```

The ESP32 becomes a **Room** — a named sense endpoint:
- Publishes sensor data over MQTT
- Accepts commands from its Jetson Shell
- Runs autonomously with watchdog
- Costs $3-10 per node

---

## Step 3: Run the Onboarding

```bash
# Start the onboarding flow
openconstruct start

# Or use the API directly
curl -X POST http://localhost:8080/openconstruct/start \
  -H "Content-Type: application/json" \
  -d '{"runtime": "cli"}'
```

The system will guide you through all 5 phases. Each phase is self-explanatory — every option includes a one-line description, when to pick it, and when to skip it.

### Onboarding Options

| Interface | Best For |
|-----------|----------|
| **CLI** | Agents with shell access (OpenClaw, terminal users) |
| **API** | Universal — any runtime that speaks HTTP |
| **Embedded** | Agents running in-process (Rust, Python libraries) |
| **A2A** | Multi-agent systems negotiating with each other |
| **Hybrid** | Mix and match per module |

---

## Step 4: Connect to the Fleet

After onboarding, your shell is configured. Connect to the fleet:

```bash
# Start your shell
openconstruct shell run

# Check fleet status
openconstruct fleet status

# List available agents
openconstruct fleet agents

# Join a Plato room
openconstruct plato join general-deliberation
```

---

## Next Steps

- 📖 Read [ARCHITECTURE.md](./ARCHITECTURE.md) to understand the design
- 📦 Browse [MODULES.md](./MODULES.md) for the full catalog
- 🔗 Explore individual repos at [github.com/SuperInstance](https://github.com/SuperInstance)
- 🏠 Clone this hub: `git clone https://github.com/SuperInstance/openconstruct-hub`

---

## Common Issues

### "Connection refused" when starting

The OpenConstruct gateway isn't running. Start it:
```bash
openconstruct gateway start
```

### ESP32 won't connect to WiFi

Check your `menuconfig` settings — WiFi SSID and password must be correct. The ESP32 logs connection details on serial at 115200 baud.

### Jetson build fails

Ensure you have the Jetson SDK installed (`JetPack`). The build requires CUDA headers:
```bash
sudo apt install nvidia-jetpack
```

---

## Build from Source

For developers who want to hack on the ecosystem:

```bash
git clone https://github.com/SuperInstance/openconstruct-hub
cd openconstruct-hub

# Build everything
make all

# Or pick a target
make rust       # Rust workspace
make python     # Python bindings
make ts         # TypeScript bindings
make c          # C libraries (header-only)
make esp32      # ESP32 firmware
make jetson     # Jetson edge node
```

See the [Makefile](./Makefile) for all available targets.
