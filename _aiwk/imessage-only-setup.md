# Stripping Other Communication Methods for iMessage-Only OpenClaw

**Date:** 2026-02-16
**Target:** Mac Mini running OpenClaw as iMessage receiver agent

---

## Summary

OpenClaw loads channel plugins from `extensions/*` based on config and `plugins.allow`/`plugins.deny`. To run **iMessage-only**:

1. Use `plugins.allow` to load only iMessage-related plugins
2. Omit all other channel config sections (or set `enabled: false`)
3. Optionally remove/disable extension directories for defense-in-depth

---

## Method 1: Config-Only (Recommended)

No code changes. Use `plugins.allow` and only configure iMessage.

### Step 1: Choose iMessage Path

| Path | Plugin ID | Notes |
|------|-----------|-------|
| **BlueBubbles** (recommended) | `bluebubbles` | REST API, webhooks, richer features |
| **Legacy imsg** | `imessage` | JSON-RPC over stdio, CLI-based |

### Step 2: Set `plugins.allow`

```json5
{
  plugins: {
    allow: ["bluebubbles"],
    // Or for legacy imsg: allow: ["imessage"]
  },
}
```

When `plugins.allow` is set, **only** listed plugins load. All other channel extensions (telegram, discord, slack, signal, whatsapp, etc.) are excluded.

### Step 3: Omit Other Channel Config

Do **not** add `channels.telegram`, `channels.discord`, `channels.slack`, etc. If a channel has no config section, it won't start.

### Step 4: Explicitly Disable Web Channel (if present)

The web channel powers WhatsApp and other web-based auth. To disable:

```json5
{
  channels: {
    web: { enabled: false },
  },
}
```

---

## Method 2: Extensions Directory (Defense-in-Depth)

For maximum isolation, remove or rename channel extension folders so they cannot load.

**Extensions location:** `~/.openclaw/extensions/` (or `$OPENCLAW_STATE_DIR/extensions/`)

**Channel extensions to remove/disable** (when using BlueBubbles or imessage only):

```
telegram/
discord/
slack/
signal/
whatsapp/
msteams/
matrix/
irc/
googlechat/
mattermost/
feishu/
line/
nostr/
twitch/
tlon/
nextcloud-talk/
zalo/
zalouser/
```

**Keep:**
- `bluebubbles/` — if using BlueBubbles
- `imessage/` — if using legacy imsg
- Any provider auth plugins you need (e.g. `google-antigravity-auth` for models)
- `device-pair`, `phone-control`, `talk-voice` — bundled, often enabled by default

**Procedure:**
```bash
cd ~/.openclaw/extensions
mkdir -p _disabled
mv telegram discord slack signal whatsapp msteams matrix irc _disabled/
# Repeat for others as needed
```

Restore by moving back: `mv _disabled/telegram .`

---

## Method 3: Build-Time Exclusion (Advanced)

Not officially supported. The plugin loader discovers plugins under `extensions/` and configured `plugins.load.paths`. A custom build could exclude channel plugins, but this requires forking or patching the loader. **Prefer Method 1 or 2.**

---

## Verification

### Check loaded channels

```bash
openclaw channels status
openclaw channels status --probe
```

Expected: only `bluebubbles` (or `imessage`) and no errors for disabled channels.

### Check config

```bash
openclaw config get channels
openclaw config get plugins.allow
```

### Security audit

```bash
openclaw security audit
```

The audit flags "Extensions exist but plugins.allow is not set" when extensions exist without an explicit allowlist. With `plugins.allow: ["bluebubbles"]`, only bluebubbles loads and the finding is resolved.

---

## Minimal iMessage Config (BlueBubbles)

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    port: 18789,
    auth: { mode: "token", token: "YOUR_LONG_RANDOM_TOKEN" },
  },
  plugins: {
    allow: ["bluebubbles"],
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
}
```

## Minimal iMessage Config (Legacy imsg)

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    port: 18789,
    auth: { mode: "token", token: "YOUR_LONG_RANDOM_TOKEN" },
  },
  plugins: {
    allow: ["imessage"],
  },
  channels: {
    web: { enabled: false },
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/YOU/Library/Messages/chat.db",
      dmPolicy: "pairing",
      allowFrom: [],
    },
  },
}
```

---

## References

- `docs/channels/imessage.md`
- `docs/channels/bluebubbles.md`
- `docs/gateway/configuration-reference.md`
- `docs/tools/plugin.md` (plugins.allow)
- `src/plugins/config-state.ts` (allow/deny logic)
- `src/config/plugin-auto-enable.ts` (channel auto-enable)
