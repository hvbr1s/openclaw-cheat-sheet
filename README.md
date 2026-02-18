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

### Gateway Control
```bash
# Stop the gateway service
openclaw gateway stop

# Restart the gateway service
openclaw gateway restart
```

---

## 🌐 Tailscale Integration

Expose the OpenClaw gateway securely over your Tailscale network without opening any public ports.

### Step 1 — Install Tailscale on the VM

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --auth-key=YOUR_TAILSCALE_AUTH_KEY
```

### Step 2 — Enable HTTPS Certificates

Go to [https://login.tailscale.com/admin/dns](https://login.tailscale.com/admin/dns) and enable **HTTPS Certificates**. This is required for `tailscale serve` to work with TLS.

### Step 3 — Configure OpenClaw

Edit `~/.openclaw/openclaw.json` and update the `gateway` section:

```json
{
  "gateway": {
    "bind": "loopback",
    "trustedProxies": ["127.0.0.1"],
    "auth": {
      "mode": "token",
      "allowTailscale": true
    },
    "tailscale": {
      "mode": "serve",
      "resetOnExit": true
    }
  }
}
```

**What each setting does:**

| Setting | Value | Effect |
| --- | --- | --- |
| `bind` | `"loopback"` | Gateway only listens on `127.0.0.1` — never exposed directly to the internet |
| `trustedProxies` | `["127.0.0.1"]` | Trust forwarded identity headers from the Tailscale Serve proxy |
| `auth.mode` | `"token"` | Token-based authentication |
| `auth.allowTailscale` | `true` | Tailnet devices authenticate via identity headers — no manual token needed |
| `tailscale.mode` | `"serve"` | OpenClaw auto-configures `tailscale serve` on startup, proxying HTTPS to the local gateway port |
| `tailscale.resetOnExit` | `true` | Removes the serve config when OpenClaw stops |

### Step 4 — Restart the Gateway

```bash
openclaw gateway restart

# Verify serve is active
tailscale serve status
```

### Step 5 — Approve Your Browser Device (Important!)

When you first open the dashboard from a new device, you'll see a **"pairing required"** message. This is expected — OpenClaw treats each browser as a new operator device that needs a one-time approval.

```bash
# Find the pending device request
openclaw devices list

# Approve it by ID
openclaw devices approve <id>
```

### Result

Dashboard is now accessible from any device on your tailnet at:

```
https://YOUR_HOSTNAME.YOUR_TAILNET.ts.net/
```

Tailnet devices are trusted automatically — no token prompt after initial device approval.

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

## 💾 Backup & Restore

### Create Backup
```bash
# Create dated backup archive
tar -czf ~/openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/

# Create backup with timestamp (for multiple backups per day)
tar -czf ~/openclaw-backup-$(date +%Y%m%d-%H%M%S).tar.gz ~/.openclaw/

# Backup with progress indicator
tar -czf ~/openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/ --verbose
```

### Restore from Backup
```bash
# Extract backup to home directory
tar -xzf ~/openclaw-backup-YYYYMMDD.tar.gz -C ~/

# Preview backup contents without extracting
tar -tzf ~/openclaw-backup-YYYYMMDD.tar.gz | less
```

### Transfer Backup to Local Machine
```bash
# From your local machine, download backup from VM
scp -i ~/.ssh/YOUR_SSH_KEY_NAME YOUR_USERNAME@YOUR_VM_IP:~/openclaw-backup-*.tar.gz ~/Downloads/

# Or use rsync for resumable transfers
rsync -avz -e "ssh -i ~/.ssh/YOUR_SSH_KEY_NAME" YOUR_USERNAME@YOUR_VM_IP:~/openclaw-backup-*.tar.gz ~/Downloads/
```

### Backup Best Practices
- **Before major changes**: Always backup before updating configurations or rotating secrets
- **Regular schedule**: Consider daily/weekly backups via cron
- **Store off-VM**: Download backups to local machine or cloud storage
- **Test restores**: Periodically verify backups can be restored successfully

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

## 🔧 Troubleshooting

### npm ENOTEMPTY Error During Update

**Symptom:** `npm install -g openclaw@latest` fails with:

```
npm error code ENOTEMPTY
npm error syscall rename
npm error path /home/YOUR_USERNAME/.npm-global/lib/node_modules/openclaw
npm error dest /home/YOUR_USERNAME/.npm-global/lib/node_modules/.openclaw-XXXXXXXX
npm error errno -39
npm error ENOTEMPTY: directory not empty, rename '...openclaw' -> '...openclaw-XXXXXXXX'
```

**Cause:** npm's atomic rename during install fails because the existing `openclaw` module directory is non-empty (e.g., from a previous partial install or running process).

**Fix:** Manually remove both the package directory and any leftover temp directory, then reinstall:

```bash
# Remove the existing install
rm -rf ~/.npm-global/lib/node_modules/openclaw

# Remove any leftover npm temp directories (the random suffix varies)
rm -rf ~/.npm-global/lib/node_modules/.openclaw-*

# Reinstall
npm install -g openclaw@latest
```

**Note:** The `npm warn deprecated` messages that appear after a successful install are harmless — they refer to transitive dependencies and do not affect functionality.

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
| Stop gateway          | `openclaw gateway stop`                              |
| Restart gateway       | `openclaw gateway restart`                           |
| List sessions         | `openclaw sessions list`                             |
| Kill session          | `openclaw sessions kill <session-key>`               |
| Edit secrets          | `nano ~/.openclaw/workspace/secrets/.env`            |
| Create backup         | `tar -czf ~/openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/` |