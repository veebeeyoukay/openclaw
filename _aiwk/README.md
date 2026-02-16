# Mac Mini Deployment: OpenClaw + AgentZero + Ollama

Personal deployment documentation for running a multi-agent AI system on Mac Mini.

## Overview

| Component | Purpose | Port |
|-----------|---------|------|
| **OpenClaw** | iMessage conduit agent | 18789 |
| **BlueBubbles** | iMessage API bridge | 1234 |
| **AgentZero** | Secondary task agent | TBD |
| **Ollama** | Local LLM (memory/learning) | 11434 |

## Architecture

```
Phone (iMessage)
       │
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ BlueBubbles  │────►│   OpenClaw   │     │  AgentZero   │
│   :1234      │     │    :18789    │     │    :TBD      │
└──────────────┘     └──────┬───────┘     └──────┬───────┘
                            │                    │
                            ▼                    │
                     ┌──────────────┐            │
                     │ ~/agents/    │◄───────────┘
                     │ (task handoff)│
                     └──────┬───────┘
                            │
                            ▼
                     ┌──────────────┐
                     │    Ollama    │
                     │   :11434     │
                     └──────────────┘
```

## Key Design Decisions

1. **Complete isolation** - No direct API calls between agents
2. **File-based task handoff** - Via `~/agents/tasks/` with full audit trail
3. **Shared memory** - Both agents use same Ollama instance
4. **Single-account login** - Security requires only `vikasplanbhatia` logged in

## Documentation Index

| Document | Description |
|----------|-------------|
| [setup-guide.md](setup-guide.md) | **Start here** - Full installation guide |
| [security-audit-report.md](security-audit-report.md) | Phase B+C security findings |
| [phase-c-findings.md](phase-c-findings.md) | BlueBubbles-specific analysis |
| [phase-d-validation.md](phase-d-validation.md) | Config validation results |
| [multi-agent-architecture.md](multi-agent-architecture.md) | Task handoff protocol |
| [openclaw-imessage-minimal.json5](openclaw-imessage-minimal.json5) | Hardened config template |
| [networking-guide.md](networking-guide.md) | Network configuration |
| [todos.md](todos.md) | Installation checklist |
| [CHANGELOG.md](CHANGELOG.md) | Version history |
| [MEMORY.md](MEMORY.md) | Session context for AI assistants |

## Quick Start

```bash
# 1. On Mac Mini, install prerequisites
brew install node@22 pnpm ollama

# 2. Install OpenClaw
pnpm add -g openclaw@latest

# 3. Copy hardened config
cp openclaw-imessage-minimal.json5 ~/.openclaw/openclaw.json

# 4. Generate secure token
openssl rand -hex 32
# Add to config: gateway.auth.token

# 5. Start services
ollama serve &
openclaw gateway run
```

See [setup-guide.md](setup-guide.md) for complete instructions.

## Security Status

| Phase | Status | Summary |
|-------|--------|---------|
| B: Codebase Audit | ✅ Complete | 43 findings, mitigations documented |
| C: BlueBubbles Audit | ✅ Complete | 2 critical mitigated by single-account login |
| D: Config Validation | ✅ Complete | PASS - config valid for deployment |
| E: Runtime Audit | ⏳ Pending | Run on Mac Mini after install |

## File Permissions

```bash
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/openclaw.json
chmod 700 ~/agents
```

## Maintenance

- Check [CHANGELOG.md](CHANGELOG.md) before updates
- Run `openclaw security audit --deep` after changes
- Review audit logs in `~/agents/audit/`

---

**Last Updated:** 2026-02-16
