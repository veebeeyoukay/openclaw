# OpenClaw Security Audit Plan: Mac Mini iMessage Receiver Agent

**Date:** 2026-02-16
**Scope:** Security audit of OpenClaw codebase for installation on Mac Mini as iMessage-only receiver agent with limited "do anything" capabilities (conduit role: receive requests, feed info from other agents back to user).

---

## 1. Audit Objectives

| Objective | Description |
|-----------|-------------|
| **Pre-install audit** | Assess codebase security before deploying to Mac Mini |
| **iMessage-only hardening** | Document how to strip other communication channels |
| **Limited agent scope** | Configure agent as receiver/conduit with minimal tool blast radius |
| **OWASP alignment** | Map findings to OWASP Top 10 (Web, LLM, Agentic) |

---

## 2. Built-in Security Tools (Baseline)

OpenClaw ships `openclaw security audit`:

```bash
openclaw security audit       # Standard checks
openclaw security audit --deep # Includes live gateway probe
openclaw security audit --fix # Apply safe guardrails (perms, groupPolicy)
```

**What it checks:**
- Inbound access (DM policies, group policies, allowlists)
- Tool blast radius (elevated tools + open rooms)
- Network exposure (Gateway bind/auth, Tailscale, weak tokens)
- Browser control exposure
- Local disk hygiene (permissions, symlinks)
- Plugins (extensions without explicit allowlist)
- Policy drift (sandbox vs config, tool overrides)

---

## 3. Complete Security Audit Phases

### Phase A: Pre-Audit Preparation
- [ ] Clone repo at specific commit/tag for reproducibility
- [ ] Run `pnpm install`; ensure no modified `node_modules`
- [ ] Set up `~/.openclaw` state dir with correct perms (700)
- [ ] Create minimal config template for iMessage-only

### Phase B: Codebase Security Review

**B1. Authentication & Credentials (OWASP A07, LLM02)**
- [ ] `src/gateway/auth.ts` — Gateway auth modes, token handling
- [ ] `src/agents/auth-profiles/store.ts` — OAuth/API key storage
- [ ] `~/.openclaw/credentials/**` — Channel creds, pairing allowlists
- [ ] `~/.openclaw/agents/*/agent/auth-profiles.json` — Model auth
- [ ] Check for secrets in config (`openclaw.json` tokens)

**B2. Channel Surface (OWASP A01, LLM06)**
- [ ] `src/channels/plugins/` — Channel loading, routing
- [ ] `extensions/imessage/`, `extensions/bluebubbles/` — iMessage paths
- [ ] `plugins.allow` / `plugins.deny` — Explicit plugin allowlist
- [ ] DM/group policies per channel (`dmPolicy`, `groupPolicy`)

**B3. Agent Tool Policy (OWASP LLM06 Excessive Agency)**
- [ ] `src/agents/tool-policy.ts` — `minimal`, `messaging`, `coding`, `full` profiles
- [ ] `src/agents/openclaw-tools.ts` — Tool creation, `disableMessageTool`
- [ ] `tools.profile`, `tools.allow`, `tools.deny` resolution
- [ ] Per-agent overrides (`agents.list[].tools`)

**B4. Execution & Sandbox (OWASP A03 Injection, LLM05)**
- [ ] `tools.exec` — Shell command execution, `host` (sandbox | gateway | node)
- [ ] `tools.elevated` — Host escape hatch
- [ ] Sandbox config (`agents.defaults.sandbox`, `workspaceAccess`)
- [ ] `nodes.run` — Remote code execution on paired nodes

**B5. Network & Gateway Exposure**
- [ ] `gateway.bind` (loopback | lan | tailnet | custom)
- [ ] `gateway.auth` (token | password | trusted-proxy)
- [ ] Tailscale Serve/Funnel exposure
- [ ] HTTP endpoints (`/tools/invoke`, Control UI, webhooks)

**B6. Input/Output Handling (OWASP LLM01, LLM05)**
- [ ] Prompt construction (`src/agents/system-prompt.ts`)
- [ ] User input concatenation into prompts
- [ ] Tool output sanitization before delivery
- [ ] Logging redaction (`logging.redactSensitive`)

**B7. Plugins & Extensions**
- [ ] `plugins.allow` — Only load explicitly trusted plugins
- [ ] `extensions/*` — Installed plugin list, npm lifecycle scripts
- [ ] No `workspace:*` in plugin deps (install breaks)

### Phase C: iMessage-Specific Checks
- [ ] BlueBubbles vs legacy imsg — chosen path and config
- [ ] Webhook auth (BlueBubbles `webhookPath` + password)
- [ ] `channels.imessage` / `channels.bluebubbles` — `dmPolicy`, `allowFrom`
- [ ] Pairing flow for DMs
- [ ] Full Disk Access + Automation (macOS) — process context

### Phase D: Limited-Agent Configuration Validation
- [ ] `tools.profile: "minimal"` or custom allowlist
- [ ] No `exec`, `process`, `browser`, `gateway`, `cron` unless required
- [ ] `sessions_list`, `sessions_history`, `sessions_send`, `session_status` for conduit role
- [ ] `message` tool — restrict to reply-only (no cross-channel send)

### Phase E: Post-Audit
- [ ] Run `openclaw security audit --fix`
- [ ] Document findings with severity (Critical/High/Medium/Low/Info)
- [ ] Create hardened config template
- [ ] Establish re-audit cadence (e.g., before each release)

---

## 4. OWASP Mapping (Quick Reference)

| Risk | OWASP ID | OpenClaw Surface |
|------|----------|------------------|
| Prompt injection | LLM01 | User messages, web_fetch, browser content |
| Sensitive disclosure | LLM02 | Logs, transcripts, auth-profiles.json |
| Excessive agency | LLM06 | Tool policy, exec, gateway, nodes |
| System prompt leakage | LLM07 | Error messages, crafted prompts |
| Broken access control | A01 | DM/group policies, pairing, allowlists |
| Cryptographic failures | A02 | Token storage, TLS |
| Injection | A03 | exec, apply_patch, tool params |
| Misconfiguration | A05 | gateway.bind, plugins.allow, tool deny |

---

## 5. Deliverables

1. **Security Audit Report** (`_aiwk/security-audit-report.md`) — Findings + remediation
2. **iMessage-Only Config Guide** (`_aiwk/imessage-only-setup.md`) — How to strip channels
3. **Limited Agent Config** (`_aiwk/limited-agent-config.md`) — Receiver/conduit role
4. **Hardened Config Template** (`_aiwk/openclaw-imessage-minimal.json5`) — Copy-paste baseline

---

## 6. References

- [OpenClaw Security Docs](https://docs.openclaw.ai/gateway/security)
- [Formal Verification (Security Models)](https://docs.openclaw.ai/security/formal-verification)
- [Tools (allow/deny/profiles)](https://docs.openclaw.ai/tools)
- [iMessage Channel](https://docs.openclaw.ai/channels/imessage)
- [BlueBubbles Channel](https://docs.openclaw.ai/channels/bluebubbles)
- [Pairing](https://docs.openclaw.ai/channels/pairing)
