# OpenClaw Mac Mini Setup Guide

**Date:** 2026-02-16
**Target:** Mac Mini as iMessage-only receiver/conduit agent
**Role:** Receive requests via iMessage, feed info from other agents back to user

---

## Prerequisites

### Hardware/OS
- Mac Mini (Apple Silicon or Intel)
- macOS 14+ (Sonoma) recommended
- Signed into iCloud with iMessage enabled

### Software
- **Node.js 22+** (required)
  ```bash
  # Check version
  node --version  # Must be v22.x or higher

  # Install via Homebrew if needed
  brew install node@22
  ```

- **pnpm** (recommended) or npm
  ```bash
  # Install pnpm
  npm install -g pnpm
  ```

- **BlueBubbles Server** (for iMessage)
  - Download from: https://bluebubbles.app/downloads/
  - Requires Full Disk Access + Automation permissions

### External Storage (Raycue M4 Dock + NVMe)

**Hardware:**
- Raycue M4 dock (40Gbps Thunderbolt)
- 1TB NVMe drive

**Strategy:** Selective symlinks, NOT full home directory on external.

**Rationale:** If dock disconnects, system still boots and OpenClaw starts. Only high-I/O directories on NVMe.

---

## Step 0: Configure External Storage

**Mount the NVMe:**

```bash
# Format if needed (APFS recommended)
diskutil list  # Find the NVMe disk
diskutil apfs createContainer /dev/diskX
diskutil apfs addVolume diskX APFS "AgentStorage"

# Verify mount
ls /Volumes/AgentStorage
```

**Create directory structure on NVMe:**

```bash
# Create agent directories
mkdir -p /Volumes/AgentStorage/agents/{tasks/{pending,in-progress,completed,failed},audit/{openclaw,agentzero,shared},decisions,changes,memory/context}

# Create ollama models directory
mkdir -p /Volumes/AgentStorage/ollama/models

# Create openclaw logs directory
mkdir -p /Volumes/AgentStorage/openclaw/logs

# Set permissions
chmod -R 700 /Volumes/AgentStorage/agents
chmod -R 700 /Volumes/AgentStorage/ollama
chmod -R 700 /Volumes/AgentStorage/openclaw
```

**Create symlinks from home:**

```bash
# Agents directory (task handoff, audit logs)
ln -s /Volumes/AgentStorage/agents ~/agents

# Ollama models (large, re-downloadable)
mkdir -p ~/.ollama
ln -s /Volumes/AgentStorage/ollama/models ~/.ollama/models

# OpenClaw logs (after Step 2)
# Done after ~/.openclaw directory exists
```

**What goes where:**

| Path | Location | Rationale |
|------|----------|-----------|
| `~/.openclaw/openclaw.json` | Internal | Config must survive dock disconnect |
| `~/.openclaw/credentials/` | Internal | Critical, small |
| `~/.openclaw/logs/` | NVMe (symlink) | Large, recreatable |
| `~/agents/` | NVMe (symlink) | High I/O, task handoff |
| `~/.ollama/models/` | NVMe (symlink) | Large models (GBs) |

**Auto-mount on boot:**

The NVMe should auto-mount if formatted as APFS. Verify after reboot:

```bash
# Check mount
mount | grep AgentStorage

# If not auto-mounting, add to /etc/fstab or use Login Items
```

**Graceful degradation:**

If dock disconnects:
- System boots normally
- OpenClaw starts (config on internal)
- Error: "~/agents directory unavailable" (logged)
- Reconnect dock → services resume

---

## Step 1: Install OpenClaw

```bash
# Global install (recommended for production)
pnpm add -g openclaw@latest
# or
npm install -g openclaw@latest

# Verify installation
openclaw --version
```

**Pin version for stability:**
```bash
# Check current version
npm view openclaw version

# Install specific version
pnpm add -g openclaw@2026.2.16
```

---

## Step 2: Create State Directory

```bash
mkdir -p ~/.openclaw
chmod 700 ~/.openclaw

# Symlink logs to NVMe (if using external storage)
ln -s /Volumes/AgentStorage/openclaw/logs ~/.openclaw/logs
```

---

## Step 3: Install Hardened Config

Copy the hardened config template:

```bash
# From this repo (if cloned)
cp _aiwk/openclaw-imessage-minimal.json5 ~/.openclaw/openclaw.json

# Or create manually (see below)
```

**Minimal hardened config:**
```json5
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

**Generate secure token:**
```bash
# Generate 256-bit token
openssl rand -hex 32

# Add to config
openclaw config set gateway.auth.token "YOUR_GENERATED_TOKEN"
```

---

## Step 4: Install BlueBubbles Server

1. **Download & Install**
   - https://bluebubbles.app/downloads/
   - Run the .dmg installer

2. **Grant Permissions** (System Settings → Privacy & Security)
   - Full Disk Access → BlueBubbles Server ✓
   - Automation → BlueBubbles Server → System Events ✓
   - Accessibility → BlueBubbles Server ✓ (optional, for typing indicators)

3. **Configure BlueBubbles**
   - Open BlueBubbles Server app
   - Set server password (use in OpenClaw config)
   - Note the port (default: 1234)
   - Enable "Private API" for full features

4. **Test BlueBubbles**
   ```bash
   # Should return JSON
   curl -s http://127.0.0.1:1234/api/v1/ping \
     -H "Authorization: Bearer YOUR_BLUEBUBBLES_PASSWORD"
   ```

---

## Step 5: Configure OpenClaw for BlueBubbles

```bash
# Set BlueBubbles connection
openclaw config set channels.bluebubbles.serverUrl "http://127.0.0.1:1234"
openclaw config set channels.bluebubbles.password "YOUR_BLUEBUBBLES_PASSWORD"

# Verify config
openclaw config get channels.bluebubbles
```

---

## Step 6: Set Up Model Authentication

```bash
# Anthropic (recommended)
openclaw auth add anthropic

# Or OpenAI
openclaw auth add openai

# Verify
openclaw auth list
```

---

## Step 7: Install Daemon (Auto-Start)

```bash
# Install launchd service
openclaw onboard --install-daemon

# Or manually
openclaw daemon install
openclaw daemon start

# Verify
openclaw daemon status
```

---

## Step 8: Start Gateway

```bash
# Manual start (for testing)
openclaw gateway run --verbose

# Or use daemon
openclaw daemon start
```

---

## Step 9: Verify Installation

```bash
# Check gateway status
openclaw gateway status

# Check channels
openclaw channels status
openclaw channels status --probe

# Check plugins
openclaw plugins list

# Security audit
openclaw security audit
openclaw security audit --deep
```

---

## Step 10: Test iMessage Flow

1. **Approve your phone number**
   ```bash
   # Check pending pairing requests
   openclaw pairing list

   # Approve your number
   openclaw pairing approve bluebubbles YOUR_PAIRING_CODE
   ```

2. **Send test message from your phone**
   - Send "hello" to the Mac Mini's iMessage
   - Should see pairing code or response

3. **Verify agent response**
   ```bash
   # Check agent logs
   openclaw agent status
   ```

---

## Troubleshooting

### Gateway won't start
```bash
# Check port availability
lsof -i :18789

# Check logs
tail -f ~/.openclaw/logs/gateway.log
```

### BlueBubbles not connecting
```bash
# Test BlueBubbles directly
curl http://127.0.0.1:1234/api/v1/ping

# Check Full Disk Access permission
# System Settings → Privacy & Security → Full Disk Access
```

### iMessage not received
```bash
# Check BlueBubbles webhook
openclaw channels status --probe

# Verify dmPolicy
openclaw config get channels.bluebubbles.dmPolicy
```

### Security audit warnings
```bash
# Run with fixes
openclaw security audit --fix

# Check specific issues
openclaw doctor
```

---

## Maintenance Commands

```bash
# Restart gateway
openclaw daemon restart

# View logs
openclaw logs --follow

# Update OpenClaw (see changelog first!)
# 1. Check changelog: _aiwk/changelog.md
# 2. Backup config: cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
# 3. Update: pnpm add -g openclaw@latest
# 4. Verify: openclaw security audit

# Check disk usage
du -sh ~/.openclaw/
```

---

## Security Checklist

Before going live:

- [ ] Token is 32+ characters (256-bit)
- [ ] `bind: "loopback"` (not lan/0.0.0.0)
- [ ] `elevated.enabled: false`
- [ ] `exec`, `process`, `apply_patch` in deny list
- [ ] `workspaceAccess: "none"`
- [ ] `dmPolicy: "pairing"` (not "open")
- [ ] `allowFrom: []` (empty until you approve numbers)
- [ ] BlueBubbles has strong password (32+ chars)
- [ ] `openclaw security audit` passes
- [ ] Model auth configured (not API keys in config)
- [ ] Config file permissions are 0o600 (see Phase C)
- [ ] Single-account login enforced (see Phase C)

---

## BlueBubbles Security Hardening (Phase C)

**Context:** Phase C of the security audit identified BlueBubbles-specific risks. These are mitigated by single-account login and proper file permissions.

### Single-Account Login Requirement

**Critical:** The Mac Mini must only have ONE user logged in when OpenClaw runs.

| Account | Role | Login Status |
|---------|------|--------------|
| `vikasplanbhatia` | OpenClaw operator | Logged IN |
| `vbadmin` | Primary admin | Logged OUT |

**Why this matters:**
- macOS allows multiple simultaneous GUI logins
- Any logged-in user's processes can reach `127.0.0.1` services
- With only one user logged in, cross-account attacks are eliminated

**Verification:**
```bash
# Check who is logged in
who

# Should show ONLY vikasplanbhatia
```

### Config File Permissions

**Critical:** The OpenClaw config contains the BlueBubbles password.

```bash
# Set correct permissions
chmod 600 ~/.openclaw/openclaw.json

# Verify
ls -la ~/.openclaw/openclaw.json
# Should show: -rw------- (owner read/write only)
```

### BlueBubbles Password Strength

Use a strong password (32+ characters) for BlueBubbles webhook authentication:

```bash
# Generate strong password
openssl rand -hex 32

# Update BlueBubbles Server with this password
# Then update OpenClaw config:
openclaw config set channels.bluebubbles.password "YOUR_NEW_PASSWORD"
```

### BlueBubbles Private API

Enable the Private API in BlueBubbles Server for full iMessage features:

1. Open BlueBubbles Server app
2. Go to Settings → Private API
3. Enable "Use Private API"
4. Grant required permissions when prompted

**Benefits:**
- Read receipts
- Typing indicators
- Tapback reactions
- Better message delivery status

### mediaLocalRoots Setting

Leave `mediaLocalRoots` empty (default) unless you explicitly need local file access:

```json5
{
  channels: {
    bluebubbles: {
      // mediaLocalRoots: []  // Leave empty or omit
    }
  }
}
```

**Why:** Non-empty `mediaLocalRoots` enables local filesystem access for attachments, which expands the attack surface.

### Phase C Verification Steps

After deployment, run these verification steps:

```bash
# 1. Verify single user login
who
# Expected: Only vikasplanbhatia

# 2. Verify config permissions
ls -la ~/.openclaw/openclaw.json
# Expected: -rw-------

# 3. Verify BlueBubbles connection
curl -s http://127.0.0.1:1234/api/v1/ping \
  -H "Authorization: Bearer YOUR_PASSWORD"
# Expected: JSON response with "message": "pong"

# 4. Verify webhook configuration
openclaw channels status --probe
# Expected: bluebubbles: connected

# 5. Run security audit
openclaw security audit --deep
# Expected: No critical issues
```

---

## Files Reference

| File | Purpose |
|------|---------|
| `~/.openclaw/openclaw.json` | Main config |
| `~/.openclaw/credentials/` | Channel credentials |
| `~/.openclaw/agents/` | Agent sessions, auth profiles |
| `~/.openclaw/logs/` | Gateway logs |
| `~/.openclaw/pairing/` | Pairing allowlists |

---

## Multi-Agent Architecture

This Mac Mini runs multiple AI systems alongside OpenClaw:

### Service Map

| Service | Port | Purpose |
|---------|------|---------|
| OpenClaw Gateway | 18789 | Primary AI agent, iMessage interface |
| BlueBubbles | 1234 | iMessage API bridge |
| AgentZero | TBD | Secondary task agent |
| Ollama | 11434 | Local LLM for memory/learning |

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Mac Mini (vikasplanbhatia)              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │  BlueBubbles │    │   OpenClaw   │    │  AgentZero   │  │
│  │   :1234      │◄──►│    :18789    │◄──►│    :TBD      │  │
│  │  (iMessage)  │    │  (Conduit)   │    │ (Secondary)  │  │
│  └──────────────┘    └──────┬───────┘    └──────────────┘  │
│                             │                               │
│                             ▼                               │
│                      ┌──────────────┐                       │
│                      │    Ollama    │                       │
│                      │   :11434     │                       │
│                      │ (Local LLM)  │                       │
│                      └──────────────┘                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
         ▲
         │ iMessage
         ▼
    ┌─────────┐
    │  Phone  │
    └─────────┘
```

### Security Considerations

1. **Port Isolation**
   - All services bind to `127.0.0.1` only
   - Each service uses independent auth tokens
   - No service should bind to `0.0.0.0`

2. **Ollama Integration**
   - Default port: 11434
   - No auth by default - only accessible from localhost
   - Can be used by OpenClaw for local inference
   - Configure in OpenClaw: `providers.ollama.baseUrl: "http://127.0.0.1:11434"`

3. **AgentZero Coexistence**
   - Separate working directories
   - Avoid port conflicts
   - Consider resource limits (memory, CPU)

4. **Inter-Agent Communication Options**
   - OpenClaw can spawn subagents for tasks
   - AgentZero runs independently
   - Could use shared Ollama for memory/context
   - Avoid direct API calls between agents (security boundary)

### Ollama Setup

```bash
# Install Ollama
brew install ollama

# Start Ollama service
ollama serve

# Pull a model for memory/learning
ollama pull llama3.2
# Or for better quality:
ollama pull llama3.1:70b  # Requires significant RAM

# Verify
curl http://127.0.0.1:11434/api/tags
```

### OpenClaw + Ollama Config (Optional)

If using Ollama as a provider in OpenClaw:

```json5
{
  providers: {
    ollama: {
      enabled: true,
      baseUrl: "http://127.0.0.1:11434",
      // No API key needed for local Ollama
    },
  },
  // Use Ollama for specific tasks
  agents: {
    list: {
      "memory-agent": {
        provider: "ollama",
        model: "llama3.2",
        // ...
      },
    },
  },
}
```

### Resource Planning

| Service | RAM (est.) | CPU | Notes |
|---------|------------|-----|-------|
| OpenClaw | 500MB | Low | Node.js, WebSocket server |
| BlueBubbles | 200MB | Low | Swift app |
| AgentZero | TBD | TBD | Depends on config |
| Ollama (idle) | 500MB | None | Base memory |
| Ollama (7B model) | 4-8GB | High | During inference |
| Ollama (70B model) | 40GB+ | High | Requires M2 Max+ |

**Recommendation:** For Mac Mini with 16GB RAM, use 7B-13B models. For 32GB+, can run 70B models.
