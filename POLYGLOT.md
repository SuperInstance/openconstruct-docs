# OpenConstruct Polyglot Strategy

> OpenConstruct = NVIDIA/OpenShell (Apache 2.0, Rust core) + SuperInstance onboarding layer.
> Goal: any agent in any runtime can use it.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Client Models: Thin vs Full](#client-models-thin-vs-full)
3. [P0 Languages](#p0-languages---ships-first)
4. [P1 Languages](#p1-languages---second-wave)
5. [P2 Languages](#p2-languages---community-driven)
6. [API Surface Sketches (P0)](#api-surface-sketches-p0)
7. [Build & Distribution](#build--distribution)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                 Agent Runtime                    │
│  (Python, JS/TS, Rust, Go, Java, Swift, ...)    │
└──────────────┬──────────────────┬───────────────┘
               │                  │
        ┌──────▼──────┐   ┌──────▼──────┐
        │ Full Client  │   │ Thin Client │
        │ (native SDK) │   │  (HTTP/gRPC)│
        └──────┬──────┘   └──────┬──────┘
               │                  │
        ┌──────▼──────────────────▼──────┐
        │       OpenConstruct Core       │
        │  (Rust · OpenShell foundation) │
        │  + SuperInstance onboarding    │
        └────────────────────────────────┘
```

**Binding strategies:**

| Strategy | When to use | Trade-off |
|----------|-------------|-----------|
| **FFI** (C ABI) | Performance-critical, shared memory | Unsafe boundary, build complexity |
| **Native rewrite** | Only when FFI impractical (browser, WASM) | Maintenance burden, drift risk |
| **Wrapper** (thin HTTP/gRPC) | Quick coverage, any language | Latency, requires running server |

---

## Client Models: Thin vs Full

### Thin Client

- **What:** HTTP (REST or gRPC-Web) client that talks to a running OpenConstruct server.
- **Code needed per language:** ~200-500 LOC (HTTP client, type definitions, serialization).
- **Depends on:** A running OpenConstruct server instance (local or remote).
- **Best for:** Languages where FFI is painful, rapid prototyping, sandboxed runtimes.

```text
Agent → HTTP POST /api/v1/onboard/start → Server runs Phase 1-5 → returns config
Agent → HTTP POST /api/v1/modules/register → Server tracks module → returns module ID
Agent → HTTP GET  /api/v1/config/{session_id} → returns generated config
```

**Thin client spec (language-agnostic):**

```
Endpoints:
  POST /api/v1/onboard/start
    Body: { agent_id, capabilities: [...], preferences: {...} }
    Response: { session_id, phase: 1, next_action: "..." }

  POST /api/v1/onboard/advance
    Body: { session_id, phase_data: {...} }
    Response: { session_id, phase: N, next_action: "..." }

  POST /api/v1/modules/register
    Body: { session_id, module: { name, version, provides: [...], requires: [...] } }
    Response: { module_id, status: "registered" }

  GET  /api/v1/config/{session_id}
    Response: { config: {...}, format: "json", generated_at: "..." }

  GET  /api/v1/health
    Response: { status: "ok", version: "..." }
```

### Full Client

- **What:** Native implementation of the Phase 1-5 onboarding flow.
- **Code needed:** Substantial — protocol logic, state machine, config generation.
- **Depends on:** Only the core Rust library (via FFI) or a native reimplementation.
- **Best for:** Rust, Python, TypeScript/JS — the big three where agents actually live.

**Phase flow (all full clients implement this state machine):**

```
Phase 1: Discovery    → probe environment, detect capabilities
Phase 2: Negotiation  → agree on protocols, auth, data formats
Phase 3: Integration  → register modules, wire dependencies
Phase 4: Validation   → run smoke tests, verify config
Phase 5: Activation   → write final config, go live
```

**Who ships as full client:**
| Language | Strategy | Reason |
|----------|----------|--------|
| Rust | Native | It IS the core |
| Python | FFI via PyO3 + Pythonic wrapper | OpenShell already has PyPI package |
| TypeScript/JS | WASM build of core + TS wrapper | Browser + Node.js |

---

## P0 Languages - Ships First

### Rust (Core)

- **Already exists:** OpenShell's full Rust stack (crates.io). SuperInstance onboarding logic to be added.
- **Needs building:** Phase 1-5 state machine, `openconstruct-core` crate, `openconstruct-cli` binary, server mode.
- **Binding strategy:** Native. This IS the implementation.
- **Distribution:** crates.io, static binary, Docker image.

### Python

- **Already exists:** OpenShell has a PyPI package (`open-shell` / `openshell`) with Python bindings.
- **Needs building:** `openconstruct` PyPI package wrapping the new onboarding flow. PyO3 bindings to Rust core.
- **Binding strategy:** FFI via PyO3 (already proven in OpenShell), plus a Pythonic high-level API.
- **Distribution:** PyPI, conda-forge.

### TypeScript / JavaScript

- **Already exists:** Nothing from OpenShell. Web isn't OpenShell's target.
- **Needs building:**
  - `@openconstruct/core` — WASM build of Rust core (via wasm-bindgen or wasm-pack)
  - `@openconstruct/client` — Thin HTTP client for server mode
  - `@openconstruct/node` — Node.js native addon via NAPI-RS (optional, for perf)
- **Binding strategy:** WASM for browser/Deno/Bun, NAPI-RS for Node.js native, HTTP client as fallback.
- **Distribution:** npm, deno.land/x.

### C (Universal FFI)

- **Already exists:** OpenShell may expose a C ABI internally (Rust `#[no_mangle]`).
- **Needs building:** `openconstruct.h` — a stable C header file wrapping the core onboarding API. This becomes the foundation for C++, Zig, Go, and any language that can call C.
- **Binding strategy:** FFI via C ABI (`cdylib` crate type).
- **Distribution:** System library (.so/.dylib/.dll), header file, pkg-config.

---

## P1 Languages - Second Wave

### C++ (CUDA / Performance)

- **Already exists:** Nothing. C header can be consumed directly.
- **Needs building:** `openconstruct.hpp` — modern C++17+ wrapper over the C ABI. RAII wrappers, STL types, CUDA interop helpers.
- **Binding strategy:** FFI over C ABI, wrapped in idiomatic C++.
- **Priority rationale:** AI infra lives in C++. CUDA kernels, TensorRT plugins, inference servers. Must feel native.

### Zig (Modern Systems)

- **Already exists:** Nothing.
- **Needs building:** `openconstruct-zig` — `@cImport` the C header directly, then add Zig-flavored ergonomics.
- **Binding strategy:** Direct C interop (Zig's `@cImport` is first-class).
- **Priority rationale:** Zig is the darling of modern systems programming. Ultralight, cross-compile friendly. Easy win since C ABI exists.

### Go (Cloud-Native)

- **Already exists:** Nothing.
- **Needs building:** `openconstruct-go` — CGo wrapper over C ABI, plus a pure-Go thin HTTP client.
- **Binding strategy:** CGo for full client, pure Go HTTP for thin client.
- **Priority rationale:** Kubernetes operators, cloud agents, infrastructure tooling. Go is the language of the cloud.

### Java / Kotlin (Enterprise + Android)

- **Already exists:** Nothing.
- **Needs building:** `openconstruct-jvm` — JNA or JNI wrapper over C library. Kotlin Multiplatform for Android.
- **Binding strategy:** JNA (easier) or JNI (faster) over C ABI. Thin HTTP client as alternative.
- **Priority rationale:** Enterprise environments, Android on-device agents. Large existing audience.

---

## P2 Languages - Community-Driven

| Language | Strategy | Rationale |
|----------|----------|-----------|
| **Swift** | C ABI interop via `swift import` | iOS/macOS agents. Apple's C interop is seamless. |
| **Julia** | `ccall` over C ABI | Scientific computing, HPC. Julia's FFI is trivial. |
| **Fortran** | `iso_c_binding` over C ABI | Legacy HPC codes. Rare but high-value in supercomputing. |
| **Chapel** | C ABI interop | HPC language (Crays). Niche but aligned with NVIDIA ecosystem. |
| **Ada** | C ABI interop | Safety-critical systems (aerospace, defense). Small but important niche. |
| **Mojo** | Python interop (Mojo is Python-superset) | AI-native language. If Python works, Mojo mostly works. Watch for native FFI. |
| **WebGPU / WGSL** | WASM + WebGPU compute shaders | Browser-based GPU agents. Forward-looking; depends on WASM + compute API maturity. |

---

## API Surface Sketches (P0)

### Rust (Core)

```rust
use openconstruct::{OnboardBuilder, ModuleRegistry, Config};

fn main() -> Result<()> {
    // Phase 1-5 onboarding in one flow
    let session = OnboardBuilder::new()
        .agent_id("my-agent-v1")
        .capability("llm-inference")
        .capability("tool-use")
        .preference("model", "llama-3")
        .start()?;

    // Register a module
    let module_id = session.register_module(
        "web-search",
        "1.0.0",
        &["search(query: String) -> Results"],
        &["http-client"]
    )?;

    // Advance through phases (or let it auto-advance)
    let session = session.advance()?; // Phase 2
    let session = session.advance()?; // Phase 3
    let session = session.advance()?; // Phase 4
    let config = session.finalize()?; // Phase 5 → Config

    println!("{}", config.to_json()?);
    Ok(())
}
```

### Python

```python
import openconstruct

# Start onboarding
session = openconstruct.OnboardBuilder() \
    .agent_id("my-agent-v1") \
    .capability("llm-inference") \
    .capability("tool-use") \
    .preference("model", "llama-3") \
    .start()

# Register modules
session.register_module(
    name="web-search",
    version="1.0.0",
    provides=["search(query: str) -> Results"],
    requires=["http-client"]
)

# Advance through phases
while not session.is_complete():
    session.advance()

# Get generated config
config = session.config()
print(config.to_json())

# Or in one shot:
config = openconstruct.onboard(
    agent_id="my-agent-v1",
    capabilities=["llm-inference", "tool-use"],
    preferences={"model": "llama-3"},
    modules=[{
        "name": "web-search",
        "version": "1.0.0",
        "provides": ["search(query: str) -> Results"],
        "requires": ["http-client"]
    }]
)
```

### TypeScript / JavaScript

```typescript
import { OnboardBuilder } from "@openconstruct/core";
// or for thin client:
// import { OpenConstructClient } from "@openconstruct/client";

const session = await OnboardBuilder.create()
  .agentId("my-agent-v1")
  .capability("llm-inference")
  .capability("tool-use")
  .preference("model", "llama-3")
  .start();

await session.registerModule({
  name: "web-search",
  version: "1.0.0",
  provides: ["search(query: string) => Results"],
  requires: ["http-client"],
});

// Advance through phases
while (!session.isComplete()) {
  await session.advance();
}

const config = session.config();
console.log(config.toJSON());

// Thin client alternative:
const client = new OpenConstructClient("http://localhost:8080");
const config2 = await client.onboard({
  agentId: "my-agent-v1",
  capabilities: ["llm-inference", "tool-use"],
  preferences: { model: "llama-3" },
  modules: [{
    name: "web-search",
    version: "1.0.0",
    provides: ["search(query: string) => Results"],
    requires: ["http-client"],
  }],
});
```

### C (Universal FFI)

```c
#include "openconstruct.h"

int main() {
    // Create builder
    oc_builder_t* builder = oc_builder_new();
    oc_builder_set_agent_id(builder, "my-agent-v1");
    oc_builder_add_capability(builder, "llm-inference");
    oc_builder_add_capability(builder, "tool-use");
    oc_builder_set_preference(builder, "model", "llama-3");

    // Start session
    oc_session_t* session = oc_builder_start(builder);
    if (!session) {
        fprintf(stderr, "Onboarding failed\n");
        return 1;
    }

    // Register module
    oc_module_t mod = {
        .name = "web-search",
        .version = "1.0.0",
    };
    oc_module_add_provides(&mod, "search(query: String) -> Results");
    oc_module_add_requires(&mod, "http-client");
    oc_session_register_module(session, &mod);

    // Advance through phases
    while (!oc_session_is_complete(session)) {
        oc_result_t r = oc_session_advance(session);
        if (r != OC_OK) {
            fprintf(stderr, "Phase error: %s\n", oc_result_str(r));
            break;
        }
    }

    // Get config
    char* config_json = oc_session_config_json(session);
    printf("%s\n", config_json);

    // Cleanup
    oc_str_free(config_json);
    oc_session_free(session);
    oc_builder_free(builder);
    return 0;
}
```

---

## Build & Distribution

### Rust Core

```toml
[workspace]
members = [
    "crates/openconstruct-core",    # Core state machine + config
    "crates/openconstruct-ffi",     # C ABI (cdylib)
    "crates/openconstruct-server",  # HTTP/gRPC server for thin clients
    "crates/openconstruct-cli",     # CLI binary
    "crates/openconstruct-wasm",    # WASM build for TS/JS
    "bindings/python",              # PyO3 Python bindings
]
```

### Release Artifacts

| Artifact | Languages it enables |
|----------|---------------------|
| `libopenconstruct.so/.dylib/.dll` + `openconstruct.h` | C, C++, Zig, Go (CGo), Java (JNA), Swift, Fortran, Chapel, Ada |
| `openconstruct.wasm` | TypeScript/JS (browser), WebGPU |
| `openconstruct-cli` binary | Shell scripts, CI/CD, any language via subprocess |
| `openconstruct-server` Docker image | Thin clients in every language |
| PyPI `openconstruct` | Python, Mojo |
| npm `@openconstruct/core` | TypeScript/JS/Node |

### CI Matrix

```
Rust core:     cargo test + clippy + miri (FFI safety)
Python:        pytest + maturin build
TypeScript:    vitest + wasm-pack test
C header:      gcc/clang compilation test + valgrind
WASM:          wasm-pack test --node + --browser
All platforms: Linux x64, macOS arm64, Windows x64
```

---

## Summary

| Priority | Languages | Strategy | Effort |
|----------|-----------|----------|--------|
| **P0** | Rust, Python, TypeScript/JS, C | Native + FFI + WASM | Core work — ships first |
| **P1** | C++, Zig, Go, Java/Kotlin | C ABI wrappers + HTTP | Moderate — each ~1-2 weeks with C ABI |
| **P2** | Swift, Julia, Fortran, Chapel, Ada, Mojo, WGSL | C ABI / HTTP / Python compat | Community-driven |

**The C ABI is the keystone.** Once `openconstruct.h` + `libopenconstruct.so` exist, every P1 and most P2 languages get support almost for free. The thin HTTP client covers everything else.

**Ship order:**
1. Rust core + C ABI + HTTP server (enables everything)
2. Python bindings (biggest AI audience)
3. TypeScript/WASM (web agents)
4. P1 wrappers (each is a thin layer over C)
5. P2 as community PRs come in
