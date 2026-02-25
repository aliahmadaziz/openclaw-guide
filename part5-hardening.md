# Part 5: Hardening (30 min)

**Prerequisites:** Parts 1-4 complete (OpenClaw running, AI persona configured, infrastructure deployed, automation active)

**What you'll build:** Production-grade hardening layer — intrusion detection (CrowdSec), secret rotation protocol, config snapshots, three-tier recovery system, and WhatsApp scope verification to prevent pairing loops.

---

## 1. CrowdSec (Intrusion Detection & Prevention)

**WHY:** CrowdSec monitors your logs for attack patterns (SSH brute force, port scans, exploit attempts) and automatically bans malicious IPs via firewall. It's like fail2ban but crowd-sourced — you benefit from attack intelligence shared by 70,000+ servers.

### 1.1 Install CrowdSec

```bash
# Add CrowdSec repository
curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | bash

# Install CrowdSec engine
apt install -y crowdsec
```

*Why: CrowdSec is the detection engine that parses logs and identifies threats*

✅ **Verify:** `systemctl status crowdsec` shows "active (running)"

### 1.2 Install Firewall Bouncer

```bash
# Install iptables bouncer (integrates with UFW)
apt install -y crowdsec-firewall-bouncer-iptables
```

*Why: The bouncer is the enforcement layer — it translates CrowdSec's ban decisions into actual firewall rules*

✅ **Verify:** `systemctl status crowdsec-firewall-bouncer` shows "active (running)"

### 1.3 How It Works

**Detection flow:**
1. CrowdSec tails `/var/log/auth.log`, `/var/log/nginx/access.log`, etc.
2. Parsers identify attack patterns (5 failed SSH logins in 2 minutes = brute force)
3. Scenarios trigger decisions (ban IP for 4 hours)
4. Bouncer applies the ban via iptables/UFW

**Community intelligence:**
- CrowdSec shares anonymized attack data with the community
- You receive blocklists of known malicious IPs (automatic, opt-out via config)
- Your server contributes to global threat intelligence

### 1.4 Verify CrowdSec Operation

```bash
# Check active bans
cscli decisions list

# View parsed logs and metrics
cscli metrics

# List installed parsers (should include sshd)
cscli parsers list

# List active scenarios
cscli scenarios list
```

*Why: Confirms CrowdSec is processing logs and ready to ban attackers*

✅ **Verify:** `cscli metrics` shows non-zero "Lines parsed" count, `cscli decisions list` may show active bans (or be empty if no attacks yet)

### 1.5 Test CrowdSec (Optional)

**From your local machine, trigger a fake attack:**

```bash
# Attempt 6 failed SSH logins (will trigger ban after 5)
for i in {1..6}; do ssh fakeuser@YOUR_SERVER_IP; sleep 1; done
```

**On the server:**

```bash
# Check if your IP was banned
cscli decisions list

# Remove the ban (so you can still connect)
cscli decisions delete --ip YOUR_LOCAL_IP
```

*Why: Validates the detection → ban → enforcement pipeline*

**⚠️ SAFETY:** Only test from an IP you control. Use `cscli decisions delete` to unban yourself.

---

## 2. Secrets Hygiene

**Rule: NEVER hardcode secrets in scripts.** All API keys, tokens, and credentials go in `~/.clawdbot/webhook.env` (chmod 600, gitignored). Scripts source from env:

- **Shell scripts:** `source ~/.clawdbot/webhook.env` then use `$PUSHOVER_TOKEN`, `$GROQ_API_KEY`, etc.
- **Python scripts:** `from dotenv import load_dotenv; load_dotenv(os.path.expanduser("~/.clawdbot/webhook.env"))` then `os.environ.get("KEY")`

**WHY:** Your workspace is git-backed (hourly push). Hardcoded secrets in scripts = secrets in git history. Even in a private repo, this is a leak waiting to happen. Git history is permanent; `webhook.env` is gitignored and never leaves the server.

Ensure `.gitignore` includes:
```
.env
*.key
```

And verify no secrets are tracked: `git ls-files | xargs grep -l "sk_\|gsk_\|token=" 2>/dev/null`

## 3. Secret Rotation Protocol

**WHY:** Credentials decay. A compromised token 6 months old can bypass all your other security. Rotation limits the blast radius of any single leak.

### 2.1 Rotation Schedule

Create a checklist in your workspace:

```bash
nano /root/clawd/SECURITY.md
```

Add this section:

```markdown
# Secret Rotation Schedule

| Secret | Frequency | Last Rotated | Next Due |
|--------|-----------|--------------|----------|
| OpenClaw Gateway Token | 90 days | 2026-02-22 | 2026-05-23 |
| Webhook URL Secrets | 90 days | 2026-02-22 | 2026-05-23 |
| Google OAuth Tokens | Auto-refresh | (health checked daily) | N/A |
| SSH Keys | 365 days | 2026-02-22 | 2027-02-22 |
| Hetzner API Token | 365 days | 2026-02-22 | 2027-02-22 |
| Pushover Keys | Never (until compromised) | N/A | N/A |

## Rotation Procedures

### OpenClaw Gateway Token
1. Generate new token: `openclaw gateway token --new`
2. Update `~/.openclaw/config.json` → `gatewayToken` field
3. Restart gateway: `openclaw gateway restart`
4. Test: Send WhatsApp message, verify response

### Webhook URL Secrets
1. Generate new secret: `openssl rand -hex 16`
2. Update `~/.clawdbot/webhook.env` → endpoint paths
3. Update Cloudflare tunnel config (if paths changed)
4. Restart webhook: `systemctl restart openclaw-webhook`
5. Re-register Google Calendar webhook with new URL
6. Test: Trigger calendar event, verify delivery

### SSH Keys
1. On local machine: `ssh-keygen -t ed25519 -f ~/.ssh/id_openclaw_new`
2. Add new key to Hetzner console
3. Add new key to server: `~/.ssh/authorized_keys`
4. Test connection with new key
5. Remove old key from `authorized_keys` and Hetzner
6. Delete old local private key

### Google OAuth Tokens
- **Auto-refresh:** Tokens refresh automatically when expired
- **Health check:** `scripts/service-quick-check.py` validates tokens daily
- **Manual refresh:** `scripts/google-reauth.py <account>` forces re-auth
```

*Why: Written procedures prevent "how do I rotate this again?" delays when tokens expire*

### 2.2 Rotate OpenClaw Gateway Token (Example)

```bash
# Generate new token (outputs token to terminal)
openclaw gateway token --new

# Copy the token, then edit config
nano ~/.openclaw/config.json
```

Find the `gatewayToken` field and replace with new token:

```json
{
  "gatewayToken": "NEW_TOKEN_HERE",
  "whatsapp": { ... }
}
```

Save and restart:

```bash
openclaw gateway restart
openclaw gateway status
```

✅ **Verify:** Send a WhatsApp message, confirm bot responds

### 2.3 Rotate Webhook Secrets (Example)

```bash
# Generate new secret
openssl rand -hex 16

# Edit webhook env
nano ~/.clawdbot/webhook.env
```

Update the endpoint paths with new secret:

```bash
CALENDAR_WEBHOOK_PATH="/NEW_SECRET_HERE"
ALERTS_WEBHOOK_PATH="/alerts/NEW_SECRET_HERE"
AGENTMAIL_WEBHOOK_PATH="/agentmail/NEW_SECRET_HERE"
```

Restart webhook server:

```bash
systemctl restart openclaw-webhook
systemctl status openclaw-webhook
```

**Then update external services:**
- Google Calendar: Re-register webhook with new URL
- Alert services: Update webhook destination
- AgentMail: Update forwarding URL

✅ **Verify:** Trigger a test event for each webhook, confirm delivery

### 2.4 Rotation Script Skeleton

Create a helper script for automated rotation reminders:

```bash
nano /root/clawd/scripts/rotation-check.py
```

```python
#!/usr/bin/env python3
"""Check secret rotation schedule and alert if any are due."""

from datetime import datetime, timedelta
import sys

SECRETS = {
    "OpenClaw Gateway Token": {"frequency": 90, "last": "2026-02-22"},
    "Webhook URL Secrets": {"frequency": 90, "last": "2026-02-22"},
    "SSH Keys": {"frequency": 365, "last": "2026-02-22"},
}

def check_rotation():
    today = datetime.now()
    alerts = []
    
    for name, config in SECRETS.items():
        last_rotated = datetime.strptime(config["last"], "%Y-%m-%d")
        next_due = last_rotated + timedelta(days=config["frequency"])
        days_until = (next_due - today).days
        
        if days_until <= 7:
            alerts.append(f"⚠️ {name} due in {days_until} days (rotate by {next_due.date()})")
        elif days_until < 0:
            alerts.append(f"🚨 {name} OVERDUE by {abs(days_until)} days!")
    
    if alerts:
        print("\n".join(alerts))
        return 1
    else:
        print("✅ All secrets current")
        return 0

if __name__ == "__main__":
    sys.exit(check_rotation())
```

Make executable:

```bash
chmod +x /root/clawd/scripts/rotation-check.py
```

Add to daily cron (system crontab):

```bash
crontab -e
```

Add line:

```
0 10 * * * /root/clawd/scripts/rotation-check.py >> /root/clawd/memory/system-health.log 2>&1
```

*Why: Daily check alerts you 7 days before rotation is due, preventing expired credentials*

---

## 4. Config Snapshots

**WHY:** Configuration drift is silent. You change a setting, restart a service, something breaks, and you can't remember what worked. Snapshots give you a "gold standard" to restore in 3 seconds.

### 3.1 Create Snapshot Script

```bash
nano /root/clawd/scripts/restore-snapshot-config.sh
```

```bash
#!/bin/bash
# restore-snapshot-config.sh - Save/restore config snapshots

SNAPSHOT_DIR="$HOME/.clawdbot/snapshots"
LATEST_LINK="$SNAPSHOT_DIR/latest"

save_snapshot() {
    local LABEL="${1:-manual-$(date +%Y%m%d-%H%M%S)}"
    local SNAP_PATH="$SNAPSHOT_DIR/$LABEL"
    
    mkdir -p "$SNAP_PATH"
    
    echo "📸 Saving snapshot: $LABEL"
    
    # OpenClaw config
    if [ -f "$HOME/.openclaw/config.json" ]; then
        cp "$HOME/.openclaw/config.json" "$SNAP_PATH/openclaw.json"
        echo "  ✓ openclaw.json"
    fi
    
    # Systemd services
    mkdir -p "$SNAP_PATH/systemd"
    for service in openclaw-webhook.service; do
        if [ -f "/etc/systemd/system/$service" ]; then
            cp "/etc/systemd/system/$service" "$SNAP_PATH/systemd/"
            echo "  ✓ $service"
        fi
    done
    
    # SSH config
    if [ -f "/etc/ssh/sshd_config" ]; then
        cp "/etc/ssh/sshd_config" "$SNAP_PATH/sshd_config"
        echo "  ✓ sshd_config"
    fi
    
    # Crontab
    crontab -l > "$SNAP_PATH/crontab.txt" 2>/dev/null
    echo "  ✓ crontab"
    
    # UFW rules
    ufw status numbered > "$SNAP_PATH/ufw-rules.txt" 2>/dev/null
    echo "  ✓ ufw-rules"
    
    # Update 'latest' symlink
    rm -f "$LATEST_LINK"
    ln -s "$SNAP_PATH" "$LATEST_LINK"
    
    echo "✅ Snapshot saved: $SNAP_PATH"
    echo "   Restore with: $0 restore $LABEL"
}

restore_snapshot() {
    local LABEL="${1:-latest}"
    local SNAP_PATH="$SNAPSHOT_DIR/$LABEL"
    
    if [ ! -d "$SNAP_PATH" ]; then
        echo "❌ Snapshot not found: $LABEL"
        echo "Available snapshots:"
        ls -1 "$SNAPSHOT_DIR" | grep -v latest
        exit 1
    fi
    
    echo "⚠️  Restoring snapshot: $LABEL"
    read -p "This will overwrite current config. Continue? (yes/no): " confirm
    
    if [ "$confirm" != "yes" ]; then
        echo "Aborted."
        exit 0
    fi
    
    # Restore OpenClaw config
    if [ -f "$SNAP_PATH/openclaw.json" ]; then
        cp "$SNAP_PATH/openclaw.json" "$HOME/.openclaw/config.json"
        echo "  ✓ openclaw.json restored"
    fi
    
    # Restore systemd services
    if [ -d "$SNAP_PATH/systemd" ]; then
        cp "$SNAP_PATH/systemd/"* /etc/systemd/system/
        systemctl daemon-reload
        echo "  ✓ systemd services restored"
    fi
    
    # Restore SSH config
    if [ -f "$SNAP_PATH/sshd_config" ]; then
        cp "$SNAP_PATH/sshd_config" /etc/ssh/sshd_config
        echo "  ✓ sshd_config restored (restart sshd manually)"
    fi
    
    # Restore crontab
    if [ -f "$SNAP_PATH/crontab.txt" ]; then
        crontab "$SNAP_PATH/crontab.txt"
        echo "  ✓ crontab restored"
    fi
    
    echo "✅ Snapshot restored. Restart services as needed."
    echo "   Suggested: openclaw gateway restart && systemctl restart openclaw-webhook"
}

list_snapshots() {
    echo "Available snapshots:"
    ls -1t "$SNAPSHOT_DIR" | grep -v latest | while read snap; do
        echo "  - $snap"
    done
}

case "${1:-help}" in
    save)
        save_snapshot "$2"
        ;;
    restore)
        restore_snapshot "$2"
        ;;
    list)
        list_snapshots
        ;;
    *)
        echo "Usage: $0 {save|restore|list} [label]"
        echo ""
        echo "Examples:"
        echo "  $0 save gold-2026-02-22        # Save with label"
        echo "  $0 save                         # Save with timestamp"
        echo "  $0 restore gold-2026-02-22      # Restore specific snapshot"
        echo "  $0 restore                      # Restore 'latest'"
        echo "  $0 list                         # List all snapshots"
        exit 1
        ;;
esac
```

Make executable:

```bash
chmod +x /root/clawd/scripts/restore-snapshot-config.sh
```

### 3.2 Usage Examples

```bash
# Save current config with a descriptive label
/root/clawd/scripts/restore-snapshot-config.sh save gold-2026-02-22

# List all snapshots
/root/clawd/scripts/restore-snapshot-config.sh list

# Restore a specific snapshot
/root/clawd/scripts/restore-snapshot-config.sh restore gold-2026-02-22

# Restore latest (most recent save)
/root/clawd/scripts/restore-snapshot-config.sh restore
```

*Why: 3-second rollback when a config change breaks something*

### 3.3 Gold Standard Practice

**After confirming everything works (all parts of this guide complete):**

```bash
# Test everything first
openclaw gateway status
systemctl status openclaw-webhook
ufw status
crontab -l

# If all green, save your gold standard
/root/clawd/scripts/restore-snapshot-config.sh save gold-2026-02-22-final
```

*Why: This snapshot becomes your "last known good" state. If anything breaks in the future, restore this first.*

---

## 5. Recovery Scripts (Three-Tier System)

**WHY:** Failures have tiers. Config typo ≠ corrupted install ≠ dead server. Each tier needs a different recovery tool.

### 4.1 Tier 1: Config Bricked (restore-snapshot-config.sh)

**Scenario:** You edited `openclaw.json` wrong, gateway won't start, or webhook config is broken.

**Solution:** Restore last snapshot (covered in section 3)

```bash
/root/clawd/scripts/restore-snapshot-config.sh restore
openclaw gateway restart
systemctl restart openclaw-webhook
```

**Time to recovery:** 3 seconds  
**What it restores:** Config files only (openclaw.json, systemd services, SSH config, crontab, UFW rules)

### 4.2 Tier 2: OpenClaw Update Broke Things (restore-openclaw-install.sh)

**Scenario:** `npm update -g openclaw` upgraded to a buggy version, gateway crashes, pairing fails.

**Solution:** Restore previous OpenClaw version from backup

Create the script:

```bash
nano /root/clawd/scripts/restore-openclaw-install.sh
```

```bash
#!/bin/bash
# restore-openclaw-install.sh - Restore previous OpenClaw version

BACKUP_DIR="$HOME/.clawdbot/backups/openclaw"
LATEST_BACKUP="$BACKUP_DIR/latest"

echo "⚠️  Rolling back OpenClaw installation..."

# Stop gateway
openclaw gateway stop 2>/dev/null || true

# Backup current version before downgrade
CURRENT_VERSION=$(openclaw --version 2>/dev/null || echo "unknown")
echo "Current version: $CURRENT_VERSION"

# Check if backup exists
if [ ! -d "$LATEST_BACKUP" ]; then
    echo "❌ No backup found. Cannot restore."
    echo "   Backups should exist at: $BACKUP_DIR"
    exit 1
fi

# Uninstall current version
npm uninstall -g openclaw

# Restore from backup (reinstall specific version)
BACKUP_VERSION=$(cat "$LATEST_BACKUP/version.txt")
echo "Restoring version: $BACKUP_VERSION"

npm install -g openclaw@$BACKUP_VERSION

# Restore config (if it was also affected)
if [ -f "$LATEST_BACKUP/config.json" ]; then
    cp "$LATEST_BACKUP/config.json" "$HOME/.openclaw/config.json"
    echo "  ✓ Config restored from backup"
fi

# Restart gateway
openclaw gateway start

echo "✅ Rollback complete. Verify with: openclaw gateway status"
```

Make executable:

```bash
chmod +x /root/clawd/scripts/restore-openclaw-install.sh
```

*Why: Rollback to previous OpenClaw version when updates break things*

**Time to recovery:** 30 seconds  
**What it restores:** OpenClaw binary + config

### 4.3 Tier 3: Server is Dead (full-restore.sh)

**Scenario:** VPS corrupted, need to rebuild from scratch on a new server.

**Solution:** Full workspace restore from Google Drive backup

Create the script:

```bash
nano /root/clawd/scripts/full-restore.sh
```

```bash
#!/bin/bash
# full-restore.sh - Full workspace restore from Google Drive

BACKUP_NAME="clawd-workspace"
RESTORE_DIR="/root/clawd-restore"

echo "🔥 FULL RESTORE FROM GOOGLE DRIVE BACKUP"
echo "⚠️  This should only be run on a FRESH server"
read -p "Continue? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
    echo "Aborted."
    exit 0
fi

# Install rclone if not present
if ! command -v rclone &> /dev/null; then
    echo "Installing rclone..."
    curl https://rclone.org/install.sh | bash
fi

# Configure rclone (assumes you have the config)
if [ ! -f "$HOME/.config/rclone/rclone.conf" ]; then
    echo "❌ rclone not configured. Run: rclone config"
    exit 1
fi

# Pull latest backup from Google Drive
echo "📥 Downloading backup from Google Drive..."
mkdir -p "$RESTORE_DIR"
rclone copy gdrive:openclaw-backups/$BACKUP_NAME.tar.gz "$RESTORE_DIR/"

# Extract backup
cd /root
tar -xzf "$RESTORE_DIR/$BACKUP_NAME.tar.gz"

echo "✅ Backup extracted to /root/clawd"

# Next steps
echo ""
echo "Manual steps required:"
echo "1. Restore OpenClaw config: cp clawd/.openclaw/config.json ~/.openclaw/"
echo "2. Restore systemd services: cp clawd/systemd/* /etc/systemd/system/"
echo "3. Restore crontab: crontab clawd/crontab.txt"
echo "4. Install Node.js 22: curl -fsSL https://deb.nodesource.com/setup_22.x | bash && apt install nodejs"
echo "5. Install OpenClaw: npm install -g openclaw"
echo "6. Start gateway: openclaw gateway start"
echo "7. Pair WhatsApp (will need new QR scan)"
```

Make executable:

```bash
chmod +x /root/clawd/scripts/full-restore.sh
```

*Why: Last resort for catastrophic failures (server crash, data corruption, migrate to new VPS)*

**Time to recovery:** 10-15 minutes  
**What it restores:** Entire workspace + instructions for service restoration

### 4.4 Post-Recovery Checklist

**After ANY restore operation:**

```bash
# 1. Verify OpenClaw health
openclaw doctor

# 2. Test gateway connectivity
openclaw gateway probe

# 3. Send test message
# (From your phone: send "test" to bot WhatsApp number)

# 4. Check critical services
systemctl status openclaw-webhook crowdsec

# 5. Verify backups still running
crontab -l | grep backup

# 6. Save new snapshot after confirming recovery worked
/root/clawd/scripts/restore-snapshot-config.sh save post-recovery-$(date +%Y%m%d)
```

*Why: Confirms the restore actually worked and captures the recovered state*

---

## 6. Safe Gateway Restart Protocol (Critical)

**WHY:** Raw `openclaw gateway restart` can lose OpenClaw cron jobs (they're stored in memory during restart). You need a safe restart wrapper that backs up crons before restart and verifies after.

### 6.1 Create Safe Restart Script

```bash
nano /root/clawd/scripts/safe-gateway-restart.sh
```

```bash
#!/bin/bash
# safe-gateway-restart.sh - ALWAYS use instead of raw "openclaw gateway restart"
# Backs up crons before restart, verifies after, alerts on job loss

set -e

echo "🔄 Safe gateway restart starting..."

# Step 1: Backup current cron jobs
echo "  → Backing up cron jobs..."
openclaw cron list --json > /root/.clawdbot/cron-backup-prerestart.json
CRON_COUNT_BEFORE=$(openclaw cron list --json | jq '. jobs | length')
echo "    ✓ $CRON_COUNT_BEFORE jobs backed up"

# Step 2: Restart gateway
echo "  → Restarting gateway..."
openclaw gateway restart

# Step 3: Wait for gateway to be ready
echo "  → Waiting for gateway..."
sleep 5

# Step 4: Verify cron jobs survived
CRON_COUNT_AFTER=$(openclaw cron list --json | jq '.jobs | length')
echo "    ✓ $CRON_COUNT_AFTER jobs after restart"

if [ "$CRON_COUNT_BEFORE" != "$CRON_COUNT_AFTER" ]; then
    echo "❌ CRON JOB LOSS DETECTED: $CRON_COUNT_BEFORE before, $CRON_COUNT_AFTER after"
    echo "    Backup at: /root/.clawdbot/cron-backup-prerestart.json"
    echo "    Manual recovery required: openclaw cron import < backup.json"
    exit 1
else
    echo "✅ Safe restart complete. All $CRON_COUNT_AFTER cron jobs intact."
fi
```

Make executable:

```bash
chmod +x /root/clawd/scripts/safe-gateway-restart.sh
```

*Why: This wrapper ensures you never lose cron jobs during restart. Always use this instead of raw restart.*

### 6.2 Usage

**NEVER do this:**
```bash
openclaw gateway restart  # ❌ UNSAFE — can lose crons
```

**ALWAYS do this:**
```bash
/root/clawd/scripts/safe-gateway-restart.sh  # ✅ SAFE
```

### 6.3 Add to PATH (Optional)

```bash
echo 'export PATH="/root/clawd/scripts:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Now you can just run:
```bash
safe-gateway-restart.sh
```

**Bookmark this:** After every config change, use safe restart.

---

## 7. Device Scope Verification (Prevent WhatsApp Pairing Loops)

**WHY:** OpenClaw 1.x introduced stricter device scope requirements. If your paired device doesn't have `operator.write` scope, pairing will loop forever (QR code → "connected" → immediately disconnects → new QR code).

**Reference:** [OpenClaw GitHub Issue #23006](https://github.com/openclaw/openclaw/issues/23006)

### 7.1 Check Current Scope

```bash
# View current device info
openclaw status
```

Look for the `devices` section:

```json
{
  "devices": [
    {
      "id": "12345:67@s.whatsapp.net",
      "scope": "operator.read"   // ❌ Missing operator.write
    }
  ]
}
```

**Problem:** Scope only has `operator.read`. OpenClaw now requires `operator.write` for message sending.

### 7.2 Fix: Re-pair with Correct Scope

If scope is missing `operator.write`:

```bash
# 1. Stop gateway
openclaw gateway stop

# 2. Remove old pairing
rm -rf ~/.openclaw/auth_info_*

# 3. Re-run setup (will generate new QR code)
openclaw setup

# 4. Scan QR code from WhatsApp (Linked Devices → Link a Device)

# 5. Verify scope after pairing
openclaw status
```

**Correct output:**

```json
{
  "devices": [
    {
      "id": "12345:67@s.whatsapp.net",
      "scope": "operator.read,operator.write"   // ✅ Both scopes present
    }
  ]
}
```

### 7.3 Symptoms of Scope Issues

- QR code scans successfully but disconnects within 2 seconds
- Gateway logs show "device not authorized" or "missing scope"
- Bot receives messages but can't send replies (sends fail silently)
- Pairing loop: QR → connected → disconnected → new QR → repeat

**Solution:** Always re-pair (steps in 5.2) after major OpenClaw updates or if pairing loops occur.

### 7.4 Post-Update Verification Routine

**After every `openclaw update`:**

```bash
# 1. Check version
openclaw --version

# 2. Check gateway status
openclaw gateway status

# 3. Verify device scope
openclaw status | grep scope

# 4. Send test message to confirm send/receive works
# (WhatsApp message from your phone to bot)

# 5. If anything fails: re-pair (section 7.2)
```

*Why: Catches scope issues before they cause production failures*

---

## Part 5 Complete ✅

You should now have:

- [ ] CrowdSec installed and monitoring logs
- [ ] Firewall bouncer active (bans are enforced)
- [ ] Active bans visible in `cscli decisions list`
- [ ] Secret rotation schedule documented in `SECURITY.md`
- [ ] Rotation procedures written for all critical secrets
- [ ] Rotation check script running daily (`rotation-check.py`)
- [ ] Config snapshot script created (`restore-snapshot-config.sh`)
- [ ] Gold standard snapshot saved
- [ ] Three recovery scripts created (Tier 1/2/3)
- [ ] Post-recovery checklist ready
- [ ] WhatsApp device scope verified (`operator.read,operator.write`)
- [ ] Post-update verification routine documented

---

## Gold Standard Snapshot

**THIS IS THE MOMENT.** Parts 1-5 are complete. Your system is production-ready. Save this state.

```bash
# Final verification before snapshot
echo "=== FINAL VERIFICATION ==="

# 1. OpenClaw
openclaw gateway status
openclaw status | grep scope

# 2. Infrastructure
systemctl status openclaw-webhook crowdsec crowdsec-firewall-bouncer

# 3. Backups
crontab -l | grep -E 'backup|verify'

# 4. Security
ufw status
cscli metrics

# 5. Automation
openclaw cron list

# If all green:
/root/clawd/scripts/restore-snapshot-config.sh save gold-2026-02-22-final

echo "✅ Gold standard saved. Future rollbacks: restore-snapshot-config.sh restore gold-2026-02-22-final"
```

**What you just saved:**
- OpenClaw config (WhatsApp pairing, persona, models)
- All systemd services (webhook, backups, etc.)
- SSH hardening settings
- Firewall rules (UFW + CrowdSec)
- All cron jobs (OC crons + system crontab)

**When to use it:**
- "I changed something and now it's broken" → Restore this snapshot
- "OpenClaw update caused issues" → Restore snapshot, then use Tier 2 recovery
- "Starting a risky config change" → Save new snapshot first, then proceed

---

## Quick Reference Commands

```bash
# CrowdSec
cscli decisions list              # Active bans
cscli metrics                     # Log processing stats
cscli decisions delete --ip X.X.X.X   # Unban IP

# Snapshots
restore-snapshot-config.sh save <label>    # Save snapshot
restore-snapshot-config.sh restore <label> # Restore snapshot
restore-snapshot-config.sh list            # List snapshots

# Recovery
restore-snapshot-config.sh restore         # Tier 1: Config only
restore-openclaw-install.sh                # Tier 2: OpenClaw rollback
full-restore.sh                            # Tier 3: Full workspace

# WhatsApp Scope
openclaw status | grep scope               # Check scope
openclaw setup                             # Re-pair if scope missing

# Secret Rotation
rotation-check.py                          # Check rotation schedule
openclaw gateway token --new               # Generate new gateway token

# Post-Recovery
openclaw doctor                            # Health check
openclaw gateway probe                     # Connectivity test
systemctl status crowdsec openclaw-webhook  # Service status
```

---

**Estimated time:** 25-30 minutes

**Next:** Part 6 — Verification (20-point checklist, full system validation)
