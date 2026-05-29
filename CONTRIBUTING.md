# Contributing to OpenConstruct

Thank you for your interest in contributing. This document covers code style, testing requirements, the PR process, and the module template.

## Code of Conduct

Be respectful. Be constructive. Be excellent to each other.

## Quick Start

1. Fork the repository
2. Create a feature branch: `git checkout -b feat/my-feature`
3. Make your changes
4. Run tests and linting
5. Open a pull request

---

## Code Style

### Rust

- Follow `rustfmt` defaults: `cargo fmt --check`
- Clippy must pass: `cargo clippy -- -D warnings`
- Use `#[derive(Debug, Clone, Serialize, Deserialize)]` on all data types
- Error types implement `thiserror::Error`
- Public APIs are documented with `///` doc comments
- Every public function has a doc example

### Python

- Python 3.10+
- Use `uv` for all Python commands (`uv pip install`, `uv run`, `uv venv`)
- Follow PEP 8 with `ruff format`
- Type hints required on all public functions

### TypeScript

- TypeScript 5+
- Strict mode enabled
- `eslint` with recommended config
- No `any` types in public APIs

### C / Header-Only

- C11 standard
- `clang-format` with LLVM defaults
- All public functions prefixed with `oc_` (OpenConstruct)
- Header files are self-contained (include what you use)

### Markdown

- ATX-style headings (`#`, not `===`)
- Fenced code blocks with language annotation
- No trailing whitespace
- Max line length: 120 characters

---

## Testing Requirements

### Unit Tests

Every module must have unit tests covering:

- Happy path (valid inputs produce expected outputs)
- Error path (invalid inputs produce expected errors)
- Edge cases (empty inputs, boundary values, overflow)

For Rust: `#[cfg(test)] mod tests { ... }` in each source file.

### Integration Tests

Modules that interact with other modules need integration tests in a `tests/` directory:

- Test the trait contract: implement the trait, verify behavior
- Test with mock implementations of dependencies
- Test with real implementations when feasible

### Simulation Tests

OpenConstruct is simulation-first. Time-dependent components must:

- Accept an injected clock via `with_now_fn()` or equivalent
- Produce deterministic output for deterministic input
- Include at least one test using a simulated clock

### Test Naming

```
test_<module>_<scenario>_<expected_outcome>

// Examples:
test_correlator_temporal_window_fuses_cross_sense_shadows
test_vision_track_changes_detects_new_object
test_tick_board_ack_records_action_taken
```

### Running Tests

```bash
# Rust
cargo test                    # All tests
cargo test -p plato-vision    # Specific module
cargo test --doc               # Doc tests

# Python
uv run pytest

# TypeScript
npm test
```

---

## PR Process

### Before Opening a PR

1. **Rebase** on latest `main`: `git fetch origin && git rebase origin/main`
2. **Lint**: Run all linters (`cargo fmt && cargo clippy`, `ruff`, `eslint`)
3. **Test**: All tests pass (`cargo test`, `pytest`, `npm test`)
4. **Document**: Update relevant documentation
5. **Changelog**: Add entry to `CHANGELOG.md` under "Unreleased"

### PR Title Format

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

feat(vision): add privacy mask redaction
fix(correlator): handle empty temporal window
docs(readme): add Jupyter integration section
chore(deps): update tokio to 1.38
```

Types: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `ci`, `perf`

### PR Description Template

```markdown
## What
Brief description of the change.

## Why
Why this change is needed.

## How
How the change was implemented.

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing performed

## Documentation
- [ ] Docs updated (if applicable)
- [ ] CHANGELOG.md updated
```

### Review Process

1. One approving review required (two for architectural changes)
2. All CI checks must pass
3. No merge conflicts
4. Squash merge to `main`

### Commit Guidelines

- Use Conventional Commits format
- Scope is optional but encouraged
- Never reference AI tools in commit messages (no `Co-Authored-By: AI` lines)
- Keep commits atomic — one logical change per commit

---

## Module Template

Creating a new module? Follow this structure:

```
plato-<name>/
├── Cargo.toml          # Package manifest
├── README.md           # Module-specific docs
├── src/
│   ├── lib.rs          # Public API and re-exports
│   ├── types.rs        # Core types
│   └── <impl>.rs       # Implementation files
└── tests/
    ├── test_<name>.rs  # Integration tests
    └── fixtures/       # Test data
```

### Cargo.toml Template

```toml
[package]
name = "plato-<name>"
version = "0.1.0"
edition = "2021"
description = "<one-line description>"
license = "Apache-2.0"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "1"

[dev-dependencies]
# Test dependencies here

[lints]
workspace = true
```

### Required Trait Implementation

For sense modules:

```rust
#[async_trait]
pub trait SenseModule: Send + Sync {
    fn sense_type(&self) -> &str;
    async fn start(&mut self, config: SenseConfig) -> Result<ShadowStream, SenseError>;
    async fn stop(&mut self) -> Result<(), SenseError>;
    fn status(&self) -> SenseStatus;
}
```

For fleet/communication modules, implement the relevant transport or routing trait.

### Module Shadow Registration

Every module must include a `.openconstruct/module.json`:

```json
{
  "id": "<domain>.<name>",
  "name": "Human-Readable Name",
  "one_line": "What this module does, in one sentence.",
  "pick_if": "When you need this.",
  "skip_if": "When you don't.",
  "requires": [],
  "provides": [],
  "interfaces": ["cli", "api"]
}
```

---

## Reporting Issues

- **Bugs:** Open a GitHub issue with reproduction steps, expected vs actual behavior, and environment details.
- **Feature requests:** Open a GitHub issue with the use case and proposed solution.
- **Security vulnerabilities:** See [SECURITY.md](SECURITY.md). Do not file security issues publicly.

---

## Questions?

- Open a GitHub Discussion for general questions
- Check existing issues and PRs before filing new ones
- Read [ARCHITECTURE-DEEP.md](ARCHITECTURE-DEEP.md) for system internals

---

*Thank you for making OpenConstruct better.*
