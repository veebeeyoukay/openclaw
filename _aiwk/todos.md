# AI Working Folder Todos

**Last updated:** 2026-02-16

## Completed

- [x] Security audit plan (phases, OWASP mapping, deliverables)
- [x] iMessage-only setup guide (plugins.allow, config, verification)
- [x] Limited agent config (receiver/conduit role, tool deny list)
- [x] Hardened config template (`openclaw-imessage-minimal.json5`)
- [x] Execute Phase B of security audit (codebase review) ✓ 2026-02-16
- [x] Produce formal audit report (`security-audit-report.md`) ✓ 2026-02-16
- [x] Full setup guide (`setup-guide.md`) ✓ 2026-02-16
- [x] Networking guide (`networking-guide.md`) ✓ 2026-02-16
- [x] Changelog tracker (`changelog.md`) ✓ 2026-02-16

## Pending / Next Steps

### Security Audit (Remaining Phases)
- [x] Execute Phase C: iMessage-specific checks (BlueBubbles/imsg) ✓ 2026-02-16
- [x] Execute Phase D: Limited-agent config validation ✓ 2026-02-16

### Phase C Verification Steps (Post-Deployment)
- [ ] Verify single user login: `who` (only vikasplanbhatia)
- [ ] Verify config permissions: `ls -la ~/.openclaw/openclaw.json` (should be -rw-------)
- [ ] Verify BlueBubbles connection: `curl -s http://127.0.0.1:1234/api/v1/ping -H "Authorization: Bearer PASSWORD"`
- [ ] Verify webhook: `openclaw channels status --probe`
- [ ] Run security audit: `openclaw security audit --deep`

### Mac Mini Installation - Storage Setup
- [ ] Format NVMe as APFS "AgentStorage"
- [ ] Create directory structure on NVMe
- [ ] Verify auto-mount on boot
- [ ] Create symlinks after Step 2 (~/agents, ~/.ollama/models, ~/.openclaw/logs)

### Mac Mini Installation - OpenClaw
- [ ] Install Node.js 22+ on Mac Mini
- [ ] Install OpenClaw on Mac Mini
- [ ] Install BlueBubbles Server on Mac Mini
- [ ] Grant Full Disk Access + Automation permissions
- [ ] Copy hardened config to ~/.openclaw/openclaw.json
- [ ] Generate and set secure gateway token
- [ ] Configure BlueBubbles webhook
- [ ] Set up model authentication (Anthropic)
- [ ] Install daemon (launchd)
- [ ] Run `openclaw security audit --deep`
- [ ] Test iMessage send/receive flow
- [ ] Approve your phone number via pairing
- [ ] Fill in `changelog.md` with installed versions

### Mac Mini Installation - Supporting Services
- [ ] Install Ollama (`brew install ollama`)
- [ ] Pull local models (llama3.2, nomic-embed-text)
- [ ] Configure Ollama as OpenClaw provider (if needed)
- [ ] Install AgentZero
- [ ] Configure AgentZero port (avoid conflicts)
- [ ] Verify all services bind to 127.0.0.1 only
- [ ] Test resource usage under load

### Multi-Agent Infrastructure
- [ ] Create shared directory structure (`~/agents/tasks/`, `~/agents/audit/`, etc.)
- [ ] Set directory permissions (chmod 700)
- [ ] Implement OpenClaw task handoff skill/tool
- [ ] Configure AgentZero task polling
- [ ] Set up audit logging for both agents
- [ ] Test end-to-end handoff flow
- [ ] Verify audit trail completeness
