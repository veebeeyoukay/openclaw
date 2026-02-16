# AI Assistant Memory

Context and decisions for AI assistants working on this deployment.

---

## Project Identity

**Project:** Mac Mini Multi-Agent Deployment
**Owner:** vikasdevbhatia (Vikas)
**Repository:** https://github.com/veebeeyoukay/openclaw (fork)
**Working Directory:** `_aiwk/` (AI Working Knowledge)

---

## Key Facts

### Hardware
- **Target Machine:** Mac Mini
- **Operator Account:** `vikasplanbhatia` (separate iCloud for iMessage)
- **Admin Account:** `vbadmin` (NOT logged in during operation)
- **Security Model:** Single-account login required
- **External Storage:** Raycue M4 dock + 1TB NVMe → `/Volumes/AgentStorage`

### Storage Strategy
- **Internal:** Config, credentials (must survive dock disconnect)
- **NVMe (symlinked):** `~/agents/`, `~/.ollama/models/`, `~/.openclaw/logs/`
- **Rationale:** Graceful degradation if dock disconnects

### Services (All localhost only)
| Service | Port | Status |
|---------|------|--------|
| OpenClaw Gateway | 18789 | Planned |
| BlueBubbles | 1234 | Planned |
| Ollama | 11434 | Planned |
| AgentZero | TBD | Planned |

### Architecture Decisions
1. **No direct API calls** between OpenClaw and AgentZero
2. **File-based task handoff** via `~/agents/tasks/`
3. **Shared Ollama** for memory/context
4. **Full audit trail** - every decision logged

---

## Security Audit Status

| Phase | Status | Key Findings |
|-------|--------|--------------|
| A: Threat Model | ✅ | iMessage conduit, limited capabilities |
| B: Codebase | ✅ | 43 findings, all mitigated in config |
| C: BlueBubbles | ✅ | 2 critical → mitigated by single-account |
| D: Config | ✅ | PASS - validated against codebase |
| E: Runtime | ⏳ | Pending Mac Mini install |

---

## Critical Configuration

```json5
{
  // MUST be loopback only
  gateway: { bind: "loopback" },

  // MUST be disabled
  tools: { elevated: { enabled: false } },

  // MUST deny these
  tools: { deny: ["exec", "process", "read", "write", "edit", "apply_patch", ...] },

  // MUST be "none"
  agents: { defaults: { sandbox: { workspaceAccess: "none" } } },

  // MUST be "pairing"
  channels: { bluebubbles: { dmPolicy: "pairing" } }
}
```

---

## Directory Structure

```
~/agents/                    # Shared multi-agent workspace
├── tasks/
│   ├── pending/            # Tasks waiting for pickup
│   ├── in-progress/        # Currently processing
│   ├── completed/          # Done with results
│   └── failed/             # Errored tasks
├── audit/
│   ├── openclaw/           # OpenClaw logs
│   ├── agentzero/          # AgentZero logs
│   └── shared/             # Cross-agent events
├── decisions/              # Decision records
├── changes/                # Change control
└── memory/context/         # Shared context

~/.openclaw/                 # OpenClaw state
├── openclaw.json           # Main config (chmod 600!)
├── credentials/            # Channel creds
├── agents/                 # Agent sessions
└── logs/                   # Gateway logs
```

---

## Task Handoff Protocol

1. **OpenClaw** receives iMessage → creates `~/agents/tasks/pending/{id}.json`
2. **AgentZero** polls pending/ → moves to `in-progress/`
3. **AgentZero** completes → moves to `completed/` with result
4. **OpenClaw** polls completed/ → delivers via iMessage

All steps logged to `~/agents/audit/`

---

## Common Commands

```bash
# Check gateway status
openclaw gateway status

# Check channels
openclaw channels status --probe

# Security audit
openclaw security audit --deep

# View pairing requests
openclaw pairing list

# Approve a number
openclaw pairing approve bluebubbles CODE

# Check who's logged in (should only be vikasplanbhatia)
who
```

---

## Things NOT To Do

- ❌ Don't enable `elevated.enabled: true`
- ❌ Don't add `exec` or `process` to allow list
- ❌ Don't bind to `0.0.0.0` or `lan`
- ❌ Don't set `dmPolicy: "open"`
- ❌ Don't log in as `vbadmin` while OpenClaw runs
- ❌ Don't make direct API calls between agents

---

## Open Questions

1. AgentZero port number - TBD during install
2. Ollama model selection - llama3.2 planned, may upgrade
3. Polling interval for task handoff - start with 5-10 seconds
4. Inter-agent memory sharing pattern - via Ollama embeddings

---

## Session History

### 2026-02-16 - Initial Setup
- Created security audit (Phases B, C, D)
- Designed multi-agent architecture
- Documented task handoff protocol
- Created hardened config template
- Pushed to GitHub fork

### Next Session Goals
- SSH into Mac Mini and run installation
- Install OpenClaw, BlueBubbles, Ollama
- Create shared directory structure
- Run Phase E security audit
- Test iMessage flow end-to-end
