# OpenClaw Installation Guide

An idiot-proof, copy-paste guide to setting up a **hardened, production-ready OpenClaw AI assistant** from zero in 3-4 hours.

## What is OpenClaw?

[OpenClaw](https://github.com/openclaw/openclaw) connects AI models (Claude, Gemini, etc.) to messaging platforms (WhatsApp, Telegram, Discord) with persistent memory, automation, and tool use. Think of it as your personal AI assistant that lives in your chat.

!!! info "What This Guide Covers"
    This comprehensive guide takes you from a blank VPS to a fully hardened, production-ready AI assistant with automated backups, monitoring, and recovery systems.

## Installation Parts

| Part | Topic | Time | What You Get |
|------|-------|------|-------------|
| [Part 1 - Base Install](part1-base-install.md) | Base Install | 30 min | Hardened VPS + OpenClaw + WhatsApp connected |
| [Part 2 - AI Assistant](part2-ai-assistant.md) | Your AI Assistant | 30 min | Persona, models, workspace configured |
| [Part 3 - Infrastructure](part3-infrastructure.md) | Infrastructure | 1 hr | Google OAuth, webhooks, Cloudflare tunnel, backups |
| [Part 4 - Automation](part4-automation.md) | Automation | 1 hr | Cron jobs, event queue, reminders, heartbeats |
| [Part 5 - Hardening](part5-hardening.md) | Hardening | 30 min | CrowdSec, secret rotation, snapshots, recovery |
| [Part 6 - Verification](part6-verification.md) | Verification | 15 min | 21-point validation + troubleshooting |

## Prerequisites

Before you begin, make sure you have:

- **A VPS** - Guide uses [Hetzner](https://console.hetzner.cloud/) CX22 (2 vCPU, 4GB RAM, ~€4/mo)
- **WhatsApp account** - For bot pairing
- **API keys** - [Anthropic](https://console.anthropic.com) and/or [Google AI](https://aistudio.google.com)
- **Domain name** - For webhooks via Cloudflare tunnel
- **Time** - ~3-4 hours of uninterrupted time

!!! warning "Production Environment"
    This guide sets up a production-grade system. Do not skip security steps or use weak credentials.

## Design Principles

This guide follows strict principles to ensure reliability:

- **Linear, no branching** — One path, one outcome. No "if you want X, do Y."
- **Copy-paste ready** — Every command block can be pasted directly into terminal.
- **Verification checkpoints** — Each step ends with a ✅ check before moving on.
- **Production-hardened** — Not a toy setup. Firewall, intrusion detection, encrypted backups, 3-tier recovery.

## What You End Up With

By the end of this guide, you'll have:

- 🤖 **AI assistant** in WhatsApp with custom personality
- 📅 **Google Calendar, Gmail, Drive** integration
- 🔄 **Automated backups** - Hourly + nightly to encrypted Google Drive
- ⏰ **Cron jobs** - Reminders, event queue with retry logic
- 🛡️ **Security** - CrowdSec intrusion detection, UFW firewall, SSH hardening
- 🔑 **Secret rotation** - Schedule + automated alerts
- 💾 **Recovery** - Config snapshots + 3-tier recovery (3 seconds to full restore)
- ✅ **Verification** - 21-point automated validation script

## Community

- **OpenClaw Docs:** [docs.openclaw.ai](https://docs.openclaw.ai)
- **OpenClaw Source:** [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)
- **Discord:** [discord.com/invite/clawd](https://discord.com/invite/clawd)
- **Skills Marketplace:** [clawhub.com](https://clawhub.com)

## Getting Started

Ready to begin? Start with [Part 1 - Base Install](part1-base-install.md) to set up your VPS and install OpenClaw.

---

**Credits:** Built by [Your Name](https://github.com/yourusername) with help from MyBot 🦞

Based on a real production setup running 24/7 since January 2026.

**License:** MIT — use it, fork it, improve it.
