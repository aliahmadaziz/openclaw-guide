# OpenClaw Installation Guide - Part 6: Verification (15 min)

**Goal:** Validate your entire OpenClaw setup end-to-end. This is the final checkpoint.

**Prerequisites:** 
- Parts 1-5 completed
- Server is running
- WhatsApp is paired

---

## 1. Automated Verification Script

We've built a comprehensive verification script that checks all 20 critical points from Parts 1-5.

### 1.1 Run the Verification

```bash
cd /root/clawd
python3 scripts/verify-setup.py
```

**What it checks:**

| Check | Component | What's Validated |
|-------|-----------|------------------|
| **Part 1: Base Install** | | |
| 1 | SSH | Password auth disabled |
| 2 | Firewall | UFW active, only SSH allowed |
| 3 | CrowdSec | Service + bouncer running |
| 4 | Swap | 2GB active |
| **Part 2: AI Assistant** | | |
| 5 | Node.js | v22.x installed |
| 6 | OpenClaw | Binary installed |
| 7 | Gateway | Service running |
| 8 | WhatsApp | Channel configured |
| 9 | Device scope | operator.read + operator.write |
| 10 | Workspace | Core .md files exist |
| 11 | Models | Primary + fallback configured |
| **Part 3: Infrastructure** | | |
| 12 | Google OAuth | token.json exists |
| 13 | Cloudflare | Tunnel service running |
| 14 | Webhook server | Service running, health OK |
| 15 | Public webhook | HTTPS accessible |
| **Part 4: Automation** | | |
| 16 | Backup crons | Hourly + nightly scheduled |
| 17 | Rclone | Google Drive remote configured |
| 18 | OC Crons | At least 1 cron job |
| 19 | Event queue | SQLite DB exists |
| 20 | Heartbeat | HEARTBEAT.md exists |
| **Part 5: Hardening** | | |
| 21 | Snapshots | At least 1 config snapshot |

### 1.2 Expected Output

```
============================================================
           OpenClaw Setup Verification
============================================================

Timestamp: 2026-02-22 16:30:00 UTC

Part 1: Base Install
✅ SSH: Password auth disabled
✅ Firewall: UFW active, SSH only
✅ CrowdSec: Running + bouncer active
✅ Swap: 2GB active

Part 2: AI Assistant
✅ Node.js: v22.x installed
✅ OpenClaw: Installed
✅ OpenClaw: Gateway running
✅ WhatsApp: Connected
✅ Device scope: operator.read + operator.write
✅ Workspace: Core .md files exist
✅ Models: Primary + fallback configured

Part 3: Infrastructure
✅ Google OAuth: token.json exists
✅ Cloudflare tunnel: Running
✅ Webhook server: Running
✅ Public webhook: Accessible

Part 4: Automation
✅ Hourly + nightly crons
✅ Rclone: Google Drive configured
✅ OC Crons: At least 1 configured
✅ Event queue: SQLite DB exists
✅ Heartbeat: HEARTBEAT.md exists

Part 5: Hardening
✅ Config snapshot: At least 1 saved

============================================================
                      Summary
============================================================

Passed: 21/21 (100.0%)

🎉 ALL CHECKS PASSED! Your OpenClaw setup is complete.
```

✅ **Verify:** All checks show ✅ green checkmarks

---

## 2. Manual End-to-End Test

The automated script validates the infrastructure. Now let's validate the **bot responds**.

### 2.1 Send a Test Message

From your phone (the one paired with WhatsApp):

1. Open WhatsApp
2. Send a message to your bot number:
   ```
   Hey, are you alive?
   ```

3. Wait 5-10 seconds

✅ **Verify:** Bot responds with a natural reply

### 2.2 Test a Command

Send:
```
What's the current UTC time?
```

✅ **Verify:** Bot responds with the correct time

### 2.3 Test File Access

Send:
```
Read SOUL.md and tell me what your primary directive is
```

✅ **Verify:** Bot reads the file and summarizes

**If any test fails**, proceed to the Troubleshooting section below.

---

## 3. Troubleshooting Reference

Quick lookup for common issues across all parts.

### 3.1 Quick Reference Table

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| **Can't SSH to server** | SSH service not running | `systemctl restart sshd` |
| **"Permission denied (publickey)"** | SSH key not added to server | Re-add key: `ssh-copy-id root@SERVER_IP` |
| **Gateway won't start** | Port conflict or bad config | Check logs: `journalctl -u openclaw-gateway -n 50` |
| **WhatsApp disconnected** | QR code expired or phone offline | Re-pair: `openclaw gateway restart`, scan QR |
| **Bot doesn't respond** | Gateway stopped or model API issue | Check status: `systemctl status openclaw-gateway` |
| **"Model rate limit"** | Too many requests | Wait 60s, or switch to fallback model |
| **Webhook not receiving** | Tunnel down or server crashed | Restart: `systemctl restart cloudflared moltbot-webhook` |
| **Backup not running** | Cron misconfigured | Check: `crontab -l`, verify paths |
| **Token expired** | Google OAuth needs refresh | Re-auth: `scripts/google-reauth.py` |
| **"Permission denied" errors** | Wrong file ownership | Fix: `chown -R root:root /root/clawd` |

### 3.2 Top 10 Common Issues

#### 1. WhatsApp Keeps Disconnecting
**Symptom:** Bot works for a few hours, then stops responding.

**Cause:** Phone went offline, or WhatsApp logged out.

**Fix:**
```bash
# Check gateway logs
journalctl -u openclaw-gateway -n 100 | grep -i whatsapp

# If disconnected, restart and re-pair
openclaw gateway restart
```
*Keep your phone online and WhatsApp open during initial setup.*

---

#### 2. Gateway Fails to Start
**Symptom:** `systemctl status openclaw-gateway` shows "failed" or "inactive (dead)"

**Cause:** Usually bad `openclaw.json` syntax or missing API key.

**Fix:**
```bash
# Check the exact error
journalctl -u openclaw-gateway -n 50

# Validate JSON syntax
cat ~/.clawdbot/openclaw.json | jq .

# If syntax error, edit and fix
nano ~/.clawdbot/openclaw.json

# Restart
systemctl restart openclaw-gateway
```

---

#### 3. "Model not found" or API Errors
**Symptom:** Bot responds with "Model error" or doesn't respond at all.

**Cause:** Invalid API key, wrong model name, or API service down.

**Fix:**
```bash
# Check logs for specific error
journalctl -u openclaw-gateway -n 50 | grep -i error

# Verify API key is set correctly in openclaw.json
cat ~/.clawdbot/openclaw.json | jq '.model, .fallbackModel'

# Test API key manually (for Anthropic)
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: YOUR_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-sonnet-4-5","max_tokens":10,"messages":[{"role":"user","content":"hi"}]}'
```

---

#### 4. Webhook Returns 404 or 502
**Symptom:** Webhook URL returns error when visited.

**Cause:** Cloudflare tunnel not running, or webhook server crashed.

**Fix:**
```bash
# Check both services
systemctl status cloudflared
systemctl status moltbot-webhook

# Check tunnel config
cat ~/.cloudflared/config.yml

# Restart both
systemctl restart cloudflared
systemctl restart moltbot-webhook

# Test local webhook
curl http://127.0.0.1:8088/health
# Should return: {"status":"ok"}
```

---

#### 5. Backups Not Running
**Symptom:** No recent backups in Google Drive, or cron emails about failures.

**Cause:** Cron not scheduled, rclone not configured, or script permissions wrong.

**Fix:**
```bash
# Check cron is scheduled
crontab -l | grep backup

# Test hourly backup manually
/root/clawd/backup.sh

# Test nightly backup manually
/root/clawd/scripts/nightly-backup.sh

# Check rclone config
rclone listremotes
# Should show: gdrive:

# Check recent backup files
ls -lh /root/openclaw-backups/
```

---

#### 6. Google OAuth Token Expired
**Symptom:** Calendar sync fails, or "invalid_grant" errors.

**Cause:** Token expired or revoked.

**Fix:**
```bash
# Re-authenticate
cd /root/clawd
python3 scripts/google-reauth.py

# Follow the OAuth flow in browser
# New token will be saved to ~/.clawdbot/google/token.json
```

---

#### 7. Firewall Blocking Legitimate Traffic
**Symptom:** Can't access webhook, or bot can't reach external APIs.

**Cause:** UFW rule too restrictive.

**Fix:**
```bash
# Check current rules
ufw status numbered

# Allow outbound traffic (should be default, but verify)
ufw default allow outgoing

# If you need to allow a specific inbound port:
ufw allow PORT_NUMBER/tcp

# Reload
ufw reload
```
*Never open unnecessary inbound ports. Only SSH should be allowed.*

---

#### 8. Out of Disk Space
**Symptom:** Logs say "no space left on device", services fail.

**Cause:** Logs filling disk, or too many backups.

**Fix:**
```bash
# Check disk usage
df -h

# Find large directories
du -sh /var/log/* | sort -h
du -sh /root/* | sort -h

# Clear old logs
journalctl --vacuum-time=7d

# Delete old backups (keep last 7 days)
find /root/openclaw-backups/ -type f -mtime +7 -delete
```

---

#### 9. Node.js Version Mismatch
**Symptom:** OpenClaw install fails, or runtime errors.

**Cause:** Wrong Node.js version installed.

**Fix:**
```bash
# Check current version
node --version

# If not v22.x, reinstall
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt-get install -y nodejs

# Verify
node --version
# Should show: v22.x.x
```

---

#### 10. CrowdSec Blocking Legitimate IPs
**Symptom:** Can't SSH from your IP, or bot's outbound requests blocked.

**Cause:** CrowdSec false positive.

**Fix:**
```bash
# Check decisions (active bans)
cscli decisions list

# If your IP is banned, unban it
cscli decisions delete --ip YOUR_IP

# Whitelist your IP permanently
cscli decisions add --ip YOUR_IP --duration 999999h --type captcha

# Or disable bouncer temporarily
systemctl stop crowdsec-firewall-bouncer
```

---

## 4. Full Config Reference

Here's an annotated `openclaw.json` showing all key fields. **Do not copy values verbatim** — this is a structure reference.

### 4.1 Complete Structure

```json
{
  "model": "anthropic/claude-opus-4-6",
  "fallbackModel": "google/gemini-2.5-pro",
  "contextTokens": 200000,
  
  "deviceScope": [
    "operator.read",
    "operator.write"
  ],
  
  "workspaceRoot": "/root/clawd",
  
  "channels": [
    {
      "type": "whatsapp",
      "sessionName": "default",
      "autoReconnect": true
    }
  ],
  
  "heartbeat": {
    "enabled": true,
    "intervalMs": 3600000,
    "checkPath": "/root/clawd/HEARTBEAT.md"
  },
  
  "crons": [
    {
      "name": "morning-brief",
      "schedule": "0 9 * * *",
      "command": "python3 /root/clawd/scripts/morning-brief.py",
      "deliverTo": "whatsapp",
      "model": "claude-sonnet-4-0"
    }
  ],
  
  "safety": {
    "maxTokensPerRequest": 4096,
    "rateLimitPerMinute": 60
  },
  
  "systemPrompt": {
    "files": [
      "/root/clawd/AGENTS.md",
      "/root/clawd/SOUL.md"
    ]
  }
}
```

### 4.2 Key Field Explanations

| Field | Purpose | Example Value |
|-------|---------|---------------|
| `model` | Primary AI model | `anthropic/claude-opus-4-6` |
| `fallbackModel` | Backup if primary fails | `google/gemini-2.5-pro` |
| `contextTokens` | Max context window | `200000` (200K) |
| `deviceScope` | Permissions for OpenClaw | `["operator.read", "operator.write"]` |
| `workspaceRoot` | Agent's home directory | `/root/clawd` |
| `channels` | Communication platforms | WhatsApp, Discord, Telegram, etc. |
| `heartbeat.enabled` | Auto-check-in system | `true` |
| `heartbeat.intervalMs` | Time between heartbeats | `3600000` (1 hour) |
| `crons` | Scheduled tasks | Morning brief, standup, etc. |
| `safety.maxTokensPerRequest` | Token limit per response | `4096` |
| `systemPrompt.files` | Files injected into every prompt | `AGENTS.md`, `SOUL.md` |

### 4.3 Model Configuration

**Recommended Tiers:**

| Use Case | Model | Context | Cost |
|----------|-------|---------|------|
| Main session | `claude-opus-4-6` | 200K | High |
| Sub-agents | `claude-sonnet-4-5` | 200K | Medium |
| Cron jobs | `claude-sonnet-4-0` | 200K | Low |
| Fallback | `google/gemini-2.5-pro` | 1M+ | Low |

**Important:** Set `contextTokens` to the **smallest** context window in your chain. If your fallback has 200K but primary has 1M, use `200000`.

---

## 5. What's Next?

You've built the infrastructure. Now make it **yours**.

### 5.1 Customize Your Persona

Edit `/root/clawd/SOUL.md` to define your bot's personality:

```bash
nano /root/clawd/SOUL.md
```

**Suggested sections:**
- **Core directive:** What's your bot's primary purpose?
- **Tone:** Formal? Casual? Sarcastic?
- **Boundaries:** What should it refuse to do?
- **Expertise:** What domains is it expert in?

**Example:**
```markdown
# SOUL.md

## Primary Directive
I am Moltres, Ali's executive assistant. My purpose is to:
1. Manage his calendar and priorities
2. Handle research and data gathering
3. Draft communications
4. Monitor systems and alert on issues

## Tone
Professional but warm. I use first person ("I checked the calendar").
Direct and concise — no fluff.

## Boundaries
I do not:
- Make financial decisions without confirmation
- Send external messages without review
- Delete data without asking
```

### 5.2 Build Daily Rhythms

Create routines using cron jobs:

**Morning Brief (9am UTC):**
```python
# /root/clawd/scripts/morning-brief.py
# - Check calendar for today
# - Summarize unread emails
# - Review yesterday's memory log
# - Send via WhatsApp
```

**Standup (10am UTC, weekdays only):**
```python
# /root/clawd/scripts/standup.py
# - List today's scheduled tasks
# - Check active projects
# - Suggest priorities
```

**Evening Wrap (6pm UTC):**
```python
# /root/clawd/scripts/evening-wrap.py
# - Summarize what happened today
# - Write to daily memory log
# - Prep tomorrow's agenda
```

Add to `openclaw.json`:
```json
"crons": [
  {
    "name": "morning-brief",
    "schedule": "0 9 * * *",
    "command": "python3 /root/clawd/scripts/morning-brief.py",
    "deliverTo": "whatsapp"
  },
  {
    "name": "standup",
    "schedule": "0 10 * * 1-5",
    "command": "python3 /root/clawd/scripts/standup.py",
    "deliverTo": "whatsapp"
  },
  {
    "name": "evening-wrap",
    "schedule": "0 18 * * *",
    "command": "python3 /root/clawd/scripts/evening-wrap.py",
    "deliverTo": "whatsapp"
  }
]
```

### 5.3 Email Integration

**Option 1: AgentMail (Dead Simple)**
1. Sign up at [agentmail.to](https://agentmail.to)
2. Get your bot email: `yourname@agentmail.to`
3. Configure webhook in AgentMail settings
4. Point to: `https://webhook.poweker.com/agentmail/YOUR_SECRET`
5. Bot receives emails, processes via event queue

**Option 2: Gmail API (Full Control)**
1. Already set up if you completed Part 3
2. Use `scripts/gmail-check.py` to poll inbox
3. Run via cron every 15 minutes
4. Process new emails, summarize, alert

### 5.4 Create Custom Skills

Skills are reusable tools your bot can invoke.

**Example: Weather Lookup**

```python
# /root/clawd/skills/weather.py
import requests
import sys

def get_weather(city):
    api_key = "YOUR_OPENWEATHER_API_KEY"
    url = f"http://api.openweathermap.org/data/2.5/weather?q={city}&appid={api_key}"
    response = requests.get(url)
    data = response.json()
    
    temp_k = data['main']['temp']
    temp_c = temp_k - 273.15
    description = data['weather'][0]['description']
    
    return f"{city}: {temp_c:.1f}°C, {description}"

if __name__ == "__main__":
    city = sys.argv[1] if len(sys.argv) > 1 else "Karachi"
    print(get_weather(city))
```

**How to use:**
1. Save to `/root/clawd/skills/weather.py`
2. Make executable: `chmod +x /root/clawd/skills/weather.py`
3. Document in `/root/clawd/TOOLS.md`:
   ```markdown
   ## Weather
   `python3 skills/weather.py "City Name"`
   Returns current weather for any city.
   ```
4. Bot can now call it: `python3 skills/weather.py "London"`

### 5.5 Join the Community

Get help, share your setup, contribute back:

- **GitHub:** [openclaw/openclaw](https://github.com/openclaw/openclaw)
- **Discord:** [discord.gg/openclaw](https://discord.gg/openclaw)
- **Forum:** [discuss.openclaw.dev](https://discuss.openclaw.dev)

**Ways to contribute:**
- Share your custom skills
- Report bugs or suggest features
- Write guides for specific integrations
- Help other users troubleshoot

---

## 6. 🎉 Setup Complete!

**You did it.** You've built a production-grade AI assistant from scratch.

### What You've Accomplished

✅ **Hardened VPS** — SSH lockdown, firewall, intrusion detection  
✅ **AI Brain** — GPT-4-class model with fallback  
✅ **WhatsApp Interface** — Chat with your bot anytime, anywhere  
✅ **Calendar Integration** — Google Calendar sync, reminders, red-flagging  
✅ **Email Routing** — Webhook-based email processing  
✅ **Automated Backups** — Hourly code, nightly full system  
✅ **Self-Monitoring** — Heartbeats, health checks, event queue  
✅ **Recovery System** — Config snapshots, rollback scripts  

### Your Bot Can Now

- **Read and write files** in its workspace
- **Search the web** for real-time information
- **Manage your calendar** (create, edit, alert)
- **Process emails** and draft replies
- **Run shell commands** (safely, with oversight)
- **Schedule tasks** via cron
- **Monitor itself** and alert on failures
- **Recover from crashes** automatically

### The Real Work Starts Now

The infrastructure is done. Now you teach it **how you work**.

- Fill `USER.md` with your preferences, quirks, pet peeves
- Log everything in daily memory files
- Give feedback when it misses the mark
- Build routines that match your day

**This is not a tool. It's a teammate.**

Treat it like one. It'll learn your style, anticipate your needs, handle the boring stuff, and give you hours back.

---

## Final Checklist

Before you close this guide:

- [ ] All 21 verification checks passed
- [ ] Bot responds to WhatsApp messages
- [ ] `SOUL.md` reflects your bot's purpose
- [ ] At least one cron job configured
- [ ] Backups confirmed in Google Drive
- [ ] You know how to check logs: `journalctl -u openclaw-gateway -n 50`
- [ ] You bookmarked the troubleshooting table
- [ ] You know how to restore a snapshot: `scripts/restore-snapshot-config.sh`

---

## Support & Resources

**Documentation:**
- Full OpenClaw docs: [docs.openclaw.ai](https://docs.openclaw.ai)
- Source: [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)
- Community: [Discord](https://discord.com/invite/clawd)
- Skills: [clawhub.com](https://clawhub.com)

**Quick Commands:**
```bash
# Check gateway status
systemctl status openclaw-gateway

# Restart gateway
systemctl restart openclaw-gateway

# View recent logs
journalctl -u openclaw-gateway -n 100 -f

# Re-run verification
python3 /root/clawd/scripts/verify-setup.py

# Restore config snapshot
/root/clawd/scripts/restore-snapshot-config.sh

# Test webhook health
curl http://127.0.0.1:8088/health
```

**Emergency Recovery:**
```bash
# If everything is broken:
cd /root/clawd
git pull
systemctl restart openclaw-gateway

# If config is corrupted:
/root/clawd/scripts/restore-snapshot-config.sh

# If OpenClaw won't start:
journalctl -u openclaw-gateway -n 100
# Fix the error shown, then restart
```

---

**Thank you for building with OpenClaw.**

Now go build something incredible. 🚀

---

*Last updated: 2026-02-22*  
*Guide version: 1.0*  
*OpenClaw version: Latest stable*
