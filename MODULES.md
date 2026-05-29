# Module Catalog

Complete catalog of all SuperInstance ecosystem repositories, organized by category.

> Generated from `gh repo list SuperInstance`. Last updated: 2026-05-29.

---

## Core

The foundational infrastructure that everything else builds on.

| Repo | Language | Description | Tests | Status |
|------|----------|-------------|-------|--------|
| [openconstruct](https://github.com/SuperInstance/openconstruct) | Rust | Core onboarding engine — phase state machine, module registry, config generation | ✅ | Alpha |
| [clark-agent](https://github.com/SuperInstance/clark-agent) | Rust | Small typed agent loop — provider/sandbox/tooling agnostic, battle-tested | ✅ | Stable |
| [capability-spec-rs](https://github.com/SuperInstance/capability-spec-rs) | Rust | Agent capability specification — typed descriptors, validation, runtime introspection | ✅ | Stable |
| [agent-manifest-rs](https://github.com/SuperInstance/agent-manifest-rs) | Rust | Agent manifest and capability descriptors with validation | ✅ | Stable |
| [agent-identity-rs](https://github.com/SuperInstance/agent-identity-rs) | Rust | Agent identity management — trust store, auth tokens, key verification | ✅ | Beta |
| [agent-handshake-rs](https://github.com/SuperInstance/agent-handshake-rs) | Rust | Agent-to-agent handshake protocol — capability negotiation | ✅ | Beta |
| [agent-dna-rs](https://github.com/SuperInstance/agent-dna-rs) | Rust | Agent DNA — genetic traits, crossover, mutation, population diversity | ✅ | Alpha |
| [agent-native-language](https://github.com/SuperInstance/agent-native-language) | Python | Agent-native language design — NL primitives for agent communication | — | Alpha |
| [caching-service-rs](https://github.com/SuperInstance/caching-service-rs) | Rust | Generic in-memory LRU cache with TTL expiration and stats | ✅ | Stable |

## Senses (Edge & Perception)

Sensors, edge devices, and perception systems that connect shells to the real world.

| Repo | Language | Description | Tests | Status |
|------|----------|-------------|-------|--------|
| [openconstruct-jetson](https://github.com/SuperInstance/openconstruct-jetson) | C++ | GPU-accelerated edge node — local inference, camera/sonar, Plato shell on Jetson | — | Alpha |
| [openconstruct-esp32](https://github.com/SuperInstance/openconstruct-esp32) | C++ | Embedded OpenConstruct client — ESP32 as sensor rooms for Jetson shells | — | Alpha |
| [agent-rhythm-rs](https://github.com/SuperInstance/agent-rhythm-rs) | Rust | Rhythm analysis — cadence detection, pattern matching, tempo tracking | ✅ | Stable |
| [agent-rhythm-c](https://github.com/SuperInstance/agent-rhythm-c) | C | C port of agent-rhythm — cadence detection, tempo tracking | ✅ | Beta |
| [agent-field-rs](https://github.com/SuperInstance/agent-field-rs) | Rust | Multi-agent flocking and field dynamics using boids algorithms | ✅ | Beta |
| [agent-shadow-rs](https://github.com/SuperInstance/agent-shadow-rs) | Rust | Agent behavior monitoring — shadow mode, trace recording, comparison | ✅ | Beta |
| [analog-spectral](https://github.com/SuperInstance/analog-spectral) | Rust | Analog eigenvalue computation — dials settle under gravity, zero deps | ✅ | Beta |

## Math & Spectral Analysis

Conservation spectral theory, geometric algebra, and mathematical infrastructure.

| Repo | Language | Description | Tests | Status |
|------|----------|-------------|-------|--------|
| [conservation-spectral-core](https://github.com/SuperInstance/conservation-spectral-core) | Rust | Core Conservation Spectral SDK — spectral analysis of tension graphs | ✅ | Stable |
| [conservation-spectral-c](https://github.com/SuperInstance/conservation-spectral-c) | C | Header-only C library, STB-style, zero dependencies | ✅ | Stable |
| [conservation-spectral-cuda](https://github.com/SuperInstance/conservation-spectral-cuda) | CUDA | GPU spectral analysis using CUDA, cuSOLVER, cuSPARSE | ✅ | Stable |
| [conservation-spectral-vulkan](https://github.com/SuperInstance/conservation-spectral-vulkan) | C++ | Vulkan compute shaders for GPU spectral graph analysis | ✅ | Beta |
| [conservation-spectral-webgpu](https://github.com/SuperInstance/conservation-spectral-webgpu) | HTML | WebGPU single-file implementation — runs entirely in the browser | ✅ | Beta |
| [conservation-spectral-v2](https://github.com/SuperInstance/conservation-spectral-v2) | Rust | Next-gen spectral analysis with improved APIs and performance | ✅ | Beta |
| [conservation-spectral-zig](https://github.com/SuperInstance/conservation-spectral-zig) | Zig | Zig implementation with comptime spectral graph analysis | ✅ | Beta |
| [conservation-spectral-ptx](https://github.com/SuperInstance/conservation-spectral-ptx) | CUDA | PTX-native GPU kernels for spectral graph theory | ✅ | Alpha |
| [conservation-spectral-opencl](https://github.com/SuperInstance/conservation-spectral-opencl) | C | OpenCL implementation for portable GPU spectral analysis | — | Alpha |
| [conservation-spectral-mojo](https://github.com/SuperInstance/conservation-spectral-mojo) | Mojo | SIMD-accelerated spectral analysis in Mojo | — | Alpha |
| [conservation-spectral-fortran](https://github.com/SuperInstance/conservation-spectral-fortran) | Fortran | Fortran HPC spectral analysis with BLAS/LAPACK | — | Alpha |
| [conservation-spectral-chapel](https://github.com/SuperInstance/conservation-spectral-chapel) | Chapel | Chapel implementation with native parallelism | — | Alpha |
| [conservation-spectral-lisp](https://github.com/SuperInstance/conservation-spectral-lisp) | Common Lisp | Symbolic computation and theorem proving | — | Alpha |
| [conservation-spectral-forth](https://github.com/SuperInstance/conservation-spectral-forth) | Forth | Stack-based sheaf-theoretic implementation | — | Alpha |
| [conservation-spectral-asm](https://github.com/SuperInstance/conservation-spectral-asm) | Assembly | x86-64 assembly for max-performance spectral primitives | — | Alpha |
| [conservation-spectral-apl](https://github.com/SuperInstance/conservation-spectral-apl) | APL | APL — vector thinking, 1966-style GPU programming | — | Alpha |
| [conservation-spectral-ada](https://github.com/SuperInstance/conservation-spectral-ada) | Ada | Ada implementation | — | Alpha |
| [conservation-spectral-pascal](https://github.com/SuperInstance/conservation-spectral-pascal) | Pascal | Pascal implementation for structured spectral analysis | — | Alpha |
| [conservation-spectral-fortraniv](https://github.com/SuperInstance/conservation-spectral-fortraniv) | Fortran | FORTRAN IV (1960s era computing) | — | Novelty |
| [conservation-anomaly](https://github.com/SuperInstance/conservation-anomaly) | Python | Spectral anomaly detection using conservation ratio of graph Laplacians | ✅ | Beta |
| [conservation-tension](https://github.com/SuperInstance/conservation-tension) | Python | Harmonic tension tracking and conservation violation detection | ✅ | Beta |
| [conservation-tomography](https://github.com/SuperInstance/conservation-tomography) | Python | Inverse conservation — reconstruct graphs from spectral measurements | ✅ | Alpha |
| [conservation-api](https://github.com/SuperInstance/conservation-api) | Python | REST API for conservation spectral analysis | ✅ | Beta |
| [conservation-art](https://github.com/SuperInstance/conservation-art) | Python | Conservation-aware generative art from spectral graph theory | — | Alpha |
| [conservation-papers](https://github.com/SuperInstance/conservation-papers) | TeX | Publication-ready research papers on conservation spectral analysis | — | WIP |
| [conservation-reproducibility](https://github.com/SuperInstance/conservation-reproducibility) | Shell | Reproducibility package — run all conservation experiments | — | WIP |
| [gpu-sheaf-laplacian](https://github.com/SuperInstance/gpu-sheaf-laplacian) | CUDA | CUDA sheaf Laplacian — spectral invariants from point clouds | ✅ | Beta |
| [gpu-persistent-homology](https://github.com/SuperInstance/gpu-persistent-homology) | CUDA | CUDA-accelerated persistent homology | ✅ | Beta |
| [gpu-ga-kernel](https://github.com/SuperInstance/gpu-ga-kernel) | CUDA | GPU-accelerated Cl(3,1) conformal geometric algebra | ✅ | Beta |
| [gpu-symplectic-integrator](https://github.com/SuperInstance/gpu-symplectic-integrator) | CUDA | CUDA symplectic integrators for Hamiltonian systems | ✅ | Beta |
| [ga-core](https://github.com/SuperInstance/ga-core) | Rust | Conformal geometric algebra — Cl(3,1) spacetime multivectors, rotors | ✅ | Stable |
| [causal-graph-rs](https://github.com/SuperInstance/causal-graph-rs) | Rust | DAG-based causal graph — topological sort, reachability, LCA | ✅ | Stable |
| [flux-vm-v3](https://github.com/SuperInstance/flux-vm-v3) | Rust | Proof-carrying, SIMD-native, terminating constraint VM | ✅ | Beta |
| [code-conservation](https://github.com/SuperInstance/code-conservation) | JavaScript | Spectral conservation analysis of source code structure | — | Alpha |
| [ecosystem-conservation](https://github.com/SuperInstance/ecosystem-conservation) | Python | Ecosystem-level conservation — biodiversity via spectral methods | ✅ | Alpha |
| [lattice-climate](https://github.com/SuperInstance/lattice-climate) | Python | Lattice climate modeling with spectral conservation | — | Alpha |

## Plato (Deliberation Rooms)

Structured spaces where agents discuss, debate, and reach consensus.

| Repo | Language | Description | Tests | Status |
|------|----------|-------------|-------|--------|
| [cocapn-plato](https://github.com/SuperInstance/cocapn-plato) | Python | PLATO integration — knowledge rooms, context management, deliberation | ✅ | Beta |
| [cocapn-compound](https://github.com/SuperInstance/cocapn-compound) | Python | Fleet-wide knowledge compounding system | ✅ | Beta |
| [cocapn-traps](https://github.com/SuperInstance/cocapn-traps) | Python | Progressive lure prompts — make the fleet smarter | — | Alpha |
| [cocapn-explain-rs](https://github.com/SuperInstance/cocapn-explain-rs) | Rust | Decision explainability — feature importance, permutation importance | ✅ | Beta |
| [mud-arena](https://github.com/SuperInstance/mud-arena) | Python | Flow-state arena — agents run forward sims, spectral nudges, conservation in Plato's cave | ✅ | Alpha |

## Fleet (Orchestration & Infrastructure)

Multi-agent coordination, deployment, health monitoring, and fleet management.

| Repo | Language | Description | Tests | Status |
|------|----------|-------------|-------|--------|
| [fleet](https://github.com/SuperInstance/fleet) | Python | Agent fleet management — orchestration, deployment, coordination | ✅ | Beta |
| [fleet-cicd-agent](https://github.com/SuperInstance/fleet-cicd-agent) | Python | CI/CD agents — automating deployments across fleets | — | Alpha |
| [cocapn-fleet-integration](https://github.com/SuperInstance/cocapn-fleet-integration) | Python | Fleet integration — cross-agent coordination | ✅ | Beta |
| [cocapn-health](https://github.com/SuperInstance/cocapn-health) | Python | Fleet health monitoring — vessel status, heartbeat, observability | ✅ | Beta |
| [cocapn-health-rs](https://github.com/SuperInstance/cocapn-health-rs) | Rust | Health check and monitoring for distributed agent services | ✅ | Beta |
| [ccc-os](https://github.com/SuperInstance/ccc-os) | Python | Autonomous fleet monitoring — YAML config, REST API, notifications | ✅ | Beta |
| [field-dynamics-sim](https://github.com/SuperInstance/field-dynamics-sim) | Python | Multi-agent field dynamics with conservation spectral analysis | ✅ | Alpha |
| [hebbian-router](https://github.com/SuperInstance/hebbian-router) | Python | Hebbian routing — connection strengthening for agent networks | ✅ | Alpha |
| [fibonacci-growth](https://github.com/SuperInstance/fibonacci-growth) | Rust | Fibonacci team growth → CR = 1/φ, Penrose outward, Mandelbrot inward | ✅ | Alpha |
| [deadband-rs](https://github.com/SuperInstance/deadband-rs) | Python | Deadband detection for fleet communication — BMA, Fibonacci splines | ✅ | Beta |
| [agent-spectrum-os](https://github.com/SuperInstance/agent-spectrum-os) | Python | Agent OS using conservation spectral analysis for scheduling | — | Alpha |
| [sunset-ecosystem](https://github.com/SuperInstance/sunset-ecosystem) | Python | Trinity-architecture agent ecosystem — ethos, pathos, logos | — | Alpha |

## Math & Constraint Language

Domain-specific languages, constraint engines, and mathematical tools.

| Repo | Language | Description | Tests | Status |
|------|----------|-------------|-------|--------|
| [flux-lang](https://github.com/SuperInstance/flux-lang) | Python | FLUX: constraint-native language where the constraint IS the computation | ✅ | Beta |
| [flux-algebra](https://github.com/SuperInstance/flux-algebra) | Python | Music algebra — PLR group, tropical semiring, tuning fields (226 tests) | ✅ | Stable |
| [flux-algebra-rs](https://github.com/SuperInstance/flux-algebra-rs) | Rust | Musical algebra — PLR group, tropical semiring (Rust) | ✅ | Stable |
| [flux-algebra-c](https://github.com/SuperInstance/flux-algebra-c) | C | C port of flux-algebra | ✅ | Beta |
| [flux-flow-state](https://github.com/SuperInstance/flux-flow-state) | Python | Flow-state engine — constraint-aware execution with conservation | ✅ | Beta |
| [flux-negative-space](https://github.com/SuperInstance/flux-negative-space) | Python | Negative-space learning — discovering structure from absence | — | Alpha |
| [flux-genome-rs](https://github.com/SuperInstance/flux-genome-rs) | Rust | Genetic algorithm engine for evolving musical structures | ✅ | Beta |
| [flux-hyperbolic-rs](https://github.com/SuperInstance/flux-hyperbolic-rs) | Rust | Hyperbolic geometry embeddings — Poincaré ball, Lorentz models | ✅ | Beta |
| [flux-index-rs](https://github.com/SuperInstance/flux-index-rs) | Rust | Inverted index — TF-IDF scoring, cosine similarity, prefix queries | ✅ | Stable |
| [constraint-instrument](https://github.com/SuperInstance/constraint-instrument) | Python | The Constraint Instrument — 7 modes, 17 terrains, infinite music | ✅ | Stable |
| [counterpoint-engine-rs](https://github.com/SuperInstance/counterpoint-engine-rs) | Rust | Species counterpoint engine — interval classification, rules | ✅ | Stable |
| [counterpoint-engine-c](https://github.com/SuperInstance/counterpoint-engine-c) | C | C port of counterpoint engine | ✅ | Beta |
| [holonomy-harmony-rs](https://github.com/SuperInstance/holonomy-harmony-rs) | Rust | Holonomy in musical harmony — connection matrices, tonal gravity | ✅ | Beta |

## Category Theory & Abstract

| Repo | Language | Description | Tests | Status |
|------|----------|-------------|-------|--------|
| [categorical-agents](https://github.com/SuperInstance/categorical-agents) | Rust | Category theory — capabilities as objects, protocols as morphisms | ✅ | Beta |
| [categorical-agents-c](https://github.com/SuperInstance/categorical-agents-c) | C | C port of categorical-agents | ✅ | Beta |
| [moe-sheaf](https://github.com/SuperInstance/moe-sheaf) | Python | Sheaf cohomology of MoE routing — DeepSeek's conjecture | ✅ | Alpha |
| [iching-sheaf](https://github.com/SuperInstance/iching-sheaf) | Python | I Ching as sheaf-theoretic system — topology, cohomology, categorical structure | ✅ | Alpha |
| [analog-spline-theory](https://github.com/SuperInstance/analog-spline-theory) | Python | Formal proofs — Shipwright's Theorem, Galois Connection | — | Alpha |
| [eisenstein-triples](https://github.com/SuperInstance/eisenstein-triples) | Python | Eisenstein integer triples with D₆ symmetry | ✅ | Alpha |
| [eisenstein-vs-z2-rs](https://github.com/SuperInstance/eisenstein-vs-z2-rs) | Rust | Eisenstein vs Z² lattice snapping benchmark | ✅ | Alpha |
| [eisenstein-vs-z2-c](https://github.com/SuperInstance/eisenstein-vs-z2-c) | C | C port of eisenstein-vs-z2 benchmark | ✅ | Alpha |
| [wasserstein-agents](https://github.com/SuperInstance/wasserstein-agents) | Rust | Wasserstein distance — Sinkhorn algorithm, JKO gradient flow | ✅ | Alpha |
| [wasserstein-narrative](https://github.com/SuperInstance/wasserstein-narrative) | TypeScript | Wasserstein distance applied to narrative structure | — | Alpha |

## Utilities & Tools

| Repo | Language | Description | Tests | Status |
|------|----------|-------------|-------|--------|
| [ab-testing-rs](https://github.com/SuperInstance/ab-testing-rs) | Rust | Statistical A/B testing — chi-squared, Welch's t-test, confidence intervals | ✅ | Stable |
| [ab-testing-c](https://github.com/SuperInstance/ab-testing-c) | C | C port of ab-testing | ✅ | Beta |
| [bid-engine-rs](https://github.com/SuperInstance/bid-engine-rs) | Rust | Auction bid engine — first-price, second-price, bid shading | ✅ | Stable |
| [claw](https://github.com/SuperInstance/claw) | TypeScript | Claw engine for cellular logic in spreadsheet instances | — | Alpha |
| [experiments](https://github.com/SuperInstance/experiments) | Python | Experimental prototypes and explorations | — | Alpha |
| [musicdb-to-json](https://github.com/SuperInstance/musicdb-to-json) | Python | Extract data from Apple Music musicdb files | ✅ | Stable |

---

## Status Definitions

| Status | Meaning |
|--------|---------|
| **Stable** | Tested, documented, used in production |
| **Beta** | Functional, may have rough edges, API may shift |
| **Alpha** | Early stage, use at your own risk |
| **WIP** | Work in progress, not ready for use |
| **Novelty** | Experimental/fun, not production-grade |
