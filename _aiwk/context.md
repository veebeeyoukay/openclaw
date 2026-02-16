# AI Working Folder Context

**Last updated:** 2026-02-16

## Current Focus

Security audit for OpenClaw deployment on Mac Mini:
- **Phase B COMPLETE** — Codebase review (43 findings)
- **Phase C COMPLETE** — BlueBubbles-specific audit (2 critical mitigated, 3 remaining)
- **Phase D COMPLETE** — Config validation (PASS - config valid for deployment)
- iMessage-only communication via BlueBubbles
- Receiver/conduit agent role (limited "do anything")
- **Ready for Mac Mini installation**

## Multi-Agent Architecture

The Mac Mini will run multiple AI systems:

| System | Role | Port | Notes |
|--------|------|------|-------|
| OpenClaw | iMessage conduit | 18789 | Primary interface via BlueBubbles |
| AgentZero | Secondary tasks | TBD | Separate agent framework |
| Ollama | Local LLM | 11434 | Learning/memory layer |
| BlueBubbles | iMessage bridge | 1234 | Provides iMessage API |

**Design principles:**
- **Complete isolation** - No direct API calls between agents
- **Shared Ollama** - Common memory/context layer
- **File-based handoff** - Tasks via `~/agents/tasks/` with audit trail
- **Full traceability** - Every decision logged

**See:** `multi-agent-architecture.md` for full details

## Key Artifacts

| File | Purpose |
|------|---------|
| `setup-guide.md` | **Full installation guide** — Prerequisites, steps, verification |
| `networking-guide.md` | **Networking setup** — Bind modes, ports, firewall, remote access |
| `changelog.md` | **Version tracking** — Prevent updates from breaking setup |
| `security-audit-plan.md` | Complete security audit phases and OWASP mapping |
| `security-audit-report.md` | **Phase B+C findings** — 43 issues + BlueBubbles audit |
| `phase-c-findings.md` | **Phase C detailed** — BlueBubbles technical analysis |
| `phase-d-validation.md` | **Phase D results** — Config validation (PASS) |
| `multi-agent-architecture.md` | **Multi-agent design** — OpenClaw + AgentZero + Ollama |
| `imessage-only-setup.md` | How to strip other channels; plugins.allow; config |
| `limited-agent-config.md` | Tool policy for receiver/conduit; messaging profile |
| `openclaw-imessage-minimal.json5` | Hardened config template |

## OpenClaw Facts (verified)

- Channels: plugins in `extensions/*`, loaded via `listChannelPlugins()` from registry
- `plugins.allow` restricts which plugins load; when set, only listed plugins load
- Tool profiles: `minimal`, `messaging`, `coding`, `full` (see `src/agents/tool-policy.ts`)
- iMessage: BlueBubbles (recommended) or legacy imsg; both are extensions
- Security audit: `openclaw security audit`, `--deep`, `--fix`
