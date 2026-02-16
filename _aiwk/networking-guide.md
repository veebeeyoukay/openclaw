# OpenClaw Networking Setup Guide

**Date:** 2026-02-16
**Target:** Mac Mini iMessage-only receiver agent

---

## Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Mac Mini                                  │
│                                                                  │
│   ┌─────────────────┐     ┌──────────────────────────────────┐  │
│   │  BlueBubbles    │────▶│  OpenClaw Gateway                │  │
│   │  :1234          │     │  127.0.0.1:18789                 │  │
│   │  (iMessage API) │◀────│  - Receives webhooks             │  │
│   └─────────────────┘     │  - Processes messages            │  │
│           ▲               │  - Routes to Pi agent            │  │
│           │               └──────────────────────────────────┘  │
│   ┌───────┴───────┐                      │                      │
│   │ Messages.app  │                      │ (outbound only)      │
│   │ (iMessage)    │                      ▼                      │
│   └───────────────┘              ┌───────────────┐              │
│                                  │ Anthropic API │              │
│                                  │ (Claude)      │              │
│                                  └───────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Bind Modes

OpenClaw Gateway supports multiple bind modes:

| Mode | Address | Security | Use Case |
|------|---------|----------|----------|
| `loopback` | 127.0.0.1 | **Safest** | Local-only (recommended) |
| `lan` | 0.0.0.0 | Requires auth | LAN access needed |
| `tailnet` | Tailscale IP | Requires auth | Remote via Tailscale |
| `custom` | Your IP | Requires auth | Specific interface |

### Recommended: Loopback Only

For iMessage conduit, loopback is sufficient:

```json5
{
  gateway: {
    bind: "loopback",  // 127.0.0.1 only
    port: 18789,
  },
}
```

**Why loopback:**
- BlueBubbles runs on same machine → localhost communication
- No external network exposure
- No firewall rules needed
- Simplest security model

---

## Port Configuration

### Default Ports

| Service | Default Port | Configurable |
|---------|--------------|--------------|
| OpenClaw Gateway | 18789 | Yes |
| BlueBubbles Server | 1234 | Yes |
| BlueBubbles webhook | /bluebubbles-webhook | Yes |

### Changing Gateway Port

```json5
{
  gateway: {
    port: 18789,  // Change if needed
  },
}
```

### Verify Port Availability

```bash
# Check if port is in use
lsof -i :18789

# Find process using port
lsof -i :18789 | grep LISTEN
```

---

## BlueBubbles Webhook Configuration

BlueBubbles sends message events to OpenClaw via webhook:

```
BlueBubbles ──HTTP POST──▶ OpenClaw Gateway
   :1234                    127.0.0.1:18789/bluebubbles-webhook
```

### Configuration

```json5
{
  channels: {
    bluebubbles: {
      serverUrl: "http://127.0.0.1:1234",      // BlueBubbles server
      webhookPath: "/bluebubbles-webhook",      // Webhook endpoint
      password: "YOUR_BLUEBUBBLES_PASSWORD",    // Auth token
    },
  },
}
```

### BlueBubbles Server Settings

In BlueBubbles Server app:
1. Settings → Server → Port: `1234`
2. Settings → Webhooks → Add webhook:
   - URL: `http://127.0.0.1:18789/bluebubbles-webhook`
   - Events: New Message, Message Updated
3. Settings → Security → Password: Set strong password

### Test Webhook

```bash
# From Mac Mini terminal
curl -X POST http://127.0.0.1:18789/bluebubbles-webhook \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_GATEWAY_TOKEN" \
  -d '{"type": "test"}'
```

---

## Firewall Configuration

### macOS Firewall (Recommended Settings)

For loopback-only setup, **no firewall changes needed**.

If using `bind: "lan"`:

```bash
# Allow OpenClaw Gateway (if prompted)
# System Settings → Privacy & Security → Firewall → Options
# Add openclaw or node to allowed apps
```

### pf Firewall (Advanced)

```bash
# /etc/pf.conf additions (if using pf)
# Allow localhost only for OpenClaw
pass in on lo0 proto tcp from any to 127.0.0.1 port 18789
```

---

## Remote Access Options

### Option 1: Tailscale (Recommended for Remote)

If you need to access the Mac Mini remotely:

```json5
{
  gateway: {
    bind: "tailnet",  // Binds to Tailscale IP
    auth: { mode: "password", password: "STRONG_PASSWORD" },
  },
}
```

**Setup:**
1. Install Tailscale on Mac Mini
2. Install Tailscale on remote device
3. Both join same tailnet
4. Access via Tailscale IP: `http://100.x.y.z:18789`

**Tailscale Serve (HTTPS):**
```bash
# Expose gateway via Tailscale with HTTPS
tailscale serve https:18789 / http://127.0.0.1:18789
```

**Tailscale Funnel (Public Internet):**
```bash
# WARNING: Exposes to internet - requires password auth
tailscale funnel 18789
```

### Option 2: SSH Tunnel

```bash
# From remote machine
ssh -L 18789:127.0.0.1:18789 user@mac-mini.local

# Then access locally
curl http://127.0.0.1:18789/status
```

### Option 3: VPN

Use any VPN solution to access Mac Mini's network, then connect to `mac-mini.local:18789`.

---

## Network Security Checklist

### Loopback Setup (Recommended)

- [x] `bind: "loopback"` — Only localhost access
- [x] No firewall rules needed
- [x] No port forwarding needed
- [x] BlueBubbles on same machine

### LAN/Remote Setup (If Required)

- [ ] `bind: "lan"` or `bind: "tailnet"`
- [ ] Strong auth token (32+ chars)
- [ ] Firewall allows only necessary IPs
- [ ] No port 18789 exposed to internet
- [ ] Tailscale/VPN for remote access

---

## DNS / Hostname

### Local Access

```bash
# Mac Mini hostname
scutil --get LocalHostName  # e.g., mac-mini

# Access via Bonjour
http://mac-mini.local:18789
```

### Tailscale DNS

```bash
# Tailscale hostname
tailscale status  # Shows hostname

# Access via MagicDNS
http://mac-mini.tailnet-name.ts.net:18789
```

---

## Troubleshooting

### Gateway Not Accessible

```bash
# Check gateway is running
openclaw gateway status

# Check bind address
openclaw config get gateway.bind
openclaw config get gateway.port

# Check listening
lsof -i :18789 | grep LISTEN
netstat -an | grep 18789
```

### BlueBubbles Webhook Failing

```bash
# Test BlueBubbles server
curl http://127.0.0.1:1234/api/v1/ping

# Check webhook in OpenClaw logs
openclaw logs --follow | grep bluebubbles

# Verify webhook URL in BlueBubbles settings
# Should be: http://127.0.0.1:18789/bluebubbles-webhook
```

### Connection Refused

```bash
# Check firewall
sudo pfctl -s rules | grep 18789

# Check if gateway crashed
ps aux | grep openclaw

# Restart gateway
openclaw daemon restart
```

### Timeout Issues

```bash
# Check network latency
ping 127.0.0.1

# Increase timeout in config
openclaw config set gateway.timeout 30000
```

---

## Network Diagram: Full Setup

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              INTERNET                                     │
│                                  │                                        │
│                          (blocked by firewall)                            │
│                                  │                                        │
└──────────────────────────────────┼────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                           HOME NETWORK                                    │
│                                                                          │
│   ┌────────────────┐         ┌────────────────────────────────────────┐  │
│   │   iPhone       │         │            Mac Mini                     │  │
│   │                │         │                                         │  │
│   │  ┌──────────┐  │ iMessage│  ┌─────────┐    ┌───────────────────┐  │  │
│   │  │ Messages │──┼────────▶│  │Messages │───▶│ BlueBubbles       │  │  │
│   │  │   App    │  │   (E2E) │  │  .app   │    │ Server :1234      │  │  │
│   │  └──────────┘  │         │  └─────────┘    └─────────┬─────────┘  │  │
│   │                │         │                           │ webhook    │  │
│   └────────────────┘         │                           ▼            │  │
│                              │  ┌───────────────────────────────────┐ │  │
│                              │  │     OpenClaw Gateway              │ │  │
│                              │  │     127.0.0.1:18789               │ │  │
│                              │  │                                   │ │  │
│                              │  │  ┌─────────────┐  ┌────────────┐ │ │  │
│                              │  │  │ Pi Agent    │  │ Tool       │ │ │  │
│                              │  │  │ (messaging) │  │ Policies   │ │ │  │
│                              │  │  └──────┬──────┘  └────────────┘ │ │  │
│                              │  └─────────┼─────────────────────────┘ │  │
│                              │            │                           │  │
│                              └────────────┼───────────────────────────┘  │
│                                           │                              │
└───────────────────────────────────────────┼──────────────────────────────┘
                                            │ HTTPS (outbound only)
                                            ▼
                               ┌─────────────────────────┐
                               │    Anthropic API        │
                               │    api.anthropic.com    │
                               └─────────────────────────┘
```

---

## Summary

| Component | Address | Protocol | Direction |
|-----------|---------|----------|-----------|
| OpenClaw Gateway | 127.0.0.1:18789 | HTTP/WS | Inbound (local only) |
| BlueBubbles Server | 127.0.0.1:1234 | HTTP | Bidirectional (local) |
| BlueBubbles Webhook | /bluebubbles-webhook | HTTP POST | Inbound (local) |
| Anthropic API | api.anthropic.com:443 | HTTPS | Outbound only |
| iMessage | Apple servers | Proprietary | Via Messages.app |

**Key Point:** All local communication stays on loopback (127.0.0.1). Only outbound HTTPS to Anthropic API.
