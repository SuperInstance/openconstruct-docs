# Quickstart: Your First OpenConstruct Agent in 10 Minutes

Welcome! In the next ten minutes you'll go from zero to a working agent that joins a fleet, perceives the world through sense shadows, communicates with other agents, and contributes to a shared knowledge graph. Every step is spelled out — no prior experience with OpenConstruct assumed.

---

## Table of Contents

1. [What You'll Build](#what-youll-build)
2. [Prerequisites](#prerequisites)
3. [Step 1: Install the SDK](#step-1-install-the-sdk)
4. [Step 2: Create Your Agent](#step-2-create-your-agent)
5. [Step 3: Run the Onboarding](#step-3-run-the-onboarding)
6. [Step 4: Connect to Fleet](#step-4-connect-to-fleet)
7. [Step 5: Subscribe to Senses](#step-5-subscribe-to-senses)
8. [Step 6: Post Your First Tick](#step-6-post-your-first-tick)
9. [Step 7: Create a Plato Room](#step-7-create-a-plato-room)
10. [What's Next?](#whats-next)
11. [Troubleshooting](#troubleshooting)

---

## What You'll Build

In 10 minutes you'll have a running agent that:

- **Registers with the fleet** — announces itself to the network and receives a unique agent ID.
- **Subscribes to sense shadows** — receives compressed perceptual streams (vision, sonar) from other agents and sensors.
- **Posts ticks when it detects events** — leaves timestamped messages that other agents can react to.
- **Responds to policy checks** — answers authorization queries from the fleet controller.
- **Joins a Plato room** — contributes tiles to a shared knowledge graph with conflict resolution.

You'll see every one of these things happen with your own eyes by the end of this guide.

---

## Prerequisites

You need **one** of the following language runtimes installed. Pick whichever you're most comfortable with — all three SDKs are feature-equivalent.

### Python (3.10+)

```bash
# macOS (Homebrew)
brew install python@3.12

# Ubuntu / Debian
sudo apt update && sudo apt install -y python3.12 python3.12-venv python3-pip

# Verify
python3 --version   # should print 3.10 or higher
```

### Node.js / TypeScript (18+)

```bash
# Using nvm (recommended)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc
nvm install 22

# Verify
node --version   # should print v18.x or higher
npm --version
```

### Rust (1.75+)

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"

# Verify
rustc --version   # should print 1.75.0 or higher
cargo --version
```

### Shared Dependency: OpenConstruct Daemon

Every agent connects to a locally running **openconstructd** daemon. Install it once:

```bash
# All platforms via Cargo
cargo install openconstructd

# Or download a pre-built binary
curl -sSL https://releases.openconstruct.dev/latest.sh | sh
```

Start the daemon before running your agent:

```bash
openconstructd --bind 127.0.0.1:7281 &
```

You should see `openconstructd listening on 127.0.0.1:7281`. Leave it running in a background terminal.

> **Tip:** If port 7281 is already in use, pass `--bind 127.0.0.1:7282` (or any free port) and set the `OC_BIND` environment variable in later steps to match.

---

## Step 1: Install the SDK

Install the SDK for your chosen language:

=== "Python"

```bash
# Create and activate a virtual environment (recommended)
python3 -m venv ~/.venvs/oc-quickstart
source ~/.venvs/oc-quickstart/bin/activate

# Install the SDK
pip install openconstruct
```

=== "TypeScript"

```bash
# Create a project directory
mkdir oc-quickstart && cd oc-quickstart
npm init -y

# Install the SDK
npm install @openconstruct/sdk
```

=== "Rust"

```bash
# Create a new binary crate
cargo new oc-quickstart
cd oc-quickstart

# Add the SDK dependency
cargo add openconstruct
```

Verify the installation by printing the SDK version:

=== "Python"

```bash
python3 -c "import openconstruct; print(openconstruct.__version__)"
```

=== "TypeScript"

```bash
npx ts-node -e "import { version } from '@openconstruct/sdk'; console.log(version);"
```

=== "Rust"

```bash
# Add a quick version check in src/main.rs, then:
cargo run -- --version
```

You should see something like `0.9.4`. If you get an import error, double-check your virtual environment or `node_modules`.

---

## Step 2: Create Your Agent

Create a file called `agent.py` (or `agent.ts` / `src/main.rs`) and paste the complete code below. This is a **fully working agent** — not a stub.

=== "Python — agent.py"

```python
import asyncio
import signal
from openconstruct import Agent, SenseKind, Tick, PolicyResponse

agent = Agent(
    name="quickstart-demo",
    version="0.1.0",
    tags=["tutorial", "demo"],
)

@agent.on_register
async def on_registered(identity):
    """Called once when the daemon accepts our registration."""
    print(f"✅ Registered as {identity.agent_id}")
    print(f"   Capabilities: {identity.capabilities}")

@agent.on_sense(Vision=SenseKind.VISION)
async def on_vision(shadow):
    """Called every time a vision shadow arrives."""
    print(f"👁  Vision shadow from {shadow.source_id}: "
          f"{shadow.width}x{shadow.height}, {len(shadow.detections)} detections")
    for det in shadow.detections[:3]:  # show first 3
        print(f"    - {det.label} ({det.confidence:.0%})")

@agent.on_sense(Sonar=SenseKind.SONAR)
async def on_sonar(shadow):
    """Called every time a sonar shadow arrives."""
    print(f"📡 Sonar shadow from {shadow.source_id}: "
          f"range={shadow.max_range_m:.1f}m, {len(shadow.returns)} returns")

@agent.on_policy
async def on_policy(check):
    """Respond to policy/authorization queries from the fleet."""
    print(f"🔒 Policy check: {check.action} on {check.resource}")
    return PolicyResponse(allow=True, reason="tutorial agent trusts everything")

@agent.on_tick
async def on_tick(tick):
    """Called when another agent posts a tick we're subscribed to."""
    print(f"💬 Tick from {tick.author_id}: {tick.payload}")

async def main():
    print("🚀 Starting quickstart-demo agent...")
    await agent.start(host="127.0.0.1", port=7281)

    # Keep running until Ctrl+C
    stop = asyncio.Event()
    loop = asyncio.get_event_loop()
    loop.add_signal_handler(signal.SIGINT, stop.set)
    loop.add_signal_handler(signal.SIGTERM, stop.set)
    await stop.wait()

    print("\n👋 Shutting down...")
    await agent.stop()

if __name__ == "__main__":
    asyncio.run(main())
```

=== "TypeScript — agent.ts"

```typescript
import {
  Agent,
  SenseKind,
  type PolicyResponse,
  type VisionShadow,
  type SonarShadow,
  type Tick,
} from "@openconstruct/sdk";

const agent = new Agent({
  name: "quickstart-demo",
  version: "0.1.0",
  tags: ["tutorial", "demo"],
});

agent.onRegister(async (identity) => {
  console.log(`✅ Registered as ${identity.agentId}`);
  console.log(`   Capabilities: ${identity.capabilities.join(", ")}`);
});

agent.onSense(SenseKind.Vision, async (shadow: VisionShadow) => {
  console.log(
    `👁  Vision shadow from ${shadow.sourceId}: ` +
      `${shadow.width}x${shadow.height}, ${shadow.detections.length} detections`,
  );
  for (const det of shadow.detections.slice(0, 3)) {
    console.log(`    - ${det.label} (${(det.confidence * 100).toFixed(0)}%)`);
  }
});

agent.onSense(SenseKind.Sonar, async (shadow: SonarShadow) => {
  console.log(
    `📡 Sonar shadow from ${shadow.sourceId}: ` +
      `range=${shadow.maxRangeM.toFixed(1)}m, ${shadow.returns.length} returns`,
  );
});

agent.onPolicy(async (check) => {
  console.log(`🔒 Policy check: ${check.action} on ${check.resource}`);
  return { allow: true, reason: "tutorial agent trusts everything" } satisfies PolicyResponse;
});

agent.onTick(async (tick: Tick) => {
  console.log(`💬 Tick from ${tick.authorId}: ${tick.payload}`);
});

async function main() {
  console.log("🚀 Starting quickstart-demo agent...");
  await agent.start({ host: "127.0.0.1", port: 7281 });

  // Keep running until Ctrl+C
  await new Promise<void>((resolve) => {
    process.on("SIGINT", () => resolve());
    process.on("SIGTERM", () => resolve());
  });

  console.log("\n👋 Shutting down...");
  await agent.stop();
}

main().catch((err) => {
  console.error("Fatal:", err);
  process.exit(1);
});
```

=== "Rust — src/main.rs"

```rust
use openconstruct::{Agent, AgentConfig, SenseKind, PolicyResponse, Signal};

#[tokio::main]
async fn main() -> openconstruct::Result<()> {
    let agent = Agent::new(AgentConfig {
        name: "quickstart-demo".into(),
        version: "0.1.0".into(),
        tags: vec!["tutorial".into(), "demo".into()],
    })?;

    // Registration callback
    agent.on_register(|identity| {
        println!("✅ Registered as {}", identity.agent_id);
        println!("   Capabilities: {:?}", identity.capabilities);
        async {}
    });

    // Vision shadow callback
    agent.on_sense(SenseKind::Vision, |shadow| {
        println!("👁  Vision shadow from {}: {}x{}, {} detections",
            shadow.source_id,
            shadow.width,
            shadow.height,
            shadow.detections.len(),
        );
        for det in shadow.detections.iter().take(3) {
            println!("    - {} ({:.0}%)", det.label, det.confidence * 100.0);
        }
        async {}
    });

    // Sonar shadow callback
    agent.on_sense(SenseKind::Sonar, |shadow| {
        println!("📡 Sonar shadow from {}: range={:.1}m, {} returns",
            shadow.source_id,
            shadow.max_range_m,
            shadow.returns.len(),
        );
        async {}
    });

    // Policy callback
    agent.on_policy(|check| {
        println!("🔒 Policy check: {} on {}", check.action, check.resource);
        async {
            PolicyResponse::allow("tutorial agent trusts everything")
        }
    });

    // Tick callback
    agent.on_tick(|tick| {
        println!("💬 Tick from {}: {}", tick.author_id, tick.payload);
        async {}
    });

    println!("🚀 Starting quickstart-demo agent...");
    agent.start("127.0.0.1:7281").await?;

    // Wait for Ctrl+C
    Signal::ctrl_c().await?;
    println!("\n👋 Shutting down...");
    agent.stop().await?;

    Ok(())
}
```

Don't run it yet — we'll get there in Step 3. Just make sure the file compiles:

```bash
# Python: syntax check
python3 -c "import py_compile; py_compile.compile('agent.py', doraise=True)"

# TypeScript: type check
npx tsc --noEmit agent.ts

# Rust: build check
cargo check
```

If you see no errors, you're golden.

---

## Step 3: Run the Onboarding

Now the fun begins. Start your agent:

=== "Python"

```bash
python3 agent.py
```

=== "TypeScript"

```bash
npx ts-node agent.ts
```

=== "Rust"

```bash
cargo run
```

You should see the following output, phase by phase:

**Phase 1 — Connection:**
```
🚀 Starting quickstart-demo agent...
   Connecting to 127.0.0.1:7281...
   WebSocket handshake complete.
```

**Phase 2 — Registration:**
```
✅ Registered as agent_01HZQXKV3JN8R4M5TB7Y9A0FGP
   Capabilities: tick, sense, policy
```

The long string is your **agent ID** — it's a ULID, unique across the entire fleet. Write it down (or copy it); you'll use it in Step 6.

**Phase 3 — Capability negotiation:**
```
   Negotiated senses: vision, sonar
   Subscribed to fleet broadcasts.
```

If you see all three phases, your agent is live and connected to the daemon. It's now listening for sense shadows, ticks, and policy checks. Keep it running — we'll interact with it in the next steps.

> **No output?** Make sure `openconstructd` is running (Step 1) and listening on port 7281.

---

## Step 4: Connect to Fleet

Open a **second terminal** and use the bundled CLI to inspect the fleet topology:

```bash
# List all connected agents
openconstruct fleet list
```

You should see output like:

```
AGENT_ID                             NAME              STATUS    UPTIME
agent_01HZQXKV3JN8R4M5TB7Y9A0FGP    quickstart-demo   online    0:00:42
```

That's your agent! Let's see more detail:

```bash
# Show full topology including routes between agents
openconstruct fleet topology
```

```
quickstart-demo (agent_01HZQX…)
  ├─ senses:  vision, sonar
  ├─ policies: tick→allow-all, sense→allow-all
  └─ routes:
       (no peer routes — single-agent fleet)
```

Right now it's just you. In a real deployment, you'd see dozens of agents with routes showing how they communicate. The topology command is your bird's-eye view.

To see raw fleet events in real time:

```bash
openconstruct fleet watch
```

Leave this running in your second terminal — you'll see events appear as you work through the next steps.

---

## Step 5: Subscribe to Senses

Your agent is already listening for sense shadows (we set that up in Step 2). Now let's **generate** one so you can see it in action.

In a third terminal, use the CLI to publish a synthetic vision shadow:

```bash
openconstruct sense publish vision \
  --source-id sensor-camera-01 \
  --width 640 \
  --height 480 \
  --detections '[{"label":"person","confidence":0.92,"x":120,"y":80,"w":200,"h":400},
                 {"label":"chair","confidence":0.87,"x":340,"y":200,"w":150,"h":180}]'
```

Switch back to your **first terminal** (where the agent is running). Within a second, you should see:

```
👁  Vision shadow from sensor-camera-01: 640x480, 2 detections
    - person (92%)
    - chair (87%)
```

Let's do the same with sonar:

```bash
openconstruct sense publish sonar \
  --source-id sensor-sonar-north \
  --max-range-m 12.0 \
  --returns '[{"distance_m":3.2,"angle_deg":0,"intensity":0.8},
             {"distance_m":7.5,"angle_deg":15,"intensity":0.4}]'
```

Your agent prints:

```
📡 Sonar shadow from sensor-sonar-north: range=12.0m, 2 returns
```

You're now receiving real perceptual data from the fleet. In a production setup, real sensors would publish these shadows continuously — your agent would process every frame.

---

## Step 6: Post Your First Tick

A **tick** is a timestamped message that any agent can post. Other agents that are subscribed (or in the same fleet) receive it. Let's post one from the CLI and have your agent receive it.

```bash
openconstruct tick post \
  --author-id agent_01HZQXKV3JN8R4M5TB7Y9A0FGP \
  --payload '{"text":"Hello from the quickstart guide!","priority":"info"}'
```

> **Replace the agent ID** with the one you noted in Step 3.

In your agent's terminal:

```
💬 Tick from agent_01HZQXKV3JN8R4M5TB7Y9A0FGP: {"text":"Hello from the quickstart guide!","priority":"info"}
```

You can also post a tick **programmatically**. Add this to your agent code, right before the `await stop.wait()` (Python) or the `Signal::ctrl_c()` line (Rust):

=== "Python (add before the wait)"

```python
    # Post a tick from our agent
    await agent.post_tick({
        "text": "I'm alive and processing!",
        "priority": "info",
        "source": "quickstart",
    })
    print("📤 Posted a tick!")
```

=== "TypeScript (add before the SIGINT handler)"

```typescript
// Post a tick from our agent
await agent.postTick({
  text: "I'm alive and processing!",
  priority: "info",
  source: "quickstart",
});
console.log("📤 Posted a tick!");
```

=== "Rust (add before Signal::ctrl_c())"

```rust
    // Post a tick from our agent
    agent.post_tick(serde_json::json!({
        "text": "I'm alive and processing!",
        "priority": "info",
        "source": "quickstart",
    })).await?;
    println!("📤 Posted a tick!");
```

Restart your agent and you'll see it post the tick on startup, then receive it back (agents receive their own ticks by default).

---

## Step 7: Create a Plato Room

A **Plato room** is a shared knowledge graph where agents collaboratively build understanding. Each room has:

- **Tiles** — individual knowledge nodes (facts, observations, hypotheses)
- **CR (Conflict Resolution)** — automatic merge when multiple agents write to the same tile

Let's create one. In your second terminal:

```bash
# Create a new Plato room
openconstruct plato create --name "demo-room" --description "Quickstart tutorial room"
```

Output:

```
Room created: room_01HZQXKV3JN8R4M5TB7Y9A0FGP
  Name:        demo-room
  Description: Quickstart tutorial room
  CR strategy: last-writer-wins
```

Now add a tile:

```bash
openconstruct plato tile add \
  --room room_01HZQXKV3JN8R4M5TB7Y9A0FGP \
  --key "observation.hallway.person" \
  --value '{"detected":true,"confidence":0.92,"timestamp":"2026-05-29T10:45:00Z"}'
```

Output:

```
Tile added: observation.hallway.person
  Value: {"detected":true,"confidence":0.92,"timestamp":"2026-05-29T10:45:00Z"}
  Author: cli
  Version: 1
```

Let a second agent write to the same tile to see conflict resolution in action:

```bash
openconstruct plato tile update \
  --room room_01HZQXKV3JN8R4M5TB7Y9A0FGP \
  --key "observation.hallway.person" \
  --value '{"detected":false,"confidence":0.99,"timestamp":"2026-05-29T10:46:00Z"}' \
  --author "agent-camera-02"
```

Output:

```
Tile updated: observation.hallway.person
  Previous: v1 by cli
  Current:  v2 by agent-camera-02
  CR: last-writer-wins → accepted
```

View the full knowledge graph:

```bash
openconstruct plato graph --room room_01HZQXKV3JN8R4M5TB7Y9A0FGP
```

```
demo-room (room_01HZQX…)
  └─ observation.hallway.person
       v1 (cli):   {"detected":true,  "confidence":0.92, …}
       v2 (agent-camera-02): {"detected":false, "confidence":0.99, …}
       HEAD: v2
```

That's a Plato room — a living, versioned knowledge graph that multiple agents can write to simultaneously. Conflict resolution keeps it consistent even when agents disagree.

---

## What's Next?

You've got a running agent. Here's where to go from here:

- **[HOW-TO-BUILD.md](./HOW-TO-BUILD.md)** — Build a real agent with custom sense handlers, state management, and error recovery.
- **[ARCHITECTURE-DEEP.md](./ARCHITECTURE-DEEP.md)** — Understand the internals: shadow compression, fleet gossip, Plato CR strategies, and the event bus.
- **[MODULE-REFERENCE.md](./MODULE-REFERENCE.md)** — Full API reference for every SDK module: `Agent`, `SenseShadow`, `Tick`, `PlatoRoom`, `PolicyEngine`, and more.

Other resources:

- [API Reference (web)](https://docs.openconstruct.dev/api)
- [Examples repository](https://github.com/openconstruct/examples)
- [Community Discord](https://discord.gg/openconstruct)

---

## Troubleshooting

### 1. "Connection refused" when starting the agent

**Symptom:** `ConnectionRefused` or `ECONNREFUSED 127.0.0.1:7281`

**Fix:** The daemon isn't running. Start it:

```bash
openconstructd --bind 127.0.0.1:7281
```

If you're using a custom port, make sure the agent code and the `--bind` argument match. You can also set the environment variable:

```bash
export OC_BIND=127.0.0.1:7282
```

### 2. "Version mismatch" warning

**Symptom:** `WARN  SDK version 0.9.4 does not match daemon version 0.9.2`

**Fix:** Update the SDK and daemon to the same version:

```bash
# Python
pip install --upgrade openconstruct
cargo install openconstructd --force

# TypeScript
npm update @openconstruct/sdk
cargo install openconstructd --force
```

Minor version differences (0.9.2 vs 0.9.4) are usually fine, but matching versions avoid subtle protocol differences.

### 3. No sense shadows arriving

**Symptom:** Agent is registered but `on_vision` / `on_sonar` callbacks never fire.

**Fix:** Sense shadows only arrive when something publishes them. Either:

- Use `openconstruct sense publish` (as in Step 5) to generate synthetic shadows.
- Or connect a real sensor that publishes to the daemon.
- Or run the built-in simulator: `openconstructd --bind 127.0.0.1:7281 --simulate`

The `--simulate` flag generates synthetic vision and sonar shadows every 2 seconds.

### 4. "Policy denied" when posting a tick

**Symptom:** `TickRejected: policy check failed — agent not authorized for tick:write`

**Fix:** The daemon's default policy may require explicit authorization. You can:

- Start the daemon with the permissive tutorial policy:
  ```bash
  openconstructd --bind 127.0.0.1:7281 --policy allow-all
  ```
- Or add your agent to the policy ACL:
  ```bash
  openconstruct policy grant --agent agent_01HZQXKV3JN8R4M5TB7Y9A0FGP --action tick:write
  ```

### 5. Rust compilation error with `openconstruct` crate

**Symptom:** `error[E0432]: unresolved import openconstruct::Agent` or linker errors.

**Fix:** Ensure your `Cargo.toml` has the correct features enabled:

```toml
[dependencies]
openconstruct = { version = "0.9", features = ["full"] }
tokio = { version = "1", features = ["full"] }
serde_json = "1"
```

Then clean and rebuild:

```bash
cargo clean && cargo build
```

If the error persists, check that your Rust toolchain is up to date:

```bash
rustup update stable
```

---

**That's it!** You've built and run a complete OpenConstruct agent. It registers, perceives, communicates, and reasons. Not bad for ten minutes. Go build something amazing.
