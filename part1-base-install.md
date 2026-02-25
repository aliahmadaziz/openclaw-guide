# OpenClaw Installation Guide - Part 1: Base Install (30 min)

**Goal:** Replicate a hardened OpenClaw setup on a fresh VPS. One path, one outcome.

**Prerequisites:** 
- SSH key pair on your local machine (`~/.ssh/id_rsa.pub`)
- WhatsApp account for bot pairing

---

## 1. VPS Provisioning (Hetzner Cloud)

### 1.1 Create VPS

1. Login to [Hetzner Cloud Console](https://console.hetzner.cloud/)
2. Create new project (or select existing)
3. Click **"Add Server"**
4. Configure:
   - **Location:** Nuremberg (eu-central)
   - **Image:** Ubuntu 24.04
   - **Type:** CX22 (2 vCPU, 4GB RAM)
   - **SSH Keys:** Add your public key (`~/.ssh/id_rsa.pub`)
   - **Name:** openclaw-prod (or your choice)
5. Click **"Create & Buy Now"**

✅ **Verify:** Server shows "Running" status, note the IP address

### 1.2 Initial Connection

```bash
ssh root@YOUR_SERVER_IP
```

*First connection will ask to verify fingerprint, type `yes`*

✅ **Verify:** You're logged in as root, prompt shows `root@ubuntu-...`

---

## 2. SSH Hardening

### 2.1 Configure SSH Settings

Edit SSH daemon config:

```bash
nano /etc/ssh/sshd_config
```

Find and change these lines (use Ctrl+W to search):

```
PermitRootLogin prohibit-password
PasswordAuthentication no
X11Forwarding no
```

*Why: Key-only auth prevents brute force attacks, X11Forwarding reduces attack surface*

Save and exit (`Ctrl+X`, then `Y`, then `Enter`)

### 2.2 Restart SSH

```bash
systemctl restart sshd
```

*Why: Apply new SSH security settings*

### 2.3 Test SSH Connection

**Open a NEW terminal window** (keep current session open):

```bash
ssh root@YOUR_SERVER_IP
```

✅ **Verify:** You can still login without password prompt. If this fails, debug in original window before closing it.

---

## 3. OS Hardening

### 3.1 Setup Firewall (UFW)

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp comment 'SSH'
ufw --force enable
```

*Why: Default-deny firewall reduces exposure, only SSH port accessible*

✅ **Verify:** `ufw status` shows "Status: active" with SSH allowed

### 3.2 Remove Unnecessary Packages

```bash
apt remove -y cups cups-daemon nginx docker.io docker-compose
apt autoremove -y
```

*Why: Reduces attack surface and resource usage*

### 3.3 Remove Snap Packages

```bash
snap list
snap remove chromium 2>/dev/null || true
snap remove gnome-3-38-2004 2>/dev/null || true
snap remove gnome-42-2204 2>/dev/null || true
snap remove gtk-common-themes 2>/dev/null || true
snap remove snap-store 2>/dev/null || true
snap remove snapd-desktop-integration 2>/dev/null || true
apt purge -y snapd
rm -rf /snap /var/snap /var/lib/snapd
```

*Why: Snap auto-updates can break production systems, not needed for server*

✅ **Verify:** `snap list` returns "command not found"

### 3.4 Add Swap Space

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

*Why: 2GB swap prevents OOM kills during memory spikes. OpenClaw with 200K context can spike to 3GB RAM usage with large Claude model responses.*

✅ **Verify:** `free -h` shows ~2G swap

### 3.5 System Updates

```bash
apt update
apt upgrade -y
apt autoremove -y
```

*Why: Security patches and stability fixes*

✅ **Verify:** No errors, ends with "0 upgraded, 0 newly installed" on second run

---

## 4. Install Node.js 22

### 4.1 Add NodeSource Repository

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
```

*Why: NodeSource provides official, up-to-date Node.js builds*

### 4.2 Install Node.js

```bash
apt install -y nodejs
```

*Why: OpenClaw requires Node.js runtime*

✅ **Verify:** `node --version` shows `v22.22.0` or later (minimum v22.x required), `npm --version` shows `10.x` or later

---

## 5. Install OpenClaw

### 5.1 Install via NPM

```bash
npm install -g openclaw
```

*Why: Global install makes `openclaw` command available system-wide*

✅ **Verify:** `openclaw --version` shows version 2026.2.x or later

---

## 6. Initial Setup

### 6.1 Run Setup Wizard

```bash
openclaw setup
```

*Why: Configures WhatsApp connection and creates config file*

**Follow the prompts:**

1. **WhatsApp pairing method:** Choose "QR Code" (option 1)
2. **QR Code will appear in terminal**
3. **On your phone:**
   - Open WhatsApp
   - Tap ⋮ (menu) → Linked Devices → Link a Device
   - Scan the QR code from your terminal
4. Wait for "WhatsApp connected" message

✅ **Verify:** Setup completes without errors, shows "WhatsApp connected"

### 6.2 Start OpenClaw Gateway

```bash
openclaw gateway start
```

*Why: Starts the OpenClaw service to handle messages*

✅ **Verify:** `openclaw gateway status` shows "running"

**⚠️ Pairing Loop Troubleshooting:** If QR code scans successfully but immediately disconnects (repeats continuously), this is a device scope issue. See Part 5, Section 5 for troubleshooting. This happens if WhatsApp doesn't grant proper permissions during pairing.

---

## 7. Verification

### 7.1 Send Test Message

**From your phone (or another WhatsApp account):**

1. Send a message to the connected WhatsApp number: `Hi`
2. Wait 2-3 seconds

✅ **Verify:** Bot replies with a response (default personality will respond)

### 7.2 Check Logs

```bash
openclaw gateway logs
```

*Why: Confirms message processing is working*

✅ **Verify:** Logs show incoming message and outgoing response

---

## 8. Future Upgrades

### 8.1 How to Upgrade OpenClaw

**⚠️ IMPORTANT:** Always backup before upgrading. Never use `npm update -g openclaw` directly.

```bash
# Check for updates
openclaw update

# The command will show available updates and prompt for confirmation
```

*Why: `openclaw update` handles pre-upgrade backups, config migration, and rollback if upgrade fails. Raw npm updates can break your setup.*

### 8.2 Upgrade Best Practices

**Before upgrading:**
1. Save config snapshot (see Part 5, Section 3)
2. Test WhatsApp is working
3. Note current version: `openclaw --version`
4. Read release notes at [github.com/openclaw/openclaw/releases](https://github.com/openclaw/openclaw/releases)

**After upgrading:**
1. Verify gateway starts: `openclaw gateway status`
2. Test WhatsApp message exchange
3. Check device scope: `openclaw status | grep scope` (should show `operator.read,operator.write`)
4. If anything breaks: restore snapshot and report issue

✅ **Verify:** You know how to check for updates (`openclaw update`) and where to find release notes

---

## Part 1 Complete ✅

You should now have:

- [ ] Hetzner VPS running Ubuntu 24.04 (CX22)
- [ ] SSH hardened (key-only, no password, no X11)
- [ ] UFW firewall active (deny incoming except SSH)
- [ ] Unnecessary packages removed (cups, nginx, docker, snap)
- [ ] 2GB swap space active
- [ ] System fully updated
- [ ] Node.js 22 installed
- [ ] OpenClaw installed globally
- [ ] WhatsApp paired via QR code
- [ ] OpenClaw gateway running
- [ ] Test message confirmed working

**Next:** Part 2 — Your AI Assistant (persona, models, workspace setup)

---

## Quick Reference Commands

```bash
# Check status
openclaw gateway status

# View logs
openclaw gateway logs

# Restart gateway
openclaw gateway restart

# Stop gateway
openclaw gateway stop

# Server info
ufw status
free -h
node --version
df -h
```

---

**Estimated time:** 25-30 minutes (excluding VPS provisioning wait time)
