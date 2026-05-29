# Governance

How decisions are made in the OpenConstruct project, who maintains what, and how the project evolves.

## Project Status

OpenConstruct is in **alpha**. APIs may change between releases. The project is actively developed and seeking contributors.

---

## Decision Making

### Consensus-Seeking

Most decisions are made through consensus on GitHub:

1. **Small changes** (bug fixes, docs, minor features): PR with one review.
2. **Medium changes** (new modules, API additions): RFC discussion on GitHub Issues, then PR with two reviews.
3. **Large changes** (architecture modifications, breaking API changes): Formal RFC in `rfc/` directory, community discussion period (minimum 7 days), then PR with two reviews.

### RFC Process

For significant changes:

1. Copy `rfc/0000-template.md` to `rfc/NNNN-short-name.md`
2. Fill in the template with motivation, design, alternatives, and open questions
3. Open a PR with the RFC document
4. Discuss. Iterate. Reach consensus.
5. Once approved, the RFC becomes the spec for implementation
6. Implementation PRs reference the RFC number

### When Consensus Fails

The project maintainer (currently the SuperInstance team) has final say on disputed decisions. This authority is used as a last resort when consensus cannot be reached after good-faith discussion.

---

## Maintainers

### Core Team

The core team is responsible for:

- Reviewing and merging PRs
- Release management
- Architecture decisions
- Security response
- Community health

| Area | Responsibility |
|---|---|
| Core framework | Shell, senses, correlator, fleet, mesh |
| Language bindings | C ABI, Python, TypeScript, Go, Rust |
| Edge (Jetson/ESP32) | Firmware, inference, MQTT bridge |
| Documentation | All `.md` files, API docs, examples |
| CI/CD | GitHub Actions, release automation |

### Becoming a Maintainer

Contributors who consistently provide high-quality PRs, reviews, and community support may be invited to join the maintainer team. There is no formal process — it emerges from sustained contribution.

---

## Release Process

### Versioning

OpenConstruct follows [Semantic Versioning](https://semver.org/):

- **Major (x.0.0):** Breaking API changes
- **Minor (0.x.0):** New features, backwards compatible
- **Patch (0.0.x):** Bug fixes, backwards compatible

During alpha (0.x.x), minor versions may include breaking changes. This will be noted in the changelog.

### Release Cadence

- **Patch releases:** As needed for bug fixes
- **Minor releases:** Biweekly (every two weeks)
- **Major releases:** When warranted by accumulated breaking changes

### Release Checklist

1. Update `CHANGELOG.md` with release date
2. Update version in `Cargo.toml`, `package.json`, `pyproject.toml`
3. Tag release: `git tag v0.x.0`
4. Push tag: `git push origin v0.x.0`
5. CI builds and publishes artifacts
6. GitHub Release created with changelog notes

---

## Community

### Communication Channels

- **GitHub Issues and PRs:** Primary channel for all project work
- **GitHub Discussions:** Questions, ideas, and general discussion
- **Discord:** Real-time chat (see README for invite link)

### Code of Conduct

- Be respectful and inclusive
- Focus on what is best for the community
- Show empathy toward other community members
- Provide constructive feedback

---

## Module Ownership

Each module in the ecosystem has a natural owner — the person or team most actively maintaining it. Ownership is earned through contribution, not assigned.

| Module Category | Examples | Ownership Model |
|---|---|---|
| Core | plato-shell, plato-vision, plato-correlator | Core team |
| Communication | plato-tick, shell-mesh, plato-a2a | Core team |
| Fleet | plato-fleet, plato-transport | Core team |
| Bindings | openconstruct-python, openconstruct-ts | Community |
| Edge | openconstruct-esp32, openconstruct-jetson | Core team + hardware contributors |
| Verification | openconstruct-mercury | Core team |
| Rendering | a2ui-render, mud2scummvm | Community |

Community-owned modules follow the same review process but may have different maintainers reviewing PRs.

---

*Questions about governance? Open a GitHub Discussion.*
