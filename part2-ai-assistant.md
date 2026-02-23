# OpenClaw Installation Guide - Part 2: Your AI Assistant (30 min)

**Goal:** Configure the AI personality, model chain, and workspace. Transform OpenClaw from a default chatbot into your personal AI assistant.

**Prerequisites:** 
- Part 1 complete (OpenClaw installed, WhatsApp connected)
- A Claude subscription (Max plan recommended) OR an Anthropic API key
- A Google AI Studio account for Gemini access

---

## 1. Model Chain Setup

The model chain determines which AI models handle different tasks. You configure primary (conversations), fallback (backup), sub-agents (background tasks), and cron (scheduled jobs).

### 1.1 Get Your AI Credentials

**Option A — Claude Max Plan (Recommended)**

The Claude Max plan ($100/month) gives you Claude access with built-in usage limits and cooldown periods, which is much more cost-predictable than raw API usage.

1. Subscribe to Claude Max at [claude.ai/settings/billing](https://claude.ai/settings/billing)
2. Install Claude Code CLI: `npm install -g @anthropic-ai/claude-code`
3. Run `claude` in your terminal — it will prompt you to authenticate
4. During auth, it generates an access token — copy this token
5. Set it in OpenClaw:

```bash
openclaw config set anthropicApiKey YOUR_CLAUDE_CODE_TOKEN
```

*Why: The Max plan has built-in spend limits and cooldown periods. Raw API keys (from console.anthropic.com) have no guardrails and can run up significant bills fast. Max plan is safer for always-on assistants.*

**Option B — Raw Anthropic API Key**

If you prefer pay-as-you-go: get a key from [console.anthropic.com](https://console.anthropic.com) and set spend limits in the console.

```bash
openclaw config set anthropicApiKey YOUR_ANTHROPIC_KEY
```

**Google Gemini (Fallback Model)**

1. Go to [aistudio.google.com](https://aistudio.google.com)
2. Sign in with your Google account
3. Click "Get API Key" → create a key
4. Google AI Studio gives you **$300 in free credits** — more than enough for a fallback model

```bash
openclaw config set googleApiKey YOUR_AISTUDIO_KEY
```

*Why: AI Studio's free credits make Gemini essentially free as a fallback. You're not paying for a second model — Google is subsidizing your backup.*

✅ **Verify:** `openclaw config get anthropicApiKey` and `openclaw config get googleApiKey` show your keys (first few chars visible)

### 1.2 Configure Primary Model

```bash
openclaw config set defaultModel anthropic/claude-opus-4-6
```

*Why: Primary model handles all conversations and complex reasoning. Opus 4 is the most capable.*

### 1.3 Configure Fallback Model

```bash
openclaw config set fallbackModels '["google/gemini-2.5-pro"]'
```

*Why: If Anthropic API is down, OpenClaw automatically switches to Gemini. You stay online.*

### 1.4 Configure Sub-Agent Model

```bash
openclaw config set subagentModel anthropic/claude-sonnet-4-5
```

*Why: Sub-agents handle background tasks (research, file processing). Sonnet 4.5 is fast and cost-effective.*

### 1.5 Configure Cron Model

```bash
openclaw config set cronModel anthropic/claude-sonnet-4-0
```

*Why: Scheduled jobs use the older, cheaper Sonnet 4.0 for routine tasks (reminders, checks).*

### 1.6 Set Context Window

```bash
openclaw config set contextTokens 200000
```

*Why: 200K tokens = ~150K words of context. Never exceed the smallest model in your chain.*

✅ **Verify:** Run `openclaw config list | grep -E 'Model|context'` — should show all four models configured

**Model Summary:**
- **Primary (Opus 4.6):** Your main conversation partner, handles all direct messages
- **Fallback (Gemini 2.5 Pro):** Backup if primary is down, keeps you online 24/7
- **Sub-agents (Sonnet 4.5):** Background workers for research, file parsing, parallel tasks
- **Cron (Sonnet 4.0):** Scheduled jobs like reminders, alerts, periodic checks

---

## 2. Workspace Structure

The workspace is where your AI lives. These files define personality, memory, and behavior.

### 2.1 Create Workspace Directory

```bash
mkdir -p ~/clawd/memory ~/clawd/scripts
cd ~/clawd
```

*Why: Central location for all AI personality, memory, and automation files*

### 2.2 Create Core Files

```bash
touch SOUL.md USER.md IDENTITY.md AGENTS.md MEMORY.md HANDOFF.md HEARTBEAT.md TOOLS.md SECURITY.md
```

*Why: Each file serves a specific purpose in how your AI thinks and acts*

✅ **Verify:** `ls ~/clawd/*.md` shows all 9 files

**File Purposes:**

| File | Purpose |
|------|---------|
| `SOUL.md` | Personality, tone, hard rules — the AI's core character |
| `USER.md` | Info about you (name, role, timezone, preferences) |
| `IDENTITY.md` | Bot's name, emoji, avatar — how it identifies itself |
| `AGENTS.md` | Instructions for every session (memory, search, safety protocols) |
| `MEMORY.md` | Long-term curated memory — important facts that survive resets |
| `HANDOFF.md` | Current working state — survives resets, tracks in-flight tasks |
| `HEARTBEAT.md` | Periodic check-in tasks — what to do during idle time |
| `TOOLS.md` | Local tool notes and wrapper scripts — custom commands |
| `SECURITY.md` | Security policies — what requires permission, what's forbidden |
| `memory/` | Daily logs (YYYY-MM-DD.md) — raw record of what happened each day |
| `scripts/` | Utility scripts — custom automation tools |

### 2.3 Create SOUL.md Template

```bash
cat > ~/clawd/SOUL.md << 'EOF'
# SOUL.md - Core Personality

## Tone
Direct. Conversational. No corporate polish. Use sentence fragments when natural.

## Style
- **Short replies** — Don't over-explain. Answer the question, then stop.
- **Context-aware** — Read the room. Match energy. Formal when needed, casual otherwise.
- **Honest defaults** — "I don't know" > guessing. Ask clarifying questions.

## Hard Rules
1. **Memory over mental notes** — Write it down. Files > brain.
2. **Ask before public actions** — Emails, tweets, posts need approval.
3. **Safe freely** — Read files, search web, work within workspace without asking.

## Forbidden
- Corporate jargon ("delighted to assist", "circle back", "synergy")
- Unsolicited advice or life coaching
- Making up facts or hallucinating data
EOF
```

*Why: Defines how the AI talks and thinks. Short, clear rules prevent generic corporate-bot responses.*

### 2.4 Create USER.md Template

```bash
cat > ~/clawd/USER.md << 'EOF'
# USER.md - About You

**Name:** Your Name  
**Role:** Founder & CEO  
**Timezone:** UTC+5 (Your Timezone)  
**Location:** Your City

## Communication Preferences
- **Response time:** Immediate for urgent, batched for routine
- **Format:** WhatsApp is primary, keep it concise
- **Tone:** Direct, no fluff

## Work Context
- Runs multiple companies (e-commerce, logistics, tech)
- High volume of meetings, emails, and decisions
- Needs AI to handle research, scheduling, and follow-ups

## Personal Notes
- Prefers dark mode screenshots
- Uses 24-hour time format
- Frequent traveler, timezone shifts
EOF
```

*Why: Tells the AI who you are, how you work, and what you expect. It uses this to personalize responses.*

### 2.5 Create IDENTITY.md Template

```bash
cat > ~/clawd/IDENTITY.md << 'EOF'
# IDENTITY.md - Bot Identity

**Name:** MyBot  
**Emoji:** 🤖  
**Avatar:** (none set)

**Role:** Personal AI assistant  
**Primary Function:** Memory, research, automation, scheduling

**Personality Snapshot:**  
Efficient. Low-ego. Direct. Not here to impress, here to help.
EOF
```

*Why: Gives the AI a clear identity. It knows what it is and what it's not.*

✅ **Verify:** `cat ~/clawd/SOUL.md` shows the template content

---

## 3. Persona Configuration

Now point OpenClaw to your workspace so these files are injected into every conversation.

### 3.1 Set Workspace Path

```bash
openclaw config set workspace ~/clawd
```

*Why: Tells OpenClaw where to find personality and memory files*

### 3.2 Configure Project Context Files

```bash
openclaw config set projectContextFiles '["AGENTS.md","TOOLS.md"]'
```

*Why: These files are loaded into EVERY conversation as context. The AI reads them before responding.*

**Why these two files?**
- **AGENTS.md:** Session instructions (how to search memory, safety rules, when to spawn sub-agents)
- **TOOLS.md:** Local tool notes (custom scripts, API credentials, wrapper commands)

**Why not SOUL.md or USER.md?**
- Those are loaded via system prompts and persona config (set in next section)

### 3.3 Set System Prompt (Optional but Recommended)

```bash
openclaw config set systemPrompt "You are MyBot, your personal AI assistant. Read SOUL.md, USER.md, and SECURITY.md before every response. Be direct, efficient, and context-aware."
```

*Why: This instruction is prepended to every conversation, ensuring the AI always reads core files*

✅ **Verify:** `openclaw config get workspace` shows `/root/clawd`

---

## 4. First Real Conversation

Time to restart OpenClaw and talk to your newly configured AI assistant.

### 4.1 Restart Gateway

```bash
openclaw gateway restart
```

*Why: Reload all configuration changes (models, workspace, persona)*

Wait 5-10 seconds for full restart.

✅ **Verify:** `openclaw gateway status` shows "running"

### 4.2 Send Test Message

**From WhatsApp, send:**

```
Who are you and what's my name?
```

**Expected Response:**
The bot should introduce itself using the identity from `IDENTITY.md` (MyBot 🤖) and mention your name from `USER.md` (Your Name).

✅ **Verify:** Bot knows its own name and your name

### 4.3 Test Personality

**Send:**

```
Write me a long corporate email about synergy and circling back.
```

**Expected Response:**
The bot should refuse or push back based on `SOUL.md` forbidden rules (no corporate jargon).

✅ **Verify:** Bot follows SOUL.md tone rules

### 4.4 Test Timezone Awareness

**Send:**

```
What timezone am I in?
```

**Expected Response:**
Bot should reference `USER.md` and say UTC+5 / Your Timezone.

✅ **Verify:** Bot reads and uses USER.md data

### 4.5 Check Logs

```bash
openclaw gateway logs --tail 50
```

*Why: Confirms workspace files are being loaded*

✅ **Verify:** Logs show no errors, messages processed successfully

---

## 5. Fine-Tuning (Optional)

### 5.1 Adjust Memory Settings

```bash
openclaw config set memoryEnabled true
openclaw config set memoryMaxTokens 50000
```

*Why: Enables conversation memory (stores recent messages for context)*

### 5.2 Set Message Limits

```bash
openclaw config set maxReplyLength 2000
```

*Why: Prevents excessively long responses (especially on WhatsApp)*

### 5.3 Enable Thinking (Reasoning)

```bash
openclaw config set reasoning low
```

*Why: Shows brief reasoning traces for debugging AI decisions. Options: off, low, high, stream*

---

## Part 2 Complete ✅

You should now have:

- [ ] Claude access configured (Max plan token or API key)
- [ ] Model chain set (Opus 4.6 primary, Gemini fallback, Sonnet sub-agents, Sonnet cron)
- [ ] Context window set to 200K tokens
- [ ] Workspace created at `~/clawd`
- [ ] Core files created (SOUL.md, USER.md, IDENTITY.md, etc.)
- [ ] Minimal templates populated (personality, user info, identity)
- [ ] Workspace path configured in OpenClaw
- [ ] Project context files set (AGENTS.md, TOOLS.md)
- [ ] System prompt configured
- [ ] Gateway restarted with new config
- [ ] Bot responds with configured personality
- [ ] Bot knows your name and timezone from USER.md
- [ ] Bot follows SOUL.md tone and rules

**Next:** Part 3 — Infrastructure (Google OAuth, Cloudflare tunnel, webhooks, backups)

**⚠️ Note:** Some `openclaw config` paths may vary by version. Run `openclaw doctor` after configuration to validate. If a config path doesn't exist, check `openclaw config --help` or the docs at docs.openclaw.ai.

---

## Quick Reference Commands

```bash
# View all config
openclaw config list

# Check specific settings
openclaw config get defaultModel
openclaw config get workspace

# Edit personality files
nano ~/clawd/SOUL.md
nano ~/clawd/USER.md

# Restart after changes
openclaw gateway restart

# View logs
openclaw gateway logs --tail 50

# Check workspace files loaded
ls -lh ~/clawd/*.md
```

---

## Troubleshooting

**Bot doesn't know my name:**
- Check `openclaw config get workspace` points to `~/clawd`
- Verify `~/clawd/USER.md` exists and has your name
- Restart gateway: `openclaw gateway restart`

**Bot ignores SOUL.md rules:**
- Ensure `systemPrompt` is set: `openclaw config get systemPrompt`
- Check files exist: `ls ~/clawd/SOUL.md`
- Try explicit instruction: "Read SOUL.md and tell me your tone rules"

**API errors:**
- Verify keys: `openclaw config get anthropicApiKey` and `openclaw config get googleApiKey`
- If using Max plan: check usage at claude.ai/settings/usage
- If using raw API: check quotas at console.anthropic.com
- Google credits: check at aistudio.google.com → Settings
- Test fallback: temporarily disable Anthropic key to trigger Gemini

**Context too large errors:**
- Reduce `contextTokens`: `openclaw config set contextTokens 150000`
- Remove large files from workspace
- Trim daily logs in `~/clawd/memory/`

---

**Estimated time:** 25-30 minutes (excluding API key signup wait time)
