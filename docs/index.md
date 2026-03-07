# OpenClaw Installation Guide

An idiot-proof, copy-paste guide to setting up a **hardened, production-ready OpenClaw AI assistant** from zero in 3-4 hours.

## What is OpenClaw?

[OpenClaw](https://github.com/openclaw/openclaw) connects AI models (Claude, Gemini, etc.) to messaging platforms (WhatsApp, Telegram, Discord) with persistent memory, automation, and tool use. Think of it as your personal AI assistant that lives in your chat.

## What This Guide Covers

| Part | Topic | Time | What You Get |
|------|-------|------|-------------|
| [Part 1](part1-base-install.md) | Base Install | 30 min | Hardened VPS + OpenClaw + WhatsApp connected |
| [Part 2](part2-ai-assistant.md) | Your AI Assistant | 30 min | Persona, models, workspace configured |
| [Part 3](part3-infrastructure.md) | Infrastructure | 1 hr | Google OAuth, webhooks, Cloudflare tunnel, backups |
| [Part 4](part4-automation.md) | Automation | 1 hr | Cron jobs, event queue, reminders, heartbeats |
| [Part 5](part5-hardening.md) | Hardening | 30 min | CrowdSec, secret rotation, snapshots, recovery |
| [Part 6](part6-verification.md) | Verification | 15 min | 21-point validation + troubleshooting |

## Prerequisites

- A VPS (guide uses [Hetzner](https://console.hetzner.cloud/) CX22 — 2 vCPU, 4GB RAM, ~€4/mo)
- A WhatsApp account for bot pairing
- API keys for [Anthropic](https://console.anthropic.com) and/or [Google AI](https://aistudio.google.com)
- A domain name (for webhooks via Cloudflare tunnel)
- ~3-4 hours of uninterrupted time

## Design Principles

- **Linear, no branching** — One path, one outcome. No "if you want X, do Y."
- **Copy-paste ready** — Every command block can be pasted directly into terminal.
- **Verification checkpoints** — Each step ends with a ✅ check before moving on.
- **Production-hardened** — Not a toy setup. Firewall, intrusion detection, encrypted backups, 3-tier recovery.

## What You End Up With

- 🤖 AI assistant in WhatsApp with custom personality
- 📅 Google Calendar, Gmail, Drive integration
- 🔄 Automated backups (hourly + nightly to encrypted Google Drive)
- ⏰ Cron jobs, reminders, event queue with retry logic
- 🛡️ CrowdSec intrusion detection, UFW firewall, SSH hardening
- 🔑 Secret rotation schedule + automated alerts
- 💾 Config snapshots + 3-tier recovery (3 seconds to full restore)
- ✅ 21-point automated verification script

## Status

> **Last Updated:** March 7, 2026 (OpenClaw 2026.2.26+)
>
> This guide covers core setup and gets you to a fully working production system. Advanced hardening features (Chinese wall enforcement between work silos, automated config drift detection, cron backup guards) are documented in the codebase but not yet covered here. Core functionality is solid and battle-tested.

## Community

- **OpenClaw Docs:** [docs.openclaw.ai](https://docs.openclaw.ai)
- **OpenClaw Source:** [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)
- **Discord:** [discord.com/invite/clawd](https://discord.com/invite/clawd)
- **Skills Marketplace:** [clawhub.com](https://clawhub.com)

## Credits

Built by [Ali Ahmad Aziz](https://github.com/aliahmadaziz) with help from the OpenClaw community 🦞

Based on a real production setup running 24/7 since January 2026.

## License

MIT — use it, fork it, improve it.
