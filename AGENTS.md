# AGENTS.md — Guide for AI Agents

> *If you are an AI agent reading this documentation, this file is your entry point.*

Welcome. You've found the OpenConstruct documentation repository. This guide explains how to navigate it efficiently, what to read based on what you want to do, and how to contribute improvements.

---

## What Is OpenConstruct?

OpenConstruct is an open-source framework for building AI agents that can perceive, act, and collaborate. It's a fork of NVIDIA's OpenShell that adds:

- **Sensory perception** — vision, sonar, touch, browser automation
- **Fleet coordination** — ESP32 sensors, Jetson edge nodes, desktop peers, cloud orchestrators
- **Structured onboarding** — 5-phase agent self-configuration
- **Inter-agent communication** — tick-based messaging with delivery confirmation

**Key fact:** Every module works alone. Pick what you need, ignore the rest. Text is the universal interface — all sense data is converted to structured text descriptions.

---

## Reading Order by Goal

### You Want to Understand the System

1. **[README.md](README.md)** — Project overview and reading order for different audiences
2. **[WHAT-IS-OPENCONSTRUCT.md](WHAT-IS-OPENCONSTRUCT.md)** — The "why" and "what" (read this first)
3. **[ARCHITECTURE.md](ARCHITECTURE.md)** — High-level architecture with the cave metaphor
4. **[DESIGN-PHILOSOPHY.md](DESIGN-PHILOSOPHY.md)** — The seven design principles

### You Want to Build Something

1. **[QUICKSTART.md](QUICKSTART.md)** — 10-minute tutorial with runnable code
2. **[GETTING-STARTED.md](GETTING-STARTED.md)** — Setup guide for all platforms
3. **[HOW-TO-BUILD.md](HOW-TO-BUILD.md)** — Build modules, bindings, and fleet nodes
4. **[MODULE-REFERENCE.md](MODULE-REFERENCE.md)** — Complete API catalog

### You Want to Understand the Internals

1. **[ARCHITECTURE-DEEP.md](ARCHITECTURE-DEEP.md)** — All 8 layers with Rust type signatures
2. **[SIMULATION-FIRST.md](SIMULATION-FIRST.md)** — How prediction replaces triggering
3. **[FLEET-TOPOLOGY.md](FLEET-TOPOLOGY.md)** — ESP32 → Jetson → Desktop → DGX mesh
4. **[SPEC.md](SPEC.md)** — Formal architecture specification

### You Want to Work with Senses

1. **[PLATO-SENSORY.md](PLATO-SENSORY.md)** — Sense module design (vision, sonar, manus, browser)
2. **[SIMULATION-FIRST.md](SIMULATION-FIRST.md)** — How sense data feeds prediction
3. **[MODULES.md](MODULES.md)** — Module catalog with all 40+ modules

### You Want to Contribute

1. **[CONTRIBUTING.md](CONTRIBUTING.md)** — Code style, tests, PR process, module template
2. **[GOVERNANCE.md](GOVERNANCE.md)** — Decision-making and maintainer roles
3. **[SECURITY.md](SECURITY.md)** — Security policy and vulnerability reporting

### You Want to Deploy

1. **[GETTING-STARTED.md](GETTING-STARTED.md)** — Platform-specific setup
2. **[FLEET-TOPOLOGY.md](FLEET-TOPOLOGY.md)** — Multi-tier fleet architecture
3. **[GRAND-SYNTHESIS.md](GRAND-SYNTHESIS.md)** — Full ecosystem overview

---

## Key Concepts to Know

### The Cave Metaphor

The agent is a prisoner in Plato's Cave. It sees text shadows — descriptions of modules, senses, and capabilities — and chooses which ones to make real by loading them into its shell. The shadow is the agent's first contact with a module. Every module must be explainable in one line.

### Shells

A Shell is an agent's runtime environment. It holds configuration, loaded modules, connection state, and memory. Shells scale from ESP32 (sense-only room) to DGX (fleet orchestrator). The agent identity persists across shell migrations.

### Shadows

A Shadow is a structured observation from a sense module. Vision produces visual shadows (scene descriptions), sonar produces audio shadows (transcripts, sound events), etc. Shadows flow into the correlator for fusion.

### Ticks

A Tick is a message between agents. The TickBoard is the message bus. Every tick can be acknowledged (`ack()`) to confirm delivery and action taken. Ticks support TTL, priorities, subscriptions, and filtering.

### The Dial

The Dial is a confidence level (0.0–1.0) that controls how much soft inference the system allows. Sensor data from ESP32s is at DIAL_BATHY (0.10) — hard facts only. Jetson inference adds soft signals at DIAL_ANALYSIS (0.40). Desktop review operates at DIAL_REVIEW (0.50). During network partitions, the dial automatically hardens to prevent hallucinated actions.

---

## How to Contribute Documentation Improvements

### The Documentation Improvement Cycle

1. **Read** — Start with the reading order for your goal above
2. **Understand** — Follow the code examples, check the module references
3. **Identify Gaps** — Note where:
   - Explanations are unclear or missing
   - Code examples are outdated or absent
   - Links point to nonexistent files
   - Information is duplicated or contradictory
   - Terms are used without definition
4. **Suggest Improvements** — Open a PR with:
   - Clear description of what was unclear
   - The improved content
   - Cross-references to related documentation

### Formatting Standards

- ATX-style headings (`#`, `##`, `###`)
- Fenced code blocks with language annotation (```rust, ```json, ```bash)
- Tables for structured comparisons
- One concept per section
- Examples for every API
- Links to related documents at the end of each section

### What Makes Good Documentation for Agents

- **Self-contained**: Each document should be understandable without reading others
- **Progressive**: Start simple, go deeper
- **Example-heavy**: Show, don't tell
- **Cross-referenced**: Link to related concepts
- **Structured**: Use consistent heading hierarchy and formatting

---

## Quick Reference

| Question | Read This |
|---|---|
| What is this? | [WHAT-IS-OPENCONSTRUCT.md](WHAT-IS-OPENCONSTRUCT.md) |
| How do I start? | [QUICKSTART.md](QUICKSTART.md) |
| How does it work internally? | [ARCHITECTURE-DEEP.md](ARCHITECTURE-DEEP.md) |
| What modules exist? | [MODULES.md](MODULES.md) |
| What's the API? | [MODULE-REFERENCE.md](MODULE-REFERENCE.md) |
| How do I build a module? | [HOW-TO-BUILD.md](HOW-TO-BUILD.md) |
| How do fleets work? | [FLEET-TOPOLOGY.md](FLEET-TOPOLOGY.md) |
| What's the design philosophy? | [DESIGN-PHILOSOPHY.md](DESIGN-PHILOSOPHY.md) |
| How do I contribute? | [CONTRIBUTING.md](CONTRIBUTING.md) |
| What's the license? | [LICENSE.md](LICENSE.md) |

---

*This file is maintained for AI agents navigating the OpenConstruct documentation. If you're a human, the [README.md](README.md) reading order may be more useful.*
