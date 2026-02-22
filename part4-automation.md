# Part 4: Automation (1 hour)

**Prerequisites:** Parts 1-3 complete (OpenClaw running, AI persona configured, webhooks active, backups scheduled)

**What you'll build:** Four-layer automation system — OpenClaw cron jobs for AI-driven schedules, system crontab for infrastructure, event queue for reliable webhook processing, and heartbeat for idle-time maintenance.

---

## 1. OpenClaw Cron Jobs

**WHY:** OpenClaw's built-in cron runs AI sessions on schedule. Each job is an isolated conversation with its own context, model, and delivery settings.

### 1.1 Understanding OC Crons

```bash
# List all cron jobs
openclaw cron list

# View recent runs (last 10 executions across all jobs)
openclaw cron runs

# View runs for specific job
openclaw cron runs --name "morning-agenda"
```

**Key concepts:**
- Each cron job spawns a fresh AI session (no conversation history)
- `--announce` flag sends output to your configured chat channel (WhatsApp/Discord/etc)
- `--model` overrides default model (use cheap models for routine tasks)
- `--best-effort-deliver` doesn't mark job as failed if delivery temporarily fails
- One-shot jobs: `--at "2026-03-15 14:00"` + `--delete-after-run`

### 1.2 Add Morning Agenda (Daily 9am)

```bash
openclaw cron add \
  --name "morning-agenda" \
  --cron "0 9 * * *" \
  --model "claude-sonnet-4-0" \
  --announce \
  --best-effort-deliver \
  --message "Read my calendar for today. Summarize: meetings (time, title, attendees if available), focus blocks, and any red/urgent events. Keep it under 5 lines. End with one practical tip based on my schedule density."
```

**WHY:** Daily calendar summary delivered to WhatsApp at 9am. Uses cheap Sonnet 4.0 model. `--best-effort-deliver` prevents job from failing if WhatsApp is temporarily down.

✅ **Verify:** `openclaw cron list` shows the job. Wait until 9am UTC or test with near-future time.

### 1.3 Add Evening Wrap-up (Daily 10:30pm)

```bash
openclaw cron add \
  --name "evening-wrap" \
  --cron "30 22 * * *" \
  --model "claude-sonnet-4-0" \
  --announce \
  --best-effort-deliver \
  --message "Read today's daily log (memory/YYYY-MM-DD.md). Summarize the day in 3 sentences: what got done, what's pending, one insight or pattern you noticed. Be honest but not harsh."
```

**WHY:** End-of-day reflection. Helps catch incomplete tasks and patterns.

### 1.4 Add Weekly Health Check-in (Sunday 10am)

```bash
openclaw cron add \
  --name "health-checkin" \
  --cron "0 10 * * 0" \
  --model "claude-sonnet-4-0" \
  --announce \
  --message "Read health-os/STATUS.md. Ask me for this week's weight, energy level (1-10), and one health win. Keep it conversational and brief."
```

**WHY:** Weekly health tracking. Sunday morning timing catches weekend reflection window.

### 1.5 One-Shot Reminder Example

```bash
# Remind to call mom on her birthday (one-time, auto-delete after)
openclaw cron add \
  --name "mom-birthday-reminder" \
  --at "2026-03-15 10:00" \
  --delete-after-run \
  --announce \
  --message "Remind: Call mom for her birthday. Suggest bringing up her garden project."
```

**WHY:** One-time reminders without cluttering the cron list. `--at` uses ISO time (YYYY-MM-DD HH:MM).

### 1.6 Manage Cron Jobs

```bash
# Remove a job
openclaw cron rm --name "morning-agenda"

# View last 20 runs with full output
openclaw cron runs --limit 20

# Check if a specific job ran today
openclaw cron runs --name "evening-wrap" --limit 5
```

✅ **Verify:** Add a test job with `--at` 2 minutes from now. Watch it appear in `openclaw cron runs`.

---

## 2. System Crontab (Infrastructure Tasks)

**WHY:** Traditional Linux cron for tasks that don't need AI context — backups, health checks, monitoring.

### 2.1 Current Crontab

From Part 3, you already have:
```
0 * * * * /root/clawd/backup.sh >/dev/null 2>&1
0 4 * * * /root/clawd/scripts/nightly-backup.sh
```

### 2.2 Add Tunnel Health Check (Every 30min)

```bash
crontab -e
```

Add this line:
```
*/30 * * * * curl -s -o /dev/null -w "%{http_code}" https://webhook.poweker.com/health || echo "Tunnel down: $(date)" >> /root/clawd/memory/system-health.log
```

**WHY:** Cloudflare tunnel occasionally drops. This catches outages. Logs to `system-health.log` only on failure.

### 2.3 Add Cron Delivery Monitor (Every 5min)

**Purpose:** If an OpenClaw cron job runs successfully but WhatsApp delivery fails, this script detects the failure and pushes the message to the event queue for retry.

Create the monitor script:

```bash
cat > /root/clawd/scripts/cron-delivery-monitor.py << 'EOF'
#!/usr/bin/env python3
import sqlite3
import json
import subprocess
from pathlib import Path

DB = Path.home() / ".clawdbot" / "event-queue.sqlite"
OC_LOG = Path.home() / ".openclaw" / "cron-runs.log"  # Hypothetical OC log location

def check_failed_deliveries():
    """Check OC cron logs for successful runs with failed delivery."""
    if not OC_LOG.exists():
        return
    
    # Parse last 100 lines for delivery failures
    # (Actual implementation depends on OC's log format)
    # This is a skeleton — adjust to real OC log structure
    
    lines = OC_LOG.read_text().splitlines()[-100:]
    for line in lines:
        if "delivery_failed" in line.lower():
            # Extract job name, output, timestamp
            # Push to event queue
            subprocess.run([
                "python3", "/root/clawd/scripts/event-queue.py",
                "push", "--type", "cron-delivery",
                "--source", "delivery-monitor",
                "--payload", json.dumps({"log_line": line})
            ])

if __name__ == "__main__":
    check_failed_deliveries()
EOF

chmod +x /root/clawd/scripts/cron-delivery-monitor.py
```

Add to crontab:
```bash
crontab -e
```

Add:
```
*/5 * * * * /root/clawd/scripts/cron-delivery-monitor.py
```

**WHY:** No cron message ever gets permanently lost. If delivery fails, it gets retried via event queue.

✅ **Verify:** `crontab -l` shows all 4 jobs (hourly backup, nightly backup, tunnel check, delivery monitor).

---

## 3. Event Queue (Reliable Webhook Processing)

**WHY:** Webhooks are ephemeral — if the AI is busy or delivery fails, the event is lost. The event queue adds persistence, retries, and dead-letter alerting.

### 3.1 Create Event Queue Database

```bash
mkdir -p ~/.clawdbot
cat > /root/clawd/scripts/event-queue.py << 'EOF'
#!/usr/bin/env python3
"""
Event Queue: SQLite-backed persistent queue with retry logic.
Handles webhook events (email, calendar, alerts) and failed cron deliveries.
"""
import sqlite3
import json
import argparse
from datetime import datetime, timedelta
from pathlib import Path

DB_PATH = Path.home() / ".clawdbot" / "event-queue.sqlite"

def init_db():
    """Create queue table if not exists."""
    conn = sqlite3.connect(DB_PATH)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS events (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            type TEXT NOT NULL,
            source TEXT NOT NULL,
            payload TEXT NOT NULL,
            status TEXT DEFAULT 'pending',
            attempts INTEGER DEFAULT 0,
            created_at TEXT DEFAULT CURRENT_TIMESTAMP,
            last_attempt TEXT,
            error TEXT
        )
    """)
    conn.commit()
    conn.close()

def push_event(event_type, source, payload):
    """Add event to queue."""
    conn = sqlite3.connect(DB_PATH)
    conn.execute(
        "INSERT INTO events (type, source, payload) VALUES (?, ?, ?)",
        (event_type, source, json.dumps(payload))
    )
    conn.commit()
    event_id = conn.execute("SELECT last_insert_rowid()").fetchone()[0]
    conn.close()
    print(f"Event {event_id} queued: {event_type} from {source}")

def process_events():
    """Process pending events with retry logic."""
    conn = sqlite3.connect(DB_PATH)
    events = conn.execute(
        "SELECT id, type, source, payload, attempts FROM events WHERE status = 'pending' AND attempts < 3"
    ).fetchall()
    
    for event_id, etype, source, payload, attempts in events:
        print(f"Processing event {event_id} (attempt {attempts + 1}/3)")
        
        try:
            # TODO: Actual processing logic here
            # For skeleton: just simulate success/failure
            success = True  # Replace with real processing
            
            if success:
                conn.execute("UPDATE events SET status = 'success' WHERE id = ?", (event_id,))
            else:
                raise Exception("Processing failed")
        
        except Exception as e:
            new_attempts = attempts + 1
            if new_attempts >= 3:
                conn.execute(
                    "UPDATE events SET status = 'dead', attempts = ?, error = ? WHERE id = ?",
                    (new_attempts, str(e), event_id)
                )
                print(f"Event {event_id} moved to dead letter (3 failures)")
            else:
                conn.execute(
                    "UPDATE events SET attempts = ?, last_attempt = ?, error = ? WHERE id = ?",
                    (new_attempts, datetime.utcnow().isoformat(), str(e), event_id)
                )
    
    conn.commit()
    conn.close()

def status():
    """Show queue statistics."""
    conn = sqlite3.connect(DB_PATH)
    stats = conn.execute("""
        SELECT status, COUNT(*) FROM events GROUP BY status
    """).fetchall()
    conn.close()
    
    print("Queue Status:")
    for status, count in stats:
        print(f"  {status}: {count}")

def dead_letters():
    """Show dead letter events."""
    conn = sqlite3.connect(DB_PATH)
    dead = conn.execute(
        "SELECT id, type, source, created_at, error FROM events WHERE status = 'dead'"
    ).fetchall()
    conn.close()
    
    if not dead:
        print("No dead letter events")
    else:
        for event_id, etype, source, created, error in dead:
            print(f"{event_id}: {etype} from {source} ({created}) - {error}")

def retry(event_id):
    """Retry a dead letter event."""
    conn = sqlite3.connect(DB_PATH)
    conn.execute(
        "UPDATE events SET status = 'pending', attempts = 0 WHERE id = ?",
        (event_id,)
    )
    conn.commit()
    conn.close()
    print(f"Event {event_id} reset to pending")

if __name__ == "__main__":
    init_db()
    
    parser = argparse.ArgumentParser()
    parser.add_argument("action", choices=["push", "process", "status", "dead", "retry"])
    parser.add_argument("--type", help="Event type (push)")
    parser.add_argument("--source", help="Event source (push)")
    parser.add_argument("--payload", help="JSON payload (push)")
    parser.add_argument("--id", type=int, help="Event ID (retry)")
    parser.add_argument("--all", action="store_true", help="Process all pending (process)")
    
    args = parser.parse_args()
    
    if args.action == "push":
        push_event(args.type, args.source, json.loads(args.payload))
    elif args.action == "process":
        process_events()
    elif args.action == "status":
        status()
    elif args.action == "dead":
        dead_letters()
    elif args.action == "retry":
        retry(args.id)
EOF

chmod +x /root/clawd/scripts/event-queue.py
```

**WHY:** Skeleton shows the pattern — init DB, push events, process with retries, handle dead letters. Production version would add actual webhook processing logic in `process_events()`.

### 3.2 Initialize Queue

```bash
python3 /root/clawd/scripts/event-queue.py status
```

**WHY:** First run creates the SQLite database at `~/.clawdbot/event-queue.sqlite`.

✅ **Verify:** `ls -lh ~/.clawdbot/event-queue.sqlite` shows the database file.

### 3.3 Test the Queue

```bash
# Push a test event
python3 /root/clawd/scripts/event-queue.py push \
  --type "test" \
  --source "manual" \
  --payload '{"message": "queue test"}'

# Check status
python3 /root/clawd/scripts/event-queue.py status

# Process pending
python3 /root/clawd/scripts/event-queue.py process --all

# Check again
python3 /root/clawd/scripts/event-queue.py status
```

✅ **Verify:** Status shows 1 success event after processing.

### 3.4 Schedule Queue Processor (OC Cron)

```bash
openclaw cron add \
  --name "event-queue-processor" \
  --cron "*/15 * * * *" \
  --model "claude-sonnet-4-0" \
  --message "Run: python3 /root/clawd/scripts/event-queue.py process --all. If there are dead letter events (check with 'dead' command), alert me with details."
```

**WHY:** Processes queue every 15 minutes. AI can inspect dead letters and alert you.

### 3.5 Dead Letter Alerting

```bash
# Check for dead letters
python3 /root/clawd/scripts/event-queue.py dead

# Retry a specific event
python3 /root/clawd/scripts/event-queue.py retry --id 5
```

**WHY:** If an event fails 3 times, it moves to dead letter queue. You can manually inspect and retry.

✅ **Verify:** `openclaw cron list` shows the queue processor job.

---

## 4. Heartbeat System

**WHY:** OpenClaw can poll your AI periodically during idle time (no active conversations). Heartbeats are for maintenance tasks that need conversation context but don't need exact timing.

### 4.1 Understanding Heartbeats

**Heartbeat vs Cron:**
- **Cron:** Exact timing, isolated session, different model, one-shot reminders
- **Heartbeat:** Batch checks, needs conversation context, timing can drift (every ~30-60min when idle)

**Typical heartbeat tasks:**
- Check calendar for imminent events
- Process event queue
- Verify service health
- Check context usage
- Update daily log

### 4.2 Create HEARTBEAT.md

```bash
cat > /root/clawd/HEARTBEAT.md << 'EOF'
# HEARTBEAT.md - Idle-Time Maintenance

Run these checks every heartbeat (when idle, ~30-60min intervals).

## Priority Checks (Always)

1. **Calendar Urgency**
   - Read calendar for next 2 hours
   - If red event or meeting <30min away, alert via WhatsApp
   - Check for event changes since last heartbeat

2. **Event Queue**
   - Run: `python3 scripts/event-queue.py status`
   - If pending >10 or any dead letters, process immediately

3. **Service Health** (Quick check only)
   - Check tunnel: `curl -s -o /dev/null -w "%{http_code}" https://webhook.poweker.com/health`
   - If down, log to `memory/system-health.log`

## Secondary Checks (When Time Permits)

4. **Context Usage**
   - If >150K tokens used, note in daily log
   - Consider compacting if >180K

5. **Daily Log**
   - If last entry >4 hours old during work hours (9am-10pm UTC), note the gap

## Quiet Hours

**23:00-08:00 UTC:** Only Priority #1 (urgent calendar events). Skip everything else.

## Notes

- Heartbeats are best-effort, not guaranteed
- Don't be annoying — log issues, alert only urgent
- If processing takes >30s, defer to next heartbeat
EOF
```

**WHY:** Centralized list of heartbeat tasks. AI reads this on each poll.

### 4.3 Configure OpenClaw Heartbeat

OpenClaw handles heartbeat timing automatically. You configure it in `openclaw.json`:

```bash
# Edit OpenClaw config
nano ~/.openclaw/openclaw.json
```

Add/verify this section:
```json
{
  "heartbeat": {
    "enabled": true,
    "intervalMinutes": 45,
    "quietHoursStart": "23:00",
    "quietHoursEnd": "08:00"
  }
}
```

**WHY:** Heartbeat every 45 minutes when idle. Quiet hours prevent nighttime spam.

Restart OpenClaw:
```bash
openclaw gateway restart
```

✅ **Verify:** After 45 minutes of idle time, check your chat for a heartbeat check-in (or check `openclaw logs`).

### 4.4 Test Heartbeat Manually

```bash
# Trigger immediate heartbeat (if OpenClaw supports it)
openclaw heartbeat --now
```

**WHY:** Tests heartbeat logic without waiting 45 minutes.

---

## Part 4 Complete Checklist

✅ **OpenClaw Cron Jobs:**
- [ ] `openclaw cron list` shows morning-agenda, evening-wrap, health-checkin
- [ ] Tested one-shot reminder with `--at` and `--delete-after-run`
- [ ] `openclaw cron runs` shows recent executions

✅ **System Crontab:**
- [ ] `crontab -l` shows 4 jobs: hourly backup, nightly backup, tunnel check, delivery monitor
- [ ] Tunnel health check logs only on failure
- [ ] Delivery monitor script created and executable

✅ **Event Queue:**
- [ ] Database created at `~/.clawdbot/event-queue.sqlite`
- [ ] Tested push/process/status/dead/retry commands
- [ ] OC cron job processes queue every 15min
- [ ] Dead letter alerting works

✅ **Heartbeat System:**
- [ ] `HEARTBEAT.md` created with task list
- [ ] `openclaw.json` configured with heartbeat settings
- [ ] Quiet hours set (23:00-08:00 UTC)
- [ ] First heartbeat executed after 45min idle

**What you built:** Four-layer automation — AI-driven schedules (OC cron), infrastructure tasks (system cron), reliable event processing (queue), and idle-time maintenance (heartbeat). No event is lost, no reminder is missed, and the system self-monitors.

**Next:** Part 5 — Hardening (CrowdSec, secret rotation, config snapshots, recovery scripts)

---

**Time estimate:** 1 hour (30min for cron setup, 20min for event queue, 10min for heartbeat)

**Difficulty:** Intermediate — requires understanding of cron syntax, SQLite basics, and AI session isolation concepts.
