# OpenClaw Deployment Changelog

**Purpose:** Track version changes to prevent updates from breaking the iMessage conduit setup.

---

## Current Installation

| Field | Value |
|-------|-------|
| **Installed Version** | `_____` (fill in after install) |
| **Install Date** | `_____` |
| **Node Version** | `_____` |
| **macOS Version** | `_____` |
| **BlueBubbles Version** | `_____` |

---

## Config Customizations (Do Not Lose)

These are non-default settings critical to the hardened setup:

```json5
// CRITICAL: These settings differ from defaults
{
  plugins: {
    allow: ["bluebubbles"],                    // Default: none (loads all)
    deny: ["device-pair", "phone-control", "talk-voice"],  // Default: none
  },
  tools: {
    profile: "messaging",                       // Default: "full"
    elevated: { enabled: false },               // Default: may be true
    deny: ["exec", "process", "apply_patch", ...],  // Default: none
    sessions: { visibility: "tree" },           // Default: "all"
  },
  agents: {
    defaults: {
      sandbox: { workspaceAccess: "none" },     // Default: "rw"
    },
  },
  channels: {
    web: { enabled: false },                    // Default: true
    bluebubbles: { dmPolicy: "pairing" },       // Default: varies
  },
}
```

---

## Before Updating

1. **Read release notes**
   ```bash
   # Check current version
   openclaw --version

   # Check latest version
   npm view openclaw version

   # Read changelog
   # https://github.com/openclaw/openclaw/releases
   ```

2. **Backup config**
   ```bash
   cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.$(date +%Y%m%d)
   ```

3. **Check breaking changes below**

4. **Update**
   ```bash
   pnpm add -g openclaw@X.Y.Z  # Specific version preferred
   ```

5. **Verify**
   ```bash
   openclaw security audit --deep
   openclaw channels status --probe
   ```

---

## Breaking Changes to Watch

### Config Keys (may change between versions)

| Key | Status | Notes |
|-----|--------|-------|
| `plugins.allow` | Stable | Core feature |
| `plugins.deny` | Stable | Core feature |
| `tools.profile` | Stable | Core feature |
| `tools.elevated` | Stable | Security critical |
| `tools.deny` | Stable | Security critical |
| `sandbox.workspaceAccess` | Stable | Security critical |
| `channels.bluebubbles.*` | Extension | May change with extension updates |
| `dmPolicy` | Stable | Security critical |

### Tool Names (may be renamed)

| Tool | Current Name | Watch For |
|------|--------------|-----------|
| Shell execution | `exec` | Aliases: `bash`, `shell` |
| File patch | `apply_patch` | Aliases: `apply-patch` |
| Background process | `process` | May split into subtypes |

### Plugin IDs (may change)

| Plugin | Current ID | Notes |
|--------|------------|-------|
| BlueBubbles | `bluebubbles` | Stable |
| Legacy iMessage | `imessage` | May deprecate |
| Device Pair | `device-pair` | Bundled, auto-enables |
| Phone Control | `phone-control` | Bundled, auto-enables |
| Talk Voice | `talk-voice` | Bundled, auto-enables |

---

## Version History

### v_____ (Initial Install)
- **Date:** _____
- **Config:** Hardened iMessage-only conduit
- **Notes:**
  - Installed from `_aiwk/openclaw-imessage-minimal.json5`
  - Phase B audit findings applied

### Template for Future Updates

```
### vX.Y.Z (YYYY-MM-DD)
- **Previous:** vA.B.C
- **Breaking Changes:**
  - [ ] None / [ ] See notes
- **Config Changes Required:**
  - None / List changes
- **Verification:**
  - [ ] `openclaw security audit` passes
  - [ ] `openclaw channels status --probe` works
  - [ ] iMessage send/receive tested
- **Notes:**
  - ...
```

---

## Rollback Procedure

If an update breaks the setup:

```bash
# 1. Stop gateway
openclaw daemon stop

# 2. Restore config backup
cp ~/.openclaw/openclaw.json.YYYYMMDD ~/.openclaw/openclaw.json

# 3. Downgrade OpenClaw
pnpm add -g openclaw@PREVIOUS_VERSION

# 4. Restart
openclaw daemon start

# 5. Verify
openclaw security audit
openclaw channels status --probe
```

---

## Known Issues / Workarounds

| Issue | Version | Workaround |
|-------|---------|------------|
| (none yet) | | |

---

## Monitoring After Updates

```bash
# Quick health check
openclaw gateway status && openclaw channels status

# Full security check
openclaw security audit --deep

# Test iMessage flow
# Send message from phone, verify response

# Check logs for errors
openclaw logs --follow | grep -i error
```

---

## Update Notification

To be notified of new releases:

1. **GitHub Watch** — https://github.com/openclaw/openclaw → Watch → Releases only
2. **Check manually**
   ```bash
   npm view openclaw version
   ```

---

## Emergency Contacts / Resources

- **Docs:** https://docs.openclaw.ai
- **Discord:** https://discord.gg/clawd
- **Issues:** https://github.com/openclaw/openclaw/issues
- **Security:** See `_aiwk/security-audit-report.md`
