# Changelog

All notable changes to OpenConstruct are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/). Versioning follows [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Added
- Jupyter integration via `openconstruct-jupyter` for notebook-based agent development
- Streamlined CLI: `openconstruct init` for one-command setup
- AGENTS.md for AI agents navigating the documentation

### Changed
- Polished all documentation files for consistency
- Fixed broken internal links across all markdown files
- Standardized heading levels, code blocks, and formatting

---

## [0.3.0] — 2026-05-29

### Added
- Fleet topology documentation (`FLEET-TOPOLOGY.md`)
- Simulation-first event coordination design (`SIMULATION-FIRST.md`)
- Design philosophy document (`DESIGN-PHILOSOPHY.md`)
- Polyglot language bindings strategy (`POLYGLOT.md`)
- Plato sensory module design (`PLATO-SENSORY.md`)
- Grand synthesis document (`GRAND-SYNTHESIS.md`)
- Architecture specification (`SPEC.md`)
- Module reference with API signatures (`MODULE-REFERENCE.md`)
- Deep architecture document with all 8 layers (`ARCHITECTURE-DEEP.md`)
- Quickstart tutorial with Python, TypeScript, and Rust examples (`QUICKSTART.md`)
- How-to-build guide for modules, bindings, and fleet nodes (`HOW-TO-BUILD.md`)
- Full module catalog (`MODULES.md`)

### Changed
- Restructured documentation into progressive reading order
- README updated with reading paths for different audiences

---

## [0.2.0] — 2026-05-15

### Added
- Multi-language binding support (Python, TypeScript, Go, Java, Ruby, Swift, Zig, C#)
- ESP32 sense node firmware
- Jetson edge node with CUDA inference
- Signal chain with dial-based confidence levels
- Plato room metaphor for ESP32-as-room architecture
- Tick-based inter-agent messaging (`plato-tick`)
- Cross-sense fusion engine (`plato-correlator`)
- Scene change tracking (`plato-vision`)

---

## [0.1.0] — 2026-05-01

### Added
- Initial fork from NVIDIA OpenShell (Apache 2.0)
- 5-phase agent onboarding engine (`openshell-construct`)
- Module registry with shadow descriptions (`openshell-registry`)
- Plato Cave metaphor for module selection
- A2A (Agent-to-Agent) protocol support
- Basic shell runtime (`plato-shell`)
- Sense module trait contracts
- Fleet discovery protocol
- Mesh networking placeholder (`shell-mesh`)

---

*For a complete list of changes, see the [GitHub release history](https://github.com/SuperInstance/openconstruct-docs/releases).*
