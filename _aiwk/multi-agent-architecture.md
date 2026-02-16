# Multi-Agent Architecture: OpenClaw + AgentZero + Ollama

**Date:** 2026-02-16
**Platform:** Mac Mini (vikasplanbhatia account)

---

## Design Principles

1. **Complete isolation** - No direct API calls between agents
2. **Shared memory** - Ollama provides common context/learning layer
3. **File-based handoff** - Tasks passed via filesystem with audit trail
4. **Full traceability** - Every decision logged and auditable

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Mac Mini (vikasplanbhatia)                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐                              ┌──────────────┐         │
│  │  BlueBubbles │                              │  AgentZero   │         │
│  │    :1234     │                              │    :XXXX     │         │
│  │  (iMessage)  │                              │ (Secondary)  │         │
│  └──────┬───────┘                              └──────┬───────┘         │
│         │                                             │                 │
│         ▼                                             │                 │
│  ┌──────────────┐                                     │                 │
│  │   OpenClaw   │                                     │                 │
│  │    :18789    │                                     │                 │
│  │  (Conduit)   │                                     │                 │
│  └──────┬───────┘                                     │                 │
│         │                                             │                 │
│         │    ┌────────────────────────────────┐       │                 │
│         │    │      Shared Filesystem         │       │                 │
│         │    ├────────────────────────────────┤       │                 │
│         ├───►│  ~/agents/tasks/pending/       │◄──────┤                 │
│         │    │  ~/agents/tasks/in-progress/   │       │                 │
│         │    │  ~/agents/tasks/completed/     │       │                 │
│         │    │  ~/agents/audit/               │       │                 │
│         │    │  ~/agents/decisions/           │       │                 │
│         │    │  ~/agents/changes/             │       │                 │
│         │    └────────────────────────────────┘       │                 │
│         │                                             │                 │
│         │         ┌──────────────┐                    │                 │
│         └────────►│    Ollama    │◄───────────────────┘                 │
│                   │   :11434     │                                      │
│                   │ (Shared LLM) │                                      │
│                   └──────────────┘                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Shared Directory Structure

```
~/agents/
├── tasks/
│   ├── pending/           # Tasks waiting to be picked up
│   │   └── {task-id}.json
│   ├── in-progress/       # Tasks currently being worked
│   │   └── {task-id}.json
│   ├── completed/         # Finished tasks with results
│   │   └── {task-id}.json
│   └── failed/            # Tasks that errored
│       └── {task-id}.json
├── audit/
│   ├── openclaw/          # OpenClaw audit logs
│   │   └── {date}.jsonl
│   ├── agentzero/         # AgentZero audit logs
│   │   └── {date}.jsonl
│   └── shared/            # Cross-agent interactions
│       └── {date}.jsonl
├── decisions/
│   └── {decision-id}.md   # Documented decisions with rationale
├── changes/
│   └── {change-id}.md     # Change control records
└── memory/
    └── context/           # Shared context files (optional)
```

---

## Task Handoff Protocol

### Task File Schema

```json
{
  "id": "task-2026-02-16-001",
  "version": 1,
  "created": "2026-02-16T10:30:00Z",
  "updated": "2026-02-16T10:30:00Z",

  "source": {
    "agent": "openclaw",
    "session": "agent:pi:bluebubbles:dm:+1234567890",
    "user": "+1234567890"
  },

  "task": {
    "type": "research|action|analysis|other",
    "priority": "low|normal|high|urgent",
    "title": "Brief task description",
    "description": "Full task details...",
    "context": {
      "conversation_summary": "...",
      "relevant_files": [],
      "constraints": []
    }
  },

  "assignment": {
    "target_agent": "agentzero|any",
    "picked_up_by": null,
    "picked_up_at": null
  },

  "result": {
    "status": "pending|in-progress|completed|failed",
    "output": null,
    "artifacts": [],
    "error": null
  },

  "audit": {
    "state_changes": [
      {
        "from": "pending",
        "to": "in-progress",
        "by": "agentzero",
        "at": "2026-02-16T10:35:00Z",
        "reason": "Picked up for processing"
      }
    ]
  }
}
```

### Handoff Flow

```
1. OpenClaw receives request via iMessage
   │
2. OpenClaw creates task file in ~/agents/tasks/pending/
   │ - Generates unique task ID
   │ - Logs to ~/agents/audit/openclaw/
   │
3. AgentZero polls ~/agents/tasks/pending/
   │ - Picks up task matching its capabilities
   │ - Moves to ~/agents/tasks/in-progress/
   │ - Logs pickup to ~/agents/audit/agentzero/
   │
4. AgentZero processes task
   │ - May query Ollama for context/memory
   │ - Updates task file with progress
   │
5. AgentZero completes task
   │ - Moves to ~/agents/tasks/completed/
   │ - Writes result to task file
   │ - Logs completion to audit
   │
6. OpenClaw polls ~/agents/tasks/completed/
   │ - Picks up results for tasks it created
   │ - Delivers response via iMessage
   │ - Archives or cleans up task file
```

---

## Ollama Shared Memory

### Memory Patterns

1. **Embedding Store** - Both agents can store/retrieve embeddings
2. **Conversation Context** - Shared understanding of ongoing work
3. **Learning Log** - Accumulated knowledge from interactions

### Ollama Configuration

```bash
# Ollama serves both agents
OLLAMA_HOST=127.0.0.1:11434

# Recommended models
ollama pull llama3.2          # Fast, for real-time tasks
ollama pull nomic-embed-text  # For embeddings/memory
```

### Memory Access Pattern

```
┌──────────────┐     ┌──────────────┐
│   OpenClaw   │     │  AgentZero   │
└──────┬───────┘     └──────┬───────┘
       │                    │
       │  POST /api/embed   │  POST /api/embed
       │  POST /api/chat    │  POST /api/chat
       │                    │
       └────────┬───────────┘
                │
                ▼
         ┌──────────────┐
         │    Ollama    │
         │   :11434     │
         │              │
         │ ┌──────────┐ │
         │ │  Models  │ │
         │ │ llama3.2 │ │
         │ │ nomic-   │ │
         │ │ embed    │ │
         │ └──────────┘ │
         └──────────────┘
```

---

## Audit Log Format

### JSONL Entry Schema

```json
{
  "ts": "2026-02-16T10:30:00.000Z",
  "agent": "openclaw",
  "event": "task_created|task_picked|task_completed|decision|error",
  "task_id": "task-2026-02-16-001",
  "session": "agent:pi:bluebubbles:dm:+1234567890",
  "action": "Description of what happened",
  "details": {},
  "checksum": "sha256:..."
}
```

### Audit Categories

| Category | Path | Contents |
|----------|------|----------|
| Agent-specific | `~/agents/audit/{agent}/` | All actions by that agent |
| Cross-agent | `~/agents/audit/shared/` | Handoffs, pickups, deliveries |
| Decisions | `~/agents/decisions/` | Significant choices with rationale |
| Changes | `~/agents/changes/` | System/config modifications |

---

## Decision Log Format

```markdown
# Decision: {decision-id}

**Date:** 2026-02-16
**Agent:** openclaw
**Task:** task-2026-02-16-001

## Context
What situation prompted this decision?

## Options Considered
1. Option A - pros/cons
2. Option B - pros/cons

## Decision
What was decided and why?

## Outcome
(Filled in later) What happened as a result?
```

---

## Change Control Format

```markdown
# Change: {change-id}

**Date:** 2026-02-16
**Agent:** agentzero
**Type:** config|code|data|system

## Description
What changed?

## Rationale
Why was this change made?

## Before
State before the change.

## After
State after the change.

## Rollback
How to undo this change if needed.

## Verification
How to verify the change worked.
```

---

## Security Considerations

### File Permissions

```bash
# Shared directories
chmod 700 ~/agents
chmod 700 ~/agents/tasks
chmod 700 ~/agents/audit
chmod 700 ~/agents/decisions
chmod 700 ~/agents/changes

# Task files readable/writable by owner only
# (Both agents run as vikasplanbhatia, so this works)
```

### Isolation Guarantees

| Boundary | Enforced By | Notes |
|----------|-------------|-------|
| No direct API calls | Design | Agents never call each other's APIs |
| Shared filesystem | OS permissions | Both agents same user |
| Ollama access | localhost only | No remote access |
| Task validation | Schema | Agents validate task files before processing |

### Tampering Detection

- Each audit entry includes SHA256 checksum
- Log rotation preserves historical entries
- Consider append-only audit logs (chattr +a on Linux, not available on macOS)

---

## Setup Checklist

- [ ] Create directory structure: `mkdir -p ~/agents/{tasks/{pending,in-progress,completed,failed},audit/{openclaw,agentzero,shared},decisions,changes,memory/context}`
- [ ] Set permissions: `chmod -R 700 ~/agents`
- [ ] Configure OpenClaw task handoff (see implementation notes)
- [ ] Configure AgentZero task polling
- [ ] Set up Ollama with required models
- [ ] Test handoff flow end-to-end
- [ ] Verify audit logging works
- [ ] Document polling intervals

---

## Implementation Notes

### OpenClaw Side

To enable task handoff in OpenClaw, you'll need:

1. A skill or tool that writes to `~/agents/tasks/pending/`
2. A watcher that monitors `~/agents/tasks/completed/` for results
3. Audit logging to `~/agents/audit/openclaw/`

### AgentZero Side

AgentZero needs:

1. File watcher on `~/agents/tasks/pending/`
2. Task processing logic
3. Result writing to completed/
4. Audit logging

### Polling vs Watch

| Method | Pros | Cons |
|--------|------|------|
| Polling | Simple, reliable | Latency, CPU overhead |
| FSEvents | Instant, efficient | More complex, edge cases |

**Recommendation:** Start with polling (5-10 second intervals), optimize later if needed.
