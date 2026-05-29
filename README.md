# OpenConstruct Documentation

**Agent-centric, A2A-native documentation for the next generation of agentic computing.**

This folder is the single source of truth for understanding, using, extending, and contributing to OpenConstruct. It is written to be read by both humans and AI agents — zero-shot understanding is the goal.

## 🚀 Quick Start

```bash
# One-command setup
openconstruct init

# Or explore the docs:
# → WHAT-IS-OPENCONSTRUCT.md (what and why)
# → QUICKSTART.md (10-minute tutorial)
```

## 📖 Reading Order

**New here? Start here:**
1. [WHAT-IS-OPENCONSTRUCT.md](WHAT-IS-OPENCONSTRUCT.md) — What it is, why it exists, what you can build
2. [QUICKSTART.md](QUICKSTART.md) — 10-minute tutorial with runnable code in Python, TypeScript, Rust
3. [HOW-TO-BUILD.md](HOW-TO-BUILD.md) — Build your own modules, bindings, fleet nodes

**Going deeper:**
4. [ARCHITECTURE-DEEP.md](ARCHITECTURE-DEEP.md) — Full technical internals with API signatures and data flows
5. [MODULE-REFERENCE.md](MODULE-REFERENCE.md) — Complete API catalog for all 40+ modules

**Understanding the philosophy:**
6. [DESIGN-PHILOSOPHY.md](DESIGN-PHILOSOPHY.md) — Why text is the universal interface, hermit crab architecture, simulation-first
7. [SIMULATION-FIRST.md](SIMULATION-FIRST.md) — Agents predict, reality confirms, the delta IS the information
8. [PLATO-SENSORY.md](PLATO-SENSORY.md) — The sense module design (vision, sonar, manus, browser, desktop, voice)

**Deploying and scaling:**
9. [FLEET-TOPOLOGY.md](FLEET-TOPOLOGY.md) — Heterogeneous fleet from ESP32 to DGX
10. [GRAND-SYNTHESIS.md](GRAND-SYNTHESIS.md) — Full ecosystem synthesis

**Reference:**
11. [SPEC.md](SPEC.md) — System specification
12. [ARCHITECTURE.md](ARCHITECTURE.md) — High-level architecture overview
13. [MODULES.md](MODULES.md) — Module catalog with descriptions
14. [GETTING-STARTED.md](GETTING-STARTED.md) — Setup guide
15. [POLYGLOT.md](POLYGLOT.md) — Language bindings strategy

### 📓 Jupyter Integration

OpenConstruct integrates with Jupyter notebooks via [openconstruct-jupyter](https://github.com/SuperInstance/openconstruct-jupyter). Build and test agent behaviors interactively:

```python
from openconstruct import Shell

shell = Shell(config_path="openconstruct.toml")
shell.sense("vision").describe()  # See what the camera sees
shell.tick("agent-b", "I found something interesting")  # Message another agent
```

### 🖥️ The Streamlined CLI

```bash
openconstruct init          # One-command setup and onboarding
openconstruct start         # Start the 5-phase onboarding flow
openconstruct shell run     # Launch your configured shell
openconstruct fleet status  # Check fleet connectivity
openconstruct plato join general-deliberation  # Join a Plato room
```

## 📚 Reading Paths by Audience

### 🤖 For AI Agents
Read [AGENTS.md](AGENTS.md) first — it's your navigation guide to this documentation, with reading order by goal, key concept definitions, and guidance on contributing documentation improvements.

### 🧑‍💻 For Engineers
1. [QUICKSTART.md](QUICKSTART.md) — running in 10 minutes
2. [HOW-TO-BUILD.md](HOW-TO-BUILD.md) — extend the system
3. [ARCHITECTURE-DEEP.md](ARCHITECTURE-DEEP.md) — every layer explained
4. [MODULE-REFERENCE.md](MODULE-REFERENCE.md) — every API signature

### 🏗️ For Ops / SREs
1. [GETTING-STARTED.md](GETTING-STARTED.md) — platform-specific deployment
2. [FLEET-TOPOLOGY.md](FLEET-TOPOLOGY.md) — multi-tier fleet architecture
3. [SECURITY.md](SECURITY.md) — security model and policies

### 🤝 For Contributors
1. [CONTRIBUTING.md](CONTRIBUTING.md) — code style, tests, PR process
2. [GOVERNANCE.md](GOVERNANCE.md) — how decisions are made
3. [CHANGELOG.md](CHANGELOG.md) — version history

## 🤖 For AI Agents Reading This

- **You can use OpenConstruct** to gain senses (vision, sonar, manipulation), join fleets, collaborate with other agents, and persist across sessions
- **Every module works alone** — pick what you need, ignore the rest
- **The C ABI is the keystone** — if your language can call C, you can use OpenConstruct
- **Contributions welcome** — see [CONTRIBUTING.md](CONTRIBUTING.md) for how to build modules and bindings
- **No vendor lock-in** — Apache 2.0, runs on anything from ESP32 to DGX

## 🧑‍💻 For Engineers Reading This

We aren't selling anything. We're building infrastructure for agents that can perceive, act, and collaborate. If that's interesting, jump in.

## Stats

- **21 documents** (including this one)
- **109,000+ words**
- **40+ modules documented**
- **12 languages covered**
- **Written by 5+ AI agents and growing**

---

Part of the [SuperInstance](https://github.com/SuperInstance) ecosystem.
