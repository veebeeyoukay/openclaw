# Limited Agent Configuration: Receiver / Conduit Role

**Date:** 2026-02-16
**Role:** Agent receives requests via iMessage and feeds information back from other agents. Limited "do anything" capability.

---

## Role Definition

| Behavior | Allowed | Not Allowed |
|----------|---------|-------------|
| Receive messages | ✓ iMessage DMs | — |
| Reply to sender | ✓ Via message tool (implicit reply) | Sending to arbitrary channels |
| List/fetch session history | ✓ From other agents | — |
| Send to other sessions | ✓ Forward tasks to subagents | — |
| Spawn subagents | ✓ For delegated work | — |
| Run shell commands | ✗ | exec, process |
| Read/write files | ✗ | read, write, edit, apply_patch |
| Browser control | ✗ | browser |
| Web search/fetch | ⚠ Optional, restricted | web_search, web_fetch |
| Gateway/cron/nodes | ✗ | gateway, cron, nodes |
| Cross-channel messaging | ✗ | message with arbitrary targets |

---

## Tool Profiles

OpenClaw provides predefined profiles:

| Profile | Tools | Use Case |
|---------|-------|----------|
| `minimal` | `session_status` only | Bare minimum |
| `messaging` | message, sessions_list, sessions_history, sessions_send, session_status | **Best fit for conduit** |
| `coding` | group:fs, group:runtime, group:sessions, group:memory, image | Dev/coding |
| `full` | No restriction | Default (dangerous for limited role) |

---

## Recommended Config: Messaging Profile + Deny Dangerous Tools

```json5
{
  tools: {
    profile: "messaging",
    deny: [
      "exec",
      "process",
      "read",
      "write",
      "edit",
      "apply_patch",
      "browser",
      "canvas",
      "nodes",
      "cron",
      "gateway",
      "web_search",
      "web_fetch",
      "image",
    ],
  },
}
```

This gives:
- `message` — Reply to requester (constrained to session target)
- `sessions_list` — List sessions
- `sessions_history` — Fetch history from other agents
- `sessions_send` — Send messages to other sessions
- `session_status` — Show usage/time/model

**Note:** `sessions_spawn` is in `group:sessions` but not in the `messaging` profile by default. To allow subagent spawning, add it:

```json5
{
  tools: {
    profile: "messaging",
    allow: ["sessions_spawn", "subagents", "agents_list"],
    deny: [ /* as above */ ],
  },
}
```

---

## Stricter: Minimal + Explicit Allow

If you want **only** session tools and no message tool (e.g. agent never sends; it only routes internally):

```json5
{
  tools: {
    profile: "minimal",
    allow: ["session_status", "sessions_list", "sessions_history", "sessions_send"],
    deny: ["*"],
  },
}
```

But then the agent cannot reply over iMessage. For a conduit that replies, keep `message` in the allowlist.

---

## Disable Message Tool for Outbound-Only Routing

If the agent should **only** receive and forward to subagents (never reply directly), you can disable the message tool. This is done at runtime when creating tools: `disableMessageTool: true`. Check `src/agents/openclaw-tools.ts` and routing logic. Typical use: internal-only agents that never post to channels.

For your case (agent **feeds info back to you**), the agent **must** have `message` so it can reply. Keep `message` enabled.

---

## Sandbox (Optional Hardening)

Even with tool deny, adding a sandbox reduces blast radius if a tool escapes or is misconfigured:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        scope: "agent",
        workspaceAccess: "none",
      },
    },
  },
}
```

With `workspaceAccess: "none"`, file tools (if accidentally enabled) have no workspace to touch.

---

## Session Visibility

For multi-user setups, restrict session visibility so the conduit cannot browse arbitrary sessions:

```json5
{
  tools: {
    sessions: { visibility: "tree" },
  },
}
```

- `self` — Only current session
- `tree` — Current + spawned subagent sessions (default)
- `agent` — All sessions for this agent
- `all` — Everything (avoid for limited role)

---

## Example: Full Limited Conduit Config

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    port: 18789,
    auth: { mode: "token", token: "YOUR_TOKEN" },
  },
  plugins: { allow: ["bluebubbles"] },
  channels: {
    web: { enabled: false },
    bluebubbles: {
      enabled: true,
      serverUrl: "http://127.0.0.1:1234",
      password: "YOUR_BB_PASSWORD",
      webhookPath: "/bluebubbles-webhook",
      dmPolicy: "pairing",
      allowFrom: [],
    },
  },
  tools: {
    profile: "messaging",
    allow: ["sessions_spawn", "subagents", "agents_list"],
    deny: [
      "exec", "process", "read", "write", "edit", "apply_patch",
      "browser", "canvas", "nodes", "cron", "gateway",
      "web_search", "web_fetch", "image",
    ],
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
}
```

---

## References

- `docs/tools/index.md` — Tool profiles, allow/deny
- `src/agents/tool-policy.ts` — TOOL_PROFILES, TOOL_GROUPS
- `docs/gateway/security/index.md` — Per-agent access profiles
- `docs/tools/multi-agent-sandbox-tools.md` — Sandbox + tool precedence
