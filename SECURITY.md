# Security Policy

## Reporting a Vulnerability

**Do not report security vulnerabilities through public GitHub issues.**

Instead, report them through one of these channels:

- **GitHub Security Advisory:** [Report a vulnerability](https://github.com/SuperInstance/openconstruct-docs/security/advisories/new)
- **Email:** security@superinstance.dev

You should receive a response within 48 hours. If you don't, please follow up to ensure we received your message.

---

## What to Include

Please include:

1. **Description** of the vulnerability
2. **Affected versions** (or specific modules/files)
3. **Steps to reproduce** (if applicable)
4. **Potential impact** (what an attacker could do)
5. **Suggested fix** (if you have one)

---

## Disclosure Policy

- **Acknowledgment:** We will acknowledge receipt within 48 hours
- **Assessment:** We will assess the vulnerability and determine severity within 7 days
- **Fix:** We will work on a fix and coordinate disclosure with you
- **Credit:** We will credit you in the security advisory (unless you prefer to remain anonymous)

### Severity Levels

| Level | Description | Response Time |
|---|---|---|
| **Critical** | Remote code execution, credential theft, data exfiltration | < 24 hours |
| **High** | Privilege escalation, sandbox escape, authentication bypass | < 72 hours |
| **Medium** | Information disclosure, denial of service | < 7 days |
| **Low** | Minor information leaks, misconfigurations | Next release |

---

## Security Architecture

OpenConstruct inherits OpenShell's security model:

### Sandbox Isolation

- Agents run inside sandboxed environments with explicit resource limits
- Network egress is routed through a policy proxy
- Filesystem access is restricted to designated paths
- Process privileges are reduced

### Fleet Security

- ESP32 ↔ Jetson: MQTT over TLS-PSK with topic ACLs per `mote_id`
- Jetson ↔ Desktop/DGX: mTLS with SPIFFE workload identity
- Actuator commands: Signed by agent private key, verified by fleet CA
- Dial hardening under partition: automatic reduction to safe confidence levels

### Credential Handling

- Secrets are never committed to repositories
- Credentials are injected at runtime, not stored in configuration
- The policy proxy manages credential injection for outbound connections
- OCSF security logging tracks all credential access

### Known Security Properties

- Supervisor-initiated outbound connections only (no inbound firewall holes)
- Policy enforcement at every egress point
- Signed actuator commands prevent unauthorized physical actions
- Dial hardening prevents ungrounded inferences from triggering actuators during network partitions

---

## Supported Versions

| Version | Supported |
|---|---|
| 0.3.x | ✅ |
| 0.2.x | ✅ |
| 0.1.x | ⚠️ Security fixes only |
| < 0.1 | ❌ |

---

## Security Best Practices for Contributors

- Never commit secrets, API keys, or credentials
- Never run destructive operations (force push, hard reset, database drops) without explicit human confirmation
- Report suspicious behavior immediately
- Use `trash` over `rm` for file operations
- Scope changes to the issue at hand — no unrelated changes in security PRs
- All security-related code changes require two reviews

---

*This security policy is adapted from NVIDIA OpenShell's security model. For OpenShell-specific security concerns, see [NVIDIA/OpenShell SECURITY.md](https://github.com/NVIDIA/OpenShell/blob/main/SECURITY.md).*
