# Phase D: Limited-Agent Config Validation

**Date:** 2026-02-16
**Scope:** Validate hardened config against codebase behavior
**Status:** COMPLETE - Config validated with minor recommendations

---

## Executive Summary

The hardened config template (`openclaw-imessage-minimal.json5`) was validated against the OpenClaw codebase. The configuration correctly restricts the agent to a messaging-only role with appropriate security controls.

| Category | Status | Notes |
|----------|--------|-------|
| Tool Profile | ✓ VALID | `messaging` profile correctly scoped |
| Allow List | ✓ VALID | Additions align with conduit role |
| Deny List | ⚠ REVIEW | 2 minor gaps identified |
| Elevated Mode | ✓ VALID | Correctly disabled |
| Sandbox | ✓ VALID | workspaceAccess: "none" |
| DM Policy | ✓ VALID | pairing requires approval |
| Plugin Allowlist | ✓ VALID | Only bluebubbles loads |

---

## Tool Profile Analysis

### "messaging" Profile Definition

From `src/agents/tool-policy.ts:72-80`:

```typescript
messaging: {
  allow: [
    "group:messaging",   // → ["message"]
    "sessions_list",
    "sessions_history",
    "sessions_send",
    "session_status",
  ],
},
```

**Group Expansion:**
- `group:messaging` → `["message"]`

**Profile allows:** `message`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status`

### Config Additions

The hardened config adds to allow:
```json5
allow: ["sessions_spawn", "subagents", "agents_list"]
```

**Validation:** These tools are appropriate for a conduit that needs to:
- `sessions_spawn`: Create subagent sessions for tasks
- `subagents`: Monitor/manage spawned subagents
- `agents_list`: View available agents

---

## Deny List Validation

### Configured Deny List

```json5
deny: [
  "exec", "process", "read", "write", "edit", "apply_patch",
  "browser", "canvas", "nodes", "cron", "gateway",
  "web_search", "web_fetch", "image",
]
```

### Complete Core Tool Inventory

| Tool | In Deny | Risk | Action |
|------|---------|------|--------|
| `exec` | ✓ | CRITICAL | Blocked |
| `process` | ✓ | HIGH | Blocked |
| `read` | ✓ | HIGH | Blocked |
| `write` | ✓ | HIGH | Blocked |
| `edit` | ✓ | HIGH | Blocked |
| `apply_patch` | ✓ | HIGH | Blocked |
| `browser` | ✓ | MEDIUM | Blocked |
| `canvas` | ✓ | LOW | Blocked |
| `nodes` | ✓ | MEDIUM | Blocked |
| `cron` | ✓ | MEDIUM | Blocked |
| `gateway` | ✓ | HIGH | Blocked |
| `web_search` | ✓ | LOW | Blocked |
| `web_fetch` | ✓ | MEDIUM | Blocked |
| `image` | ✓ | LOW | Blocked |
| `tts` | ✗ | LOW | See below |
| `memory_search` | ✗ | LOW | See below |
| `memory_get` | ✗ | LOW | See below |
| `message` | Allowed | - | Expected |
| `sessions_*` | Allowed | - | Expected |
| `subagents` | Allowed | - | Expected |
| `agents_list` | Allowed | - | Expected |

### Gap Analysis

#### 1. `tts` Tool (LOW risk)

**Description:** Text-to-speech conversion
**File:** `src/agents/tools/tts-tool.ts`

```typescript
name: "tts",
description: "Convert text to speech..."
```

**Risk Assessment:**
- Uses external TTS service (API key required)
- Generates audio files to temp directory
- Does not expose file system or exec capabilities

**Recommendation:** OPTIONAL - Add to deny list if TTS not needed:
```json5
deny: [..., "tts"]
```

#### 2. `memory_search` / `memory_get` (LOW risk)

**Description:** Search/retrieve from agent memory store
**Group:** `group:memory`

**Risk Assessment:**
- Read-only access to memory store
- Useful for context retrieval
- No write/exec capabilities

**Recommendation:** KEEP ALLOWED - Useful for conduit context

---

## Policy Resolution Verification

### Resolution Order (from `src/agents/pi-tools.policy.ts`)

1. Agent-specific override (`agents.list[].tools`)
2. Global policy (`tools.*`)
3. Provider policy (`tools.byProvider.*`)
4. Profile (`tools.profile`)
5. Group policy (`channels.*.groups.*.tools`)
6. Subagent policy (automatic for spawned agents)
7. Sandbox policy

**Key Rule:** Deny ALWAYS wins at each step.

### Verification Test Cases

| Tool | Profile Allow | Config Allow | Config Deny | Result |
|------|---------------|--------------|-------------|--------|
| `message` | ✓ | - | - | ALLOWED |
| `sessions_spawn` | ✗ | ✓ | - | ALLOWED |
| `exec` | ✗ | - | ✓ | DENIED |
| `read` | ✗ | - | ✓ | DENIED |
| `browser` | ✗ | - | ✓ | DENIED |

---

## Elevated Mode Verification

### Config Setting

```json5
elevated: { enabled: false }
```

### Code Behavior (from `src/agents/bash-tools.exec.ts:268-277`)

```typescript
if (elevatedRequested) {
  if (!elevatedDefaults?.enabled || !elevatedDefaults.allowed) {
    if (!elevatedDefaults?.enabled) {
      gates.push("enabled (tools.elevated.enabled...)");
    }
    // Blocks elevated execution
  }
}
```

**Validation:** With `enabled: false`, ALL elevated requests are blocked regardless of other settings.

---

## Sandbox Configuration Verification

### Config Setting

```json5
agents: {
  defaults: {
    sandbox: {
      mode: "all",
      scope: "agent",
      workspaceAccess: "none",
    },
  },
},
```

### Code Behavior (from `src/agents/sandbox/types.ts`)

```typescript
type SandboxWorkspaceAccess = "none" | "ro" | "rw";
```

**Validation:**
- `mode: "all"` - Sandbox enabled for all agents
- `scope: "agent"` - Per-agent isolation
- `workspaceAccess: "none"` - No filesystem access

---

## DM Policy Verification

### Config Setting

```json5
dmPolicy: "pairing",
allowFrom: [],
```

### Code Behavior (from `src/config/types.base.ts`)

```typescript
type DmPolicy = "pairing" | "allowlist" | "open" | "disabled";
```

**Validation:**
- `pairing` - Requires explicit approval via pairing flow
- Empty `allowFrom` - No pre-approved senders
- New messages trigger pairing code requirement

---

## Plugin Allowlist Verification

### Config Setting

```json5
plugins: {
  allow: ["bluebubbles"],
  deny: ["device-pair", "phone-control", "talk-voice"],
},
```

### Behavior

1. Only `bluebubbles` plugin loads
2. Bundled plugins explicitly denied (CHAN-3 mitigation)
3. Other channels (telegram, discord, slack, etc.) do not load

---

## Subagent Policy Implications

From `src/agents/pi-tools.policy.ts:44-58`:

```typescript
const SUBAGENT_TOOL_DENY_ALWAYS = [
  "gateway", "agents_list", "whatsapp_login",
  "session_status", "cron",
  "memory_search", "memory_get",
  "sessions_send",
];
```

**Impact:** Subagents spawned by the conduit agent automatically have additional restrictions:
- Cannot access gateway
- Cannot manage agents
- Cannot send sessions directly (use announce chain)
- Cannot access memory (info passed in spawn prompt)

---

## Recommendations

### Required Changes: NONE

The current config is valid for the conduit role.

### Optional Enhancements

1. **Add `tts` to deny list** (if not needed):
   ```json5
   deny: [..., "tts"]
   ```

2. **Consider `memory_*` tools** - Currently allowed implicitly. If the conduit doesn't need memory:
   ```json5
   deny: [..., "memory_search", "memory_get"]
   ```

3. **Document provider tools** - If using MCP servers or plugins with custom tools, review their exposure.

---

## Final Validated Config

```json5
// ~/.openclaw/openclaw.json - Phase D Validated
{
  gateway: {
    mode: "local",
    bind: "loopback",
    port: 18789,
    auth: { mode: "token", token: "YOUR_64_CHAR_HEX_TOKEN" },
  },
  plugins: {
    allow: ["bluebubbles"],
    deny: ["device-pair", "phone-control", "talk-voice"],
  },
  channels: {
    web: { enabled: false },
    bluebubbles: {
      enabled: true,
      serverUrl: "http://127.0.0.1:1234",
      password: "YOUR_BLUEBUBBLES_PASSWORD",
      webhookPath: "/bluebubbles-webhook",
      dmPolicy: "pairing",
      allowFrom: [],
    },
  },
  tools: {
    profile: "messaging",
    allow: ["sessions_spawn", "subagents", "agents_list"],
    deny: [
      "exec", "process", "apply_patch",
      "read", "write", "edit",
      "browser", "canvas", "nodes", "cron", "gateway",
      "web_search", "web_fetch", "image",
      // Optional: "tts", "memory_search", "memory_get"
    ],
    elevated: { enabled: false },
    sessions: { visibility: "tree" },
  },
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        scope: "agent",
        workspaceAccess: "none",
      },
    },
  },
  logging: {
    redactSensitive: "tools",
  },
}
```

---

## Validation Status: PASS

| Check | Result |
|-------|--------|
| Tool profile correct | ✓ |
| Deny list comprehensive | ✓ |
| Elevated mode disabled | ✓ |
| Sandbox isolated | ✓ |
| DM requires pairing | ✓ |
| Plugins restricted | ✓ |
| No critical gaps | ✓ |

**Conclusion:** The hardened config is valid for deployment as an iMessage conduit agent.
