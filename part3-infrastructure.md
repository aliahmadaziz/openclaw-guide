# OpenClaw Installation Guide - Part 3: Infrastructure (1 hr)

**Goal:** Set up the supporting infrastructure for a production-ready AI assistant. Google APIs for calendar/email/docs, public webhook endpoint for push notifications, and automated backups.

**Prerequisites:** 
- Part 1 & 2 complete (OpenClaw running, AI persona configured)
- Domain name with DNS access (for Cloudflare tunnel)
- Google account for API access and backup storage

---

## 1. Google OAuth Tokens

Google APIs let your AI read calendar events, send emails, access Drive files, and update spreadsheets. You'll create OAuth credentials and authorize access.

### 1.1 Create Google Cloud Project

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Click **"Select a project"** dropdown → **"New Project"**
3. **Project name:** `openclaw-bot` (or your choice)
4. **Organization:** (leave as "No organization")
5. Click **"Create"**

*Why: Google Cloud project is required to enable APIs and create credentials*

✅ **Verify:** Project appears in project selector dropdown

### 1.2 Enable Required APIs

1. In the Cloud Console, select your new project
2. Go to **"APIs & Services"** → **"Library"**
3. Search and enable each of these APIs (click API name → click **"Enable"**):
   - **Google Calendar API**
   - **Gmail API**
   - **Google Drive API**
   - **Google Sheets API**

*Why: Each API must be explicitly enabled before you can use it*

✅ **Verify:** Go to "APIs & Services" → "Enabled APIs" — all four should be listed

### 1.3 Create OAuth 2.0 Credentials

1. Go to **"APIs & Services"** → **"Credentials"**
2. Click **"Configure Consent Screen"**
   - **User Type:** External
   - Click **"Create"**
3. **OAuth consent screen** form:
   - **App name:** `OpenClaw Bot`
   - **User support email:** your email
   - **Developer contact:** your email
   - Click **"Save and Continue"**
4. **Scopes:** Click **"Add or Remove Scopes"**
   - Search and select:
     - `.../auth/calendar` (full calendar access)
     - `.../auth/gmail.send` (send email)
     - `.../auth/drive.readonly` (read Drive files)
     - `.../auth/spreadsheets` (read/write sheets)
   - Click **"Update"** → **"Save and Continue"**
5. **Test users:** Add your Google email address
   - Click **"Add Users"** → enter your email → **"Add"**
   - Click **"Save and Continue"**
6. Review summary → Click **"Back to Dashboard"**

*Why: OAuth consent screen is required before creating credentials. "External" + test users allows personal use without Google verification.*

Now create the actual credentials:

1. Go to **"Credentials"** tab → Click **"Create Credentials"** → **"OAuth client ID"**
2. **Application type:** Desktop app
3. **Name:** `openclaw-desktop`
4. Click **"Create"**
5. A dialog appears with Client ID and Client Secret
6. Click **"Download JSON"**

*Why: Desktop app type allows server-side OAuth flow. Credentials are downloaded as JSON file.*

✅ **Verify:** You have a file named `client_secret_XXXX.json` downloaded

### 1.4 Install Credentials File

```bash
mkdir -p ~/.clawdbot/google
mv ~/Downloads/client_secret_*.json ~/.clawdbot/google/credentials.json
chmod 600 ~/.clawdbot/google/credentials.json
```

*Why: OpenClaw expects credentials at this exact path. chmod 600 makes file readable only by you.*

✅ **Verify:** `ls -l ~/.clawdbot/google/credentials.json` shows file exists with `-rw-------` permissions

### 1.5 Run OAuth Authorization Flow

Create a simple Python script to generate the token:

```bash
cat > ~/clawd/scripts/google-auth.py << 'EOF'
#!/usr/bin/env python3
import os
from google_auth_oauthlib.flow import InstalledAppFlow
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials

SCOPES = [
    'https://www.googleapis.com/auth/calendar',
    'https://www.googleapis.com/auth/gmail.send',
    'https://www.googleapis.com/auth/drive.readonly',
    'https://www.googleapis.com/auth/spreadsheets'
]

CREDS_PATH = os.path.expanduser('~/.clawdbot/google/credentials.json')
TOKEN_PATH = os.path.expanduser('~/.clawdbot/google/token.json')

def main():
    creds = None
    if os.path.exists(TOKEN_PATH):
        creds = Credentials.from_authorized_user_file(TOKEN_PATH, SCOPES)
    
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file(CREDS_PATH, SCOPES)
            creds = flow.run_local_server(port=8080)
        
        with open(TOKEN_PATH, 'w') as token:
            token.write(creds.to_json())
        print(f"✅ Token saved to {TOKEN_PATH}")
    else:
        print(f"✅ Valid token already exists at {TOKEN_PATH}")

if __name__ == '__main__':
    main()
EOF

chmod +x ~/clawd/scripts/google-auth.py
```

*Why: This script handles the OAuth flow and saves the refresh token for long-term access*

Install required Python libraries:

```bash
apt install -y python3-pip
pip3 install --upgrade google-auth-oauthlib google-auth-httplib2 google-api-python-client
```

Run the authorization:

```bash
cd ~/clawd/scripts
python3 google-auth.py
```

*Why: Script launches browser for Google login, saves token after authorization*

**Follow the prompts:**

*If on a headless VPS (no browser), the script will print a URL. Copy it, open in your local browser, authorize, then paste the redirect URL back into the terminal.*

1. Browser opens (or URL is printed) for Google sign-in
2. Choose your Google account
3. Click **"Continue"** on the unverified app warning (since it's your own app)
4. Review permissions → Click **"Continue"**
5. Browser shows "The authentication flow has completed"

**Headless VPS tip:** If `run_local_server` fails, change it to `run_console()` in the script — this gives you a URL to paste instead of launching a browser.

✅ **Verify:** Script prints "✅ Token saved to ~/.clawdbot/google/token.json"

### 1.6 Test API Access

Quick test to confirm calendar access works:

```bash
cat > ~/clawd/scripts/test-calendar.py << 'EOF'
#!/usr/bin/env python3
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
import datetime

import os
TOKEN_PATH = os.path.expanduser('~/.clawdbot/google/token.json')

creds = Credentials.from_authorized_user_file(TOKEN_PATH)
service = build('calendar', 'v3', credentials=creds)

now = datetime.datetime.utcnow().isoformat() + 'Z'
events_result = service.events().list(
    calendarId='primary', 
    timeMin=now,
    maxResults=3, 
    singleEvents=True,
    orderBy='startTime'
).execute()
events = events_result.get('items', [])

if not events:
    print('No upcoming events found.')
else:
    print('Next 3 events:')
    for event in events:
        start = event['start'].get('dateTime', event['start'].get('date'))
        print(f"  - {start}: {event['summary']}")
EOF

chmod +x ~/clawd/scripts/test-calendar.py
python3 ~/clawd/scripts/test-calendar.py
```

*Why: Verifies token works by fetching your upcoming calendar events*

✅ **Verify:** Script prints your upcoming events (or "No upcoming events found")

### 1.7 Multiple Google Accounts (Optional)

If you use multiple Google accounts (personal, work, etc.), generate separate tokens:

```bash
# Example: generate token-gmail.json for personal account
python3 google-auth.py  # Rename output to token-gmail.json
# Then run again for work account → rename to token-work.json
```

*Why: Each Google account needs its own OAuth token. You reference different tokens in your scripts.*

**Common token naming:**
- `token.json` — Primary/work account
- `token-gmail.json` — Personal Gmail
- `token-work.json` — Work calendar
- `tokens/shopdev.json` — Per-domain tokens

---

## 2. Cloudflare Tunnel

Cloudflare Tunnel exposes your local webhook server (port 8088) via a public HTTPS URL without opening firewall ports. This is required for Google Calendar push notifications and other webhooks.

### 2.1 Install Cloudflared

```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
dpkg -i cloudflared-linux-amd64.deb
```

*Why: Cloudflared creates secure tunnels from your server to Cloudflare's network*

✅ **Verify:** `cloudflared --version` shows version number

### 2.2 Authenticate with Cloudflare

```bash
cloudflared tunnel login
```

*Why: Links cloudflared to your Cloudflare account*

**Follow the prompts:**
1. Browser opens to Cloudflare authorization page
2. Log in to your Cloudflare account
3. Select the domain you want to use (e.g., `yourdomain.com`)
4. Click **"Authorize"**

✅ **Verify:** Terminal shows "You have successfully logged in" and creates `~/.cloudflared/cert.pem`

### 2.3 Create Tunnel

```bash
cloudflared tunnel create openclaw
```

*Why: Creates a named tunnel (persistent connection between your server and Cloudflare)*

✅ **Verify:** Command prints tunnel UUID and creates `~/.cloudflared/TUNNEL-UUID.json`

**Save the tunnel UUID** shown in the output (you'll need it for config).

### 2.4 Configure Tunnel Routing

Create tunnel configuration file:

```bash
cat > ~/.cloudflared/config.yml << 'EOF'
tunnel: TUNNEL_UUID_HERE
credentials-file: /root/.cloudflared/TUNNEL_UUID_HERE.json

ingress:
  - hostname: webhook.YOUR_DOMAIN.com
    service: http://localhost:8088
  - service: http_status:404
EOF
```

*Why: Routes traffic from webhook.yourdomain.com to local port 8088*

**Edit the file to replace:**
- `TUNNEL_UUID_HERE` with the actual UUID from previous step
- `YOUR_DOMAIN.com` with your actual domain

```bash
nano ~/.cloudflared/config.yml
```

### 2.5 Add DNS Record

```bash
cloudflared tunnel route dns openclaw webhook.YOUR_DOMAIN.com
```

*Replace `YOUR_DOMAIN.com` with your actual domain*

*Why: Creates CNAME record pointing webhook.yourdomain.com to the tunnel*

✅ **Verify:** Check Cloudflare DNS dashboard — you should see a CNAME record for `webhook` pointing to `TUNNEL-UUID.cfargotunnel.com`

### 2.6 Test Tunnel (Before Making it a Service)

```bash
cloudflared tunnel run openclaw
```

*Why: Runs tunnel in foreground to test configuration*

**Leave this running** and open a NEW terminal window:

```bash
curl https://webhook.YOUR_DOMAIN.com
```

*Expected: Connection succeeds (even if returns error, that's fine — means tunnel routing works)*

Press `Ctrl+C` in the tunnel terminal to stop.

✅ **Verify:** `curl` doesn't show DNS errors or connection refused

### 2.7 Create Systemd Service

```bash
cat > /etc/systemd/system/cloudflared.service << 'EOF'
[Unit]
Description=Cloudflare Tunnel
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/cloudflared tunnel run openclaw
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF
```

*Why: Runs tunnel automatically on boot, restarts on failure*

Enable and start the service:

```bash
systemctl daemon-reload
systemctl enable cloudflared
systemctl start cloudflared
```

*Why: Starts tunnel now and on every boot*

✅ **Verify:** `systemctl status cloudflared` shows "active (running)"

---

## 3. Webhook Server

The webhook server listens on localhost:8088 and receives push notifications from external services (Google Calendar, email alerts, etc.). You'll create endpoints with random URL paths as security tokens.

### 3.1 Create Webhook Server Script

```bash
cat > ~/clawd/scripts/webhook-server.py << 'EOF'
#!/usr/bin/env python3
"""
Minimal webhook server for OpenClaw
Receives push notifications and forwards to OpenClaw
"""
from aiohttp import web
import json
import os
import logging
from datetime import datetime

# Load secrets from environment
CALENDAR_SECRET = os.getenv('WEBHOOK_CALENDAR_SECRET', 'CHANGE_ME')
EMAIL_SECRET = os.getenv('WEBHOOK_EMAIL_SECRET', 'CHANGE_ME')
ALERT_SECRET = os.getenv('WEBHOOK_ALERT_SECRET', 'CHANGE_ME')

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger('webhook')

async def handle_calendar(request):
    """Handle Google Calendar push notifications"""
    # Verify X-Goog-Resource-State header
    resource_state = request.headers.get('X-Goog-Resource-State')
    
    if resource_state == 'sync':
        # Initial sync verification
        return web.Response(text='OK')
    
    logger.info(f"Calendar notification: {resource_state}")
    
    # TODO: Process calendar change
    # - Fetch changed event from Google Calendar API
    # - Send notification to OpenClaw via message queue
    # - Update local cache
    
    return web.Response(text='OK')

async def handle_email(request):
    """Handle email webhook (e.g., from AgentMail)"""
    try:
        data = await request.json()
        logger.info(f"Email received: {data.get('subject', 'No subject')}")
        
        # TODO: Process email
        # - Parse sender, subject, body
        # - Enqueue for OpenClaw processing
        # - Store in event queue
        
        return web.json_response({'status': 'received'})
    except Exception as e:
        logger.error(f"Email webhook error: {e}")
        return web.Response(status=400, text=str(e))

async def handle_alert(request):
    """Handle generic alerts"""
    try:
        data = await request.json()
        logger.info(f"Alert received: {data.get('type', 'unknown')}")
        
        # TODO: Process alert
        # - Route based on alert type
        # - Send to OpenClaw
        
        return web.json_response({'status': 'received'})
    except Exception as e:
        logger.error(f"Alert webhook error: {e}")
        return web.Response(status=400, text=str(e))

async def handle_health(request):
    """Health check endpoint"""
    return web.json_response({
        'status': 'healthy',
        'timestamp': datetime.utcnow().isoformat()
    })

def main():
    app = web.Application()
    
    # Register endpoints with security tokens
    app.router.add_post(f'/{CALENDAR_SECRET}', handle_calendar)
    app.router.add_post(f'/email/{EMAIL_SECRET}', handle_email)
    app.router.add_post(f'/alerts/{ALERT_SECRET}', handle_alert)
    app.router.add_get('/health', handle_health)
    
    logger.info("Webhook server starting on port 8088")
    web.run_app(app, host='127.0.0.1', port=8088)

if __name__ == '__main__':
    main()
EOF

chmod +x ~/clawd/scripts/webhook-server.py
```

*Why: Receives external webhooks on secure endpoints, processes events, forwards to OpenClaw*

### 3.2 Generate Webhook Secrets

```bash
cat > ~/.clawdbot/webhook.env << EOF
WEBHOOK_CALENDAR_SECRET=$(openssl rand -base64 32 | tr -d '/+=' | cut -c1-22)
WEBHOOK_EMAIL_SECRET=$(openssl rand -base64 32 | tr -d '/+=' | cut -c1-22)
WEBHOOK_ALERT_SECRET=$(openssl rand -base64 32 | tr -d '/+=' | cut -c1-22)
EOF

chmod 600 ~/.clawdbot/webhook.env
```

*Why: Random URL paths act as authentication tokens (security through obscurity for webhooks)*

✅ **Verify:** `cat ~/.clawdbot/webhook.env` shows three random secrets

### 3.3 Install Python Dependencies

```bash
pip3 install aiohttp
```

*Why: aiohttp is an async HTTP server framework for Python*

### 3.4 Test Webhook Server Manually

```bash
source ~/.clawdbot/webhook.env
python3 ~/clawd/scripts/webhook-server.py
```

*Why: Run server in foreground to test configuration*

**In a NEW terminal window:**

```bash
# Load secrets
source ~/.clawdbot/webhook.env

# Test health endpoint
curl http://localhost:8088/health

# Test alert endpoint (with secret)
curl -X POST http://localhost:8088/alerts/$WEBHOOK_ALERT_SECRET \
  -H "Content-Type: application/json" \
  -d '{"type":"test","message":"Hello webhook"}'
```

*Expected: Both commands return JSON responses*

Press `Ctrl+C` in the server terminal to stop.

✅ **Verify:** Curl commands succeed, server logs show requests

### 3.5 Create Systemd Service

```bash
cat > /etc/systemd/system/webhook.service << 'EOF'
[Unit]
Description=OpenClaw Webhook Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/clawd
EnvironmentFile=/root/.clawdbot/webhook.env
ExecStart=/usr/bin/python3 /root/clawd/scripts/webhook-server.py
Restart=on-failure
RestartSec=5s
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF
```

*Why: Runs webhook server as a system service with automatic restart*

Enable and start:

```bash
systemctl daemon-reload
systemctl enable webhook
systemctl start webhook
```

*Why: Starts webhook server now and on every boot*

✅ **Verify:** `systemctl status webhook` shows "active (running)"

### 3.6 Test Public Webhook URL

```bash
source ~/.clawdbot/webhook.env
curl https://webhook.YOUR_DOMAIN.com/health
```

*Replace YOUR_DOMAIN.com with your actual domain*

*Expected: Returns JSON with `"status": "healthy"`*

✅ **Verify:** Public webhook URL is accessible via HTTPS through Cloudflare tunnel

---

## 4. Backup System

Automated backups prevent data loss. You'll set up hourly lightweight backups (git + databases) and nightly full backups (complete workspace + OpenClaw state) to Google Drive.

### 4.1 Install Rclone

```bash
curl https://rclone.org/install.sh | bash
```

*Why: Rclone syncs files to cloud storage (Google Drive, S3, etc.)*

✅ **Verify:** `rclone version` shows version number

### 4.2 Configure Google Drive Remote

```bash
rclone config
```

*Why: Interactive setup to connect rclone to Google Drive*

**Follow the prompts:**

1. `n` (new remote)
2. Name: `gdrive`
3. Storage: `drive` (type the number for Google Drive)
4. Client ID: (press Enter to skip — uses rclone's)
5. Client Secret: (press Enter to skip)
6. Scope: `1` (full access)
7. Root folder ID: (press Enter for root)
8. Service Account File: (press Enter to skip)
9. Edit advanced config? `n`
10. Use auto config? `y`
11. Browser opens → log in to Google → authorize rclone
12. Configure as team drive? `n`
13. Confirm: `y`
14. Quit: `q`

✅ **Verify:** `rclone lsd gdrive:` lists your Google Drive folders

### 4.3 Configure Encrypted Remote (Optional but Recommended)

Encrypt backups before uploading to cloud:

```bash
rclone config
```

**Follow the prompts:**

1. `n` (new remote)
2. Name: `gdrive-crypt`
3. Storage: `crypt` (type the number)
4. Remote: `gdrive:openclaw-backups` (creates encrypted folder)
5. Filename encryption: `standard`
6. Directory name encryption: `true`
7. Password: (enter a strong password — **SAVE THIS**)
8. Salt password: (enter another password or press Enter to skip)
9. Confirm: `y`
10. Quit: `q`

*Why: Encrypts all backup files before upload. Even if Google Drive is compromised, backups are secure.*

✅ **Verify:** `rclone lsd gdrive-crypt:` shows empty list (new remote)

**⚠️ SAVE YOUR ENCRYPTION PASSWORD** — losing it means losing all backup access.

### 4.4 Create Hourly Backup Script

```bash
cat > ~/clawd/backup.sh << 'EOF'
#!/bin/bash
# Hourly backup: git push + rclone databases
set -e

cd /root/clawd

# Git backup
git add -A
git commit -m "Hourly backup $(date -u +%Y-%m-%d_%H:%M:%S)" || true
git push origin main || echo "Git push failed (non-fatal)"

# Backup SQLite databases to Google Drive
rclone copy ~/.clawdbot/*.sqlite gdrive-crypt:openclaw-backups/databases/ \
  --include "*.sqlite" \
  --log-file=/var/log/backup-hourly.log

echo "✅ Hourly backup complete: $(date -u)"
EOF

chmod +x ~/clawd/backup.sh
```

*Why: Frequent lightweight backups capture work-in-progress (memory logs, databases)*

### 4.5 Create Nightly Backup Script

```bash
cat > ~/clawd/scripts/nightly-backup.sh << 'EOF'
#!/bin/bash
# Nightly backup: full workspace + OpenClaw state
set -e

BACKUP_DATE=$(date -u +%Y-%m-%d)
BACKUP_DIR="/tmp/openclaw-backup-$BACKUP_DATE"

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Backup workspace
cp -r /root/clawd "$BACKUP_DIR/workspace"

# Backup OpenClaw config
cp ~/.config/openclaw/openclaw.json "$BACKUP_DIR/openclaw-config.json" || true

# Backup credentials
cp -r ~/.clawdbot "$BACKUP_DIR/clawdbot" || true

# Backup Cloudflare tunnel config
cp -r ~/.cloudflared "$BACKUP_DIR/cloudflared" || true

# Create archive
cd /tmp
tar -czf "openclaw-backup-$BACKUP_DATE.tar.gz" "openclaw-backup-$BACKUP_DATE"

# Upload to Google Drive with 7-day versioning
rclone copy "openclaw-backup-$BACKUP_DATE.tar.gz" gdrive-crypt:openclaw-backups/full/ \
  --backup-dir gdrive-crypt:openclaw-backups/versions/$BACKUP_DATE \
  --log-file=/var/log/backup-nightly.log

# Cleanup local backup
rm -rf "$BACKUP_DIR" "openclaw-backup-$BACKUP_DATE.tar.gz"

# Delete backups older than 30 days
rclone delete gdrive-crypt:openclaw-backups/full/ --min-age 30d

echo "✅ Nightly backup complete: $BACKUP_DATE"
EOF

chmod +x ~/clawd/scripts/nightly-backup.sh
```

*Why: Complete system snapshot for disaster recovery. 7-day versioning allows rollback to any recent state.*

### 4.6 Initialize Git Repository (If Not Already)

```bash
cd ~/clawd
git init
git config user.email "bot@yourdomain.com"
git config user.name "OpenClaw Bot"
git add -A
git commit -m "Initial commit"
```

*Why: Git tracks all workspace changes, provides version history*

**Optional:** Push to GitHub for off-site redundancy:

```bash
# Create private repo on GitHub, then:
git remote add origin git@github.com:YOUR_USERNAME/clawd-backup.git
git push -u origin main
```

*Why: GitHub provides additional backup layer and remote access to workspace*

### 4.7 Test Backups Manually

```bash
# Test hourly backup
~/clawd/backup.sh

# Test nightly backup
~/clawd/scripts/nightly-backup.sh
```

*Why: Verify both scripts work before scheduling*

✅ **Verify:** Scripts complete without errors, check Google Drive for uploaded files:

```bash
rclone ls gdrive-crypt:openclaw-backups/databases/
rclone ls gdrive-crypt:openclaw-backups/full/
```

### 4.8 Schedule Backups in Crontab

```bash
crontab -e
```

*Select your editor (nano recommended for beginners), then add these lines:*

```cron
# Hourly backup (git + databases)
0 * * * * /root/clawd/backup.sh >> /var/log/backup-hourly.log 2>&1

# Nightly backup (full system) at 4 AM UTC
0 4 * * * /root/clawd/scripts/nightly-backup.sh >> /var/log/backup-nightly.log 2>&1
```

*Why: Hourly captures frequent changes, nightly creates restore points*

Save and exit (Ctrl+X, Y, Enter in nano).

✅ **Verify:** `crontab -l` shows both backup jobs

### 4.9 Monitor Backup Logs

```bash
# View hourly backup log
tail -f /var/log/backup-hourly.log

# View nightly backup log
tail -f /var/log/backup-nightly.log
```

*Press Ctrl+C to exit tail*

✅ **Verify:** Wait for next hour, check logs show successful run

---

## Part 3 Complete ✅

You should now have:

**Google OAuth:**
- [ ] Google Cloud project created
- [ ] Calendar, Gmail, Drive, Sheets APIs enabled
- [ ] OAuth consent screen configured (External, test users added)
- [ ] Desktop app credentials created and downloaded
- [ ] credentials.json installed at `~/.clawdbot/google/credentials.json`
- [ ] OAuth flow completed, token.json generated
- [ ] Test script confirms calendar access works
- [ ] (Optional) Multiple account tokens for different Google accounts

**Cloudflare Tunnel:**
- [ ] cloudflared installed
- [ ] Authenticated with Cloudflare account
- [ ] Tunnel created (`cloudflared tunnel create openclaw`)
- [ ] DNS CNAME record added (webhook.yourdomain.com)
- [ ] Tunnel config routes domain to localhost:8088
- [ ] Systemd service running tunnel on boot
- [ ] Public webhook URL accessible via HTTPS

**Webhook Server:**
- [ ] webhook-server.py created with endpoints for calendar/email/alerts
- [ ] Webhook secrets generated in ~/.clawdbot/webhook.env
- [ ] aiohttp Python library installed
- [ ] Manual test confirms server receives requests
- [ ] Systemd service runs webhook server on boot
- [ ] Public webhook URL returns health check response

**Backup System:**
- [ ] rclone installed
- [ ] Google Drive remote configured (`gdrive`)
- [ ] (Optional) Encrypted remote configured (`gdrive-crypt`)
- [ ] Git repository initialized in ~/clawd
- [ ] backup.sh created (hourly: git + databases)
- [ ] nightly-backup.sh created (full system + 7-day versioning)
- [ ] Manual test backups succeeded
- [ ] Both backups uploaded to Google Drive
- [ ] Crontab scheduled: hourly at :00, nightly at 04:00 UTC
- [ ] Backup logs created and monitored

**Next:** Part 4 — Automation (cron jobs, event queue, reminders, heartbeats)

---

## Quick Reference Commands

```bash
# Google OAuth
python3 ~/clawd/scripts/google-auth.py              # Reauthorize
python3 ~/clawd/scripts/test-calendar.py            # Test API access

# Cloudflare Tunnel
systemctl status cloudflared                        # Check tunnel status
systemctl restart cloudflared                       # Restart tunnel
cloudflared tunnel list                             # List tunnels
journalctl -u cloudflared -f                        # View tunnel logs

# Webhook Server
systemctl status webhook                            # Check webhook status
systemctl restart webhook                           # Restart webhook
journalctl -u webhook -f                            # View webhook logs
curl http://localhost:8088/health                   # Test local endpoint

# Backups
~/clawd/backup.sh                                   # Manual hourly backup
~/clawd/scripts/nightly-backup.sh                   # Manual nightly backup
rclone ls gdrive-crypt:openclaw-backups/            # List backups
tail -f /var/log/backup-hourly.log                  # Monitor hourly backups
tail -f /var/log/backup-nightly.log                 # Monitor nightly backups
crontab -l                                          # View scheduled backups

# System Health
systemctl status cloudflared webhook                # Check both services
journalctl --since "1 hour ago" -u webhook          # Recent webhook logs
ls -lh ~/.clawdbot/google/                          # Check tokens exist
```

---

## Troubleshooting

**Google OAuth "invalid_client" error:**
- Verify credentials.json is correctly downloaded
- Check file path: `~/.clawdbot/google/credentials.json`
- Ensure OAuth consent screen is configured (test users added)
- Re-download credentials from Google Cloud Console

**Google OAuth "access_denied" error:**
- Make sure your email is added as a test user
- Check that all required scopes are enabled
- Try using a different Google account

**Cloudflare tunnel connection refused:**
- Check tunnel is running: `systemctl status cloudflared`
- Verify config.yml has correct tunnel UUID
- Ensure DNS CNAME record exists (check Cloudflare dashboard)
- Test local service first: `curl http://localhost:8088/health`

**Webhook server not receiving requests:**
- Check service is running: `systemctl status webhook`
- Verify tunnel routes to correct port (8088)
- Test locally before testing publicly
- Check logs: `journalctl -u webhook -f`
- Ensure webhook.env secrets are loaded

**Webhook "Connection refused" on public URL:**
- Verify Cloudflare tunnel is running
- Check webhook server is listening on localhost:8088
- Test health endpoint: `curl https://webhook.yourdomain.com/health`
- Check both services: `systemctl status cloudflared webhook`

**Rclone "Failed to create file system" error:**
- Re-run `rclone config` to reconfigure remote
- Test authentication: `rclone lsd gdrive:`
- Check Google Drive permissions (authorize in browser)
- For crypt errors: verify encryption password is correct

**Backups not running from cron:**
- Check crontab exists: `crontab -l`
- Verify script has execute permissions: `ls -l ~/clawd/backup.sh`
- Check cron logs: `grep CRON /var/log/syslog`
- Ensure absolute paths in cron jobs (not relative)
- Test scripts manually first

**Backup script errors "command not found":**
- Cron has minimal PATH — use absolute paths (`/usr/bin/rclone`)
- Add `PATH=/usr/local/bin:/usr/bin:/bin` at top of script
- Or modify crontab: `PATH=/usr/local/bin:/usr/bin:/bin` before job

**Git push fails in backup:**
- Check remote is configured: `git remote -v`
- Verify SSH keys or HTTPS credentials
- Backup script continues even if git fails (non-fatal)

---

## Security Notes

**Webhook URL Secrets:**
- Never commit webhook.env to git
- Rotate secrets periodically: regenerate in webhook.env, restart service
- Each endpoint has its own secret — limits blast radius if one leaks

**Google OAuth Tokens:**
- token.json contains refresh token — treat as password
- Never share or commit to public repos
- Rotate if compromised: revoke in Google Cloud Console → reauthorize

**Cloudflare Tunnel:**
- Tunnel authenticates via cert.pem — keep this file secure
- Only exposes localhost:8088 — no other ports accessible
- Uses HTTPS automatically — no certificate management needed

**Backup Encryption:**
- SAVE YOUR RCLONE CRYPT PASSWORD — no recovery without it
- Consider using rclone obscure for password storage in scripts
- Test restore process regularly to verify encryption works

---

## Webhook URLs Reference

After setup, you'll have these public endpoints (replace YOUR_DOMAIN and secrets):

| Purpose | URL Format | HTTP Method |
|---------|------------|-------------|
| Health Check | `https://webhook.YOUR_DOMAIN.com/health` | GET |
| Google Calendar | `https://webhook.YOUR_DOMAIN.com/CALENDAR_SECRET` | POST |
| Email Notifications | `https://webhook.YOUR_DOMAIN.com/email/EMAIL_SECRET` | POST |
| Generic Alerts | `https://webhook.YOUR_DOMAIN.com/alerts/ALERT_SECRET` | POST |

**To get actual URLs:**

```bash
source ~/.clawdbot/webhook.env
echo "Calendar: https://webhook.YOUR_DOMAIN.com/$WEBHOOK_CALENDAR_SECRET"
echo "Email: https://webhook.YOUR_DOMAIN.com/email/$WEBHOOK_EMAIL_SECRET"
echo "Alerts: https://webhook.YOUR_DOMAIN.com/alerts/$WEBHOOK_ALERT_SECRET"
```

*Use these URLs when configuring external services (Google Calendar watch, AgentMail, monitoring alerts)*

---

**Estimated time:** 50-70 minutes (excluding OAuth browser flows and initial backup uploads)
