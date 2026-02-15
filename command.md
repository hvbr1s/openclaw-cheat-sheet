# OpenClaw GCP VM Command Reference

Quick reference for managing OpenClaw running on Google Cloud Platform VM.

---

## 🔐 VM Access & Setup

### SSH Connection
```bash
ssh -i ~/.ssh/YOUR_SSH_KEY_NAME YOUR_USERNAME@YOUR_VM_IP
```

### Update SSH Keys (if needed)
```bash
gcloud compute instances add-metadata YOUR_INSTANCE_NAME \
  --zone=YOUR_ZONE \
  --metadata-from-file ssh-keys=<(echo "YOUR_USERNAME:ssh-ed25519 AAAAC3Nza...YOUR_PUBLIC_KEY...XYZ YOUR_EMAIL@example.com")
```

### Enable Persistent Sessions
```bash
# Enable linger (allows services to run after logout)
/usr/bin/sudo.ws loginctl enable-linger YOUR_USERNAME

# Verify linger is enabled
loginctl show-user YOUR_USERNAME -p Linger
```

---

## 🗂️ Understanding the Architecture

### Install Tree Utility
```bash
sudo apt install tree
```

### View OpenClaw Directory Structure
```bash
# View the complete OpenClaw architecture
tree ~/.openclaw/

# Limit depth for a high-level overview
tree -L 2 ~/.openclaw/

# Include hidden files
tree -a ~/.openclaw/
```

**Key directories you'll see:**
- `workspace/` — Your projects and secrets
- `workspace/secrets/` — Centralized credentials (.env, private.pem)
- `memory/` — Daily logs and session memory
- `openclaw.json` — Main configuration file

---

## 🔑 Authentication & Gateway

### Gateway Token Management
Gateway token provides shared auth for the Gateway + Control UI.

**Location:**
- Config: `~/.openclaw/openclaw.json` (gateway.auth.token)
- Environment: `OPENCLAW_GATEWAY_TOKEN`
- Browser: `localStorage` (openclaw.control.settings.v1)

**Commands:**
```bash
# View current token
openclaw config get gateway.auth.token

# Generate new token
openclaw doctor --generate-gateway-token

# Open dashboard (get tokenized URL)
openclaw dashboard --no-open
```

**Note:** If prompted in Control UI, paste the token into settings or use the tokenized dashboard URL.

---

## 📊 Status & Monitoring

### Check OpenClaw Status
```bash
# Quick status check
openclaw status

# Follow logs in real-time
openclaw logs --follow
```

### System Logs (Lingered Services)
```bash
# Follow user service logs
journalctl --user -f

# Check user session status
loginctl show-user YOUR_USERNAME
```

---

## ⚙️ Configuration

### Main Config File
```bash
~/.openclaw/openclaw.json
```

### Model Configuration
- **Switch to KimiK 2.5**: [docs.openclaw.ai/providers/moonshot](https://docs.openclaw.ai/providers/moonshot)
- **Set compaction default**: 0.4

---

## 🔒 Secrets Management

### Secrets Location
All credentials stored in centralized location:
```
~/.openclaw/workspace/secrets/
├── .env          ← API keys & vault IDs
└── private.pem   ← EC signing key
```

**Note:** Both `fordefi-swap/` and `jupiter-swap/` read from here. Future projects should point to `../secrets/`.

### Rotate Credentials
```bash
# Edit environment variables
nano ~/.openclaw/workspace/secrets/.env

# Edit private key
nano ~/.openclaw/workspace/secrets/private.pem
```

**No restarts needed** — scripts read fresh on each run.

### Security Permissions
```bash
chmod 600 ~/.openclaw/.env
chmod 600 ~/.openclaw/workspace/.env
chmod 600 ~/.openclaw/workspace/secrets/.env
chmod 600 ~/.openclaw/workspace/secrets/private.pem
```

---

## 🎯 Session Management

### Session Commands
```bash
# List all sessions
openclaw sessions list

# Kill a specific session
openclaw sessions kill <session-key>
```

### Session Best Practices

**When to Start a New Session:**

| Scenario                      | Action                    |
| ----------------------------- | ------------------------- |
| New topic / task              | New session               |
| Context getting full (>80%)   | Compact or new session    |
| Different project / workspace | New session               |
| Debugging something complex   | New session (clean slate) |
| Continuing same conversation  | Stay in current           |

### What Persists Across Sessions

✅ **Persists (File-based memory):**
- `MEMORY.md` — curated long-term memory
- `memory/YYYY-MM-DD.md` — daily logs
- `SOUL.md`, `USER.md` — identity/persona
- `TOOLS.md`, `HEARTBEAT.md` — configuration
- Code files, docs, secrets

❌ **Doesn't Persist:**
- Conversation history (each session is isolated)
- Tool call history
- Temporary context

### Memory Strategy
1. **Daily logs** → `memory/YYYY-MM-DD.md` (raw events)
2. **Curated memory** → `MEMORY.md` (distilled knowledge)
3. **Per-session** → Re-read `SOUL.md`, `USER.md`, and recent memory files at startup

Each session wakes up fresh and reads memory files to rehydrate context.

---

## 🛠️ Skills & Extensions

### Add Skills
```bash
# Example: Add Solana dev skill
npx skills add https://github.com/solana-foundation/solana-dev-skill
```

---

## 📝 Quick Reference

| Task                  | Command                                              |
| --------------------- | ---------------------------------------------------- |
| SSH to VM             | `ssh -i ~/.ssh/YOUR_KEY YOUR_USER@YOUR_VM_IP`        |
| View directory tree   | `tree ~/.openclaw/`                                  |
| Check status          | `openclaw status`                                    |
| View logs             | `openclaw logs --follow`                             |
| View gateway token    | `openclaw config get gateway.auth.token`             |
| Generate new token    | `openclaw doctor --generate-gateway-token`           |
| Open dashboard        | `openclaw dashboard --no-open`                       |
| List sessions         | `openclaw sessions list`                             |
| Kill session          | `openclaw sessions kill <session-key>`               |
| Edit secrets          | `nano ~/.openclaw/workspace/secrets/.env`            |