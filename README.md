# OpenClaw Tutorial: Daily Briefing Agent with OpenAI + Telegram

A complete guide to installing OpenClaw, connecting GPT-4o-mini as your AI brain, wiring up a Telegram bot, and scheduling a daily morning briefing.

---

## Table of Contents

1. [What is OpenClaw?](#1-what-is-openclaw)
2. [Prerequisites](#2-prerequisites)
3. [Installation](#3-installation)
4. [Connect OpenAI GPT-4o](#4-connect-openai-gpt-4o)
5. [Connect Telegram](#5-connect-telegram)
6. [Full Config Reference](#6-full-config-reference)
7. [Agent Personality Files](#7-agent-personality-files)
8. [Schedule the Morning Briefing](#8-schedule-the-morning-briefing)
9. [Testing the Cron Job](#9-testing-the-cron-job)
10. [Daily Commands](#10-daily-commands)

---

## 1. What is OpenClaw?

OpenClaw is a self-hosted AI agent that runs as a persistent background daemon on your own machine. You interact with it through messaging platforms you already use — WhatsApp, Telegram, Slack, Discord, iMessage, Signal — and it can:

- Run shell commands and control your browser
- Read and write files on your machine
- Manage your calendar and send emails
- Proactively message you on a schedule (cron)
- Search the web in real-time

All data (gateway, tools, memory) lives on your machine. The AI model can be cloud-hosted (OpenAI, Anthropic, Google) or fully local via Ollama.

Here's how the architecture fits together:
  
<img width="662" height="405" alt="image" src="https://github.com/user-attachments/assets/a5bdab49-d457-48ac-93ff-2e9ab6e7c78c" />

---

## 2. Prerequisites

Before installing, you need:

- Node.js 22+ and npm or pnpm OpenClaw
- An LLM API key — OpenClaw works best with Anthropic's Claude API, but also supports OpenAI, Google, and local models via Ollama
- A messaging platform account (Telegram is the easiest to start with)

> **If below v22:** run `nvm install 22 && nvm use 22` or download from nodejs.org.

---

## 3. Installation

### Install globally via npm

```bash
npm install -g openclaw@latest
```

Or use the one-liner script:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

### Verify it installed:

```bash
openclaw --version
# Expected: openclaw/2026.x.x linux-x64 node-v22.x.x
```

### Run the onboarding wizard

```bash
openclaw onboard --install-daemon
```

Answer the prompts as follows:

| Wizard prompt | Choose |
|---|---|
| Install as system daemon? | Yes |
| Setup type | Custom |
| Gateway bind address | 127.0.0.1 (loopback, safer) |
| AI model provider | OpenAI |
| Channels to enable | Telegram |
| Install extra skills now? | Skip |

### File structure created

```
~/.openclaw/
├── openclaw.json       ← main config
├── .env                ← secrets (API keys)
├── workspace/
│   ├── SOUL.md         ← agent personality
│   ├── AGENTS.md       ← operating instructions
│   └── USER.md         ← info about you
└── memory/             ← SQLite session memory
```

> **Security:** run `chmod 600 ~/.openclaw/.env` to restrict read permissions to your user only. Never commit `.env` to git.

---

## 4. Connect OpenAI GPT-4o

### Authenticate

```bash
openclaw models auth paste-token --provider openai
# Paste your key when prompted: sk-proj-xxxxxxxxxxxx
# ✓ OpenAI credentials saved to ~/.openclaw/.env
```

### Set GPT-4o-mini as default model

```bash
# Set primary model (cost-efficient for daily briefings)
openclaw config set agents.defaults.model.primary gpt-4o-mini

# Optional: full GPT-4o as fallback for complex queries
openclaw config set agents.defaults.model.secondary gpt-4o

# Verify
openclaw config get agents.defaults.model
```

### How this looks in openclaw.json

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary":   "gpt-4o-mini",
        "secondary": "gpt-4o"
      }
    }
  }
}
```

---

## 5. Connect Telegram

### Step 5a — Create a bot via BotFather

1. Open Telegram and search for **@BotFather**
2. Send `/newbot`
3. Enter a display name — e.g. **My Daily Briefing**
4. Enter a username ending in `bot` — e.g. `mydailybriefing_bot`
5. BotFather replies with a token like: `1234567890:ABCdefGHIjklMNOpqrsTUVwxyz`
6. Save the token — treat it like a password

### Step 5b — Add bot token to OpenClaw

```bash
openclaw config set channels.telegram.botToken YOUR_BOT_TOKEN
openclaw config set channels.telegram.dmPolicy "pairing"
openclaw config set channels.telegram.streamMode "partial"
openclaw gateway restart
```
### Step 5c — Open your telegram bot
1. In Telegram, search for your bot username — e.g. `mydailybriefing_bot`
2. In your chatbot, click start
3. It will response back the user id with specific code that can be used to pair the telegram bot with OpenClaw and command as below:

<img width="353" height="405" alt="Screen Shot 2026-04-01 at 1 26 40 AM" src="https://github.com/user-attachments/assets/5f8d1d2d-7bcc-4083-98b4-ac78caafad9d" />

### Step 5d — Approve the pairing

```bash
# Or you can watch logs for the pairing code
openclaw gateway logs --follow

# In another terminal, approve:
openclaw pairing approve telegram ABCD1234
# ✓ Pairing approved successfully
```

### Step 5e — Verify

Send `/status` to your bot in Telegram. It should reply with agent status and session info.

<img width="376" height="765" alt="Screen Shot 2026-04-01 at 1 17 51 AM" src="https://github.com/user-attachments/assets/41c4bb61-2e9f-44e6-abf4-382702474161" />

---

## 6. Full Config Reference

### ~/.openclaw/openclaw.json

```json
{
  "gateway": {
    "host": "127.0.0.1",
    "port": 18789
  },

  "agents": {
    "defaults": {
      "model": {
        "primary":   "gpt-4o-mini",
        "secondary": "gpt-4o"
      },
      "workspace": "~/.openclaw/workspace"
    },
    "list": [
      {
        "id":      "chinkkuu",
        "default": true,
        "name":    "Chhinkkuu",
        "emoji":   "🌅"
      }
    ]
  },

  "channels": {
    "telegram": {
      "botToken":   "${TELEGRAM_BOT_TOKEN}",
      "dmPolicy":   "pairing",
      "streamMode": "partial"
    }
  }
}
```

---

## 7. Agent Personality Files

OpenClaw reads three Markdown files from the workspace on every session start.

### SOUL.md — personality & tone

```markdown
# Chinkkuu — Morning Briefing Assistant

You are Chinkkuu, a sharp and warm morning briefing assistant.
Your job is to make the user's day start well — you surface
the signal, cut the noise, and add just enough context to
make each story feel real.

## Tone
- Confident but never arrogant
- Curious, occasionally dry, never corporate
- No filler phrases ("Certainly!", "Great question!")
- Get to the point

## Briefing format
When delivering the morning briefing:
- Use numbered lists with bold headline titles
- Keep each item to 2 sentences max
- End with: "Reply with a number to dive deeper 👇"

## Follow-up questions
When the user asks about a specific item:
- Search for more detail first (use web search)
- Answer in 3–5 concise sentences
- Offer to set a reminder or save a note if relevant
```

### AGENTS.md — operating rules

```markdown
# Operating Instructions

## Morning briefing task
1. Search the web for today's top 5 AI and tech headlines
2. Filter to the 3 most significant stories
3. Format as the briefing template in SOUL.md
4. Send to Telegram

## Tools to use
- web_search: always use for fresh news (never rely on training data)
- telegram.send: to push briefings proactively

## What NOT to do
- Do not include advertising or sponsored content
- Do not fabricate quotes or statistics
- Do not send more than one unsolicited message per day
```

### USER.md — context about you

```markdown
# About Me

- Role: software engineer / AI researcher
- Topics I care about: AI research, open source, developer tools
- Preferred language: English
- Timezone: Asia/Seoul (KST, UTC+9)
- Morning briefing: 10:00 AM KST
- Keep summaries concise — I read on my phone
```

---

## 8. Schedule the Morning Briefing

### Add job via CLI — default agent (simplest, no chatId needed)

```bash
openclaw cron add \
  --name "morning-briefing" \
  --cron "0 10 * * *" \
  --tz "Asia/Seoul" \
  --session main \
  --system-event "Run the morning briefing: search for the top 3 AI and tech headlines from today, format as the briefing template in SOUL.md, and send the result here." \
  --wake now
```

### Add job via CLI — named agent (requires chatId)

```bash
openclaw cron add \
  --name "morning-briefing" \
  --cron "0 8 * * *" \
  --tz "Asia/Seoul" \
  --session isolated \
  --message "Search for the top 3 AI and tech headlines from today and format them as the briefing template in SOUL.md." \
  --announce \
  --channel telegram \
  --to "987654321"
```
---

## 9. Testing the Cron Job

```bash
# Get your job ID
openclaw cron list

# Trigger immediately
openclaw cron run <jobId>

# Watch outcome
openclaw cron runs --id <jobId>
openclaw gateway logs --follow
```
In your telegram bot, you can see something as below when the cron schedule is run successfully

<img width="367" height="641" alt="Screen Shot 2026-04-01 at 1 23 37 AM" src="https://github.com/user-attachments/assets/5afe8a3c-1629-40b1-8134-3eeeb268bf35" />


## 10. Daily Commands

```bash
# Gateway
openclaw gateway status        # is the daemon alive?
openclaw gateway start         # start the daemon
openclaw gateway stop          # stop the daemon
openclaw gateway restart       # restart after config changes
openclaw gateway logs --follow # live log stream
openclaw gateway logs --tail 50

# Diagnostics
openclaw doctor                # full health check
openclaw config wizard         # interactive config editor
openclaw devices list          # list paired devices

# Cron
openclaw cron list             # all scheduled jobs
openclaw cron run <jobId>      # trigger a job immediately
openclaw cron runs --id <jobId># check run history
openclaw cron rm <jobId>       # delete a job
openclaw cron edit --id <jobId># edit a job interactively

# Models
openclaw config get agents.defaults.model
openclaw models auth paste-token --provider openai
```

---

## References

### Official OpenClaw Resources

| Resource | URL | Description |
|---|---|---|
| Main repository | https://github.com/openclaw/openclaw | Source code, releases, and issue tracker |
| Official website | https://openclaw.ai | Project homepage |
| Documentation (hosted) | https://docs.openclaw.ai | Full docs index — config, channels, security, agents |
| Docs index (GitHub) | https://github.com/openclaw/openclaw/blob/main/docs/index.md | All guides organized by use case |
| Getting started guide | https://github.com/openclaw/openclaw/blob/main/docs/start/getting-started.md | Beginner install + first chat walkthrough |
| Release notes | https://github.com/openclaw/openclaw/releases | Changelog for every version |
| GitHub organization | https://github.com/openclaw | All official repos |

### Related Official Repositories

| Repository | URL | Description |
|---|---|---|
| ClawHub (skill registry) | https://github.com/openclaw/clawhub | Community skill directory; 5,400+ skills at onlycrabs.ai |
| Docker image | https://github.com/coollabsio/openclaw | Fully featured Docker image with nginx reverse proxy |
| Nix packaging | https://github.com/openclaw/nix-openclaw | Nix flake for NixOS/nix-darwin installs |
| Windows companion | https://github.com/openclaw/openclaw-windows-node | System tray app + PowerToys palette extension |
| Community Discord docs | https://github.com/openclaw/community | Discord server policies and documentation |
| acpx (ACP CLI client) | https://github.com/openclaw/acpx | Headless CLI client for Agent Client Protocol sessions |

### External Services Referenced

| Service | URL | Used for |
|---|---|---|
| OpenAI Platform | https://platform.openai.com | API key for GPT-4o-mini / GPT-4o |
| Telegram BotFather | https://t.me/BotFather | Creating and managing Telegram bots |
| Telegram @userinfobot | https://t.me/userinfobot | Getting your Telegram numeric user ID |
| Node.js | https://nodejs.org | Required runtime (v22+ or v24 recommended) |
| nvm (Node version manager) | https://github.com/nvm-sh/nvm | Switching Node.js versions |
