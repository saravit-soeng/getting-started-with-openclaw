# OpenClaw Tutorial: Daily Briefing Agent with OpenAI + Telegram

A complete guide to installing OpenClaw, connecting GPT-4o-mini as your AI brain, wiring up a Telegram bot, and scheduling a daily morning briefing — with all known errors and fixes documented.

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
10. [Known Errors & Fixes](#10-known-errors--fixes)
11. [Daily Commands](#11-daily-commands)

---

## 1. What is OpenClaw?

OpenClaw is a self-hosted AI agent that runs as a persistent background daemon on your own machine. You interact with it through messaging platforms you already use — WhatsApp, Telegram, Slack, Discord, iMessage, Signal — and it can:

- Run shell commands and control your browser
- Read and write files on your machine
- Manage your calendar and send emails
- Proactively message you on a schedule (cron)
- Search the web in real-time

All data (gateway, tools, memory) lives on your machine. The AI model can be cloud-hosted (OpenAI, Anthropic, Google) or fully local via Ollama.

### Architecture Overview

<img width="662" height="405" alt="image" src="https://github.com/user-attachments/assets/a5bdab49-d457-48ac-93ff-2e9ab6e7c78c" />

---

## 2. Prerequisites

### Node.js version check

```bash
node --version
# Must be v22.x.x or higher

npm --version
# Must be 10.x.x or higher
```

> **If below v22:** run `nvm install 22 && nvm use 22` or download from nodejs.org.

### You will also need

| Item | Where to get it | Notes |
|---|---|---|
| OpenAI API key | platform.openai.com | Starts with `sk-...` |
| Telegram account | telegram.org | Free; mobile or desktop |
| 16 GB RAM | Your machine | OpenClaw minimum |

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

### Verify

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

> **Cost note:** gpt-4o-mini costs ~$0.15/million input tokens. A daily briefing with web search uses ~2,000–3,000 tokens — well under $1/month.

---

## 5. Connect Telegram

### Step 5a — Create a bot via BotFather

1. Open Telegram and search for **@BotFather**
2. Send `/newbot`
3. Enter a display name — e.g. **My Daily Briefing**
4. Enter a username ending in `bot` — e.g. `mydailybriefing_bot`
5. BotFather replies with a token like: `1234567890:ABCdefGHIjklMNOpqrsTUVwxyz`
6. Save the token — treat it like a password

### Step 5b — Get your Telegram user ID

Search for **@userinfobot** in Telegram, start a chat, and it replies with your numeric user ID (e.g. `987654321`).

### Step 5c — Get your numeric chat ID

Send any message to your bot, then run:

```bash
curl "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates" \
  | python3 -m json.tool | grep '"id"'
# Returns your numeric chatId e.g. 987654321
```

Or check the gateway logs after sending a message:

```bash
openclaw gateway logs --tail 20
# Look for: telegram inbound chatId=987654321 from=YourName
```

### Step 5d — Add bot token to OpenClaw

```bash
openclaw config set channels.telegram.botToken YOUR_BOT_TOKEN
openclaw config set channels.telegram.dmPolicy "allowlist"
openclaw config set channels.telegram.allowFrom[0] 987654321
openclaw config set channels.telegram.streamMode "partial"
openclaw gateway restart
```

### Step 5e — Approve the pairing

```bash
# Watch logs for the pairing code
openclaw gateway logs --follow

# In another terminal, approve:
openclaw pairing approve telegram ABCD1234
# ✓ Pairing approved successfully
```

### Step 5f — Verify

Send `/status` to your bot in Telegram. It should reply with agent status and session info.

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
        "id":      "aria",
        "default": true,
        "name":    "Aria",
        "emoji":   "🌅"
      }
    ]
  },

  "channels": {
    "telegram": {
      "botToken":   "${TELEGRAM_BOT_TOKEN}",
      "dmPolicy":   "allowlist",
      "allowFrom":  [987654321],
      "streamMode": "partial"
    }
  }
}
```

### ~/.openclaw/.env

```env
OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxxxxxxxxx
TELEGRAM_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
```

---

## 7. Agent Personality Files

OpenClaw reads three Markdown files from the workspace on every session start.

### SOUL.md — personality & tone

```markdown
# Aria — Morning Briefing Agent

You are Aria, a sharp and warm morning briefing assistant.
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

- Role: software engineer / ML practitioner
- Topics I care about: AI research, open source, developer tools
- Preferred language: English
- Timezone: Asia/Seoul (KST, UTC+9)
- Morning briefing: 8:00 AM KST
- Keep summaries concise — I read on my phone
```

---

## 8. Schedule the Morning Briefing

### Important: "tasks" key is NOT valid in openclaw.json

The gateway will abort with `Unrecognized key: "tasks"` if you add a `tasks` block to `openclaw.json`. Scheduled jobs are managed separately via the `openclaw cron` CLI and stored in `~/.openclaw/cron/jobs.json`.

### The cron directory is created on first job add

`~/.openclaw/cron/` does not exist on a fresh install — it is created automatically the first time you run `openclaw cron add`.

### Which session target to use

| Situation | `sessionTarget` | `payload.kind` |
|---|---|---|
| Default agent, auto-deliver to paired Telegram | `main` | `systemEvent` |
| Named agent (`agentId` set) | `isolated` | `agentTurn` |
| One-shot test job | `isolated` | `agentTurn` |

> **Note:** `sessionTarget: "main"` only works with the **default agent**. If your agent has a custom `agentId`, you must use `isolated` + `agentTurn`.

### Add job via CLI — default agent (simplest, no chatId needed)

```bash
openclaw cron add \
  --name "morning-briefing" \
  --cron "0 8 * * *" \
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

### Edit jobs.json directly

Always stop the gateway before editing the file manually — otherwise your changes will be overwritten.

```bash
# Step 1: Stop the gateway
openclaw gateway stop

# Step 2: Edit the file
nano ~/.openclaw/cron/jobs.json
```

Full schema for jobs.json:

```json
{
  "jobs": [
    {
      "jobId": "morning-briefing",
      "name": "Morning Briefing",
      "enabled": true,
      "description": "Daily AI news digest via Telegram",

      "schedule": {
        "kind": "cron",
        "expr": "0 8 * * *",
        "tz": "Asia/Seoul"
      },

      "sessionTarget": "main",

      "payload": {
        "kind": "systemEvent",
        "message": "Run the morning briefing: search for the top 3 AI and tech headlines from today, format as the briefing template in SOUL.md, and send the result here."
      },

      "wakeMode": "now",
      "agentId": "aria",
      "deleteAfterRun": false
    }
  ]
}
```

For a **named agent**, use this structure instead:

```json
{
  "jobs": [
    {
      "jobId": "morning-briefing",
      "name": "Morning Briefing",
      "enabled": true,
      "schedule": {
        "kind": "cron",
        "expr": "0 8 * * *",
        "tz": "Asia/Seoul"
      },
      "sessionTarget": "isolated",
      "agentId": "daily-morning-brief-agent",
      "payload": {
        "kind": "agentTurn",
        "message": "Search for the top 3 AI and tech headlines from today and format them as the briefing template in SOUL.md.",
        "delivery": {
          "channel": "telegram",
          "to": "987654321"
        }
      },
      "wakeMode": "now",
      "deleteAfterRun": false
    }
  ]
}
```

> **Key:** `delivery` is nested **inside** `payload`, not a sibling field.

```bash
# Step 3: Validate JSON
cat ~/.openclaw/cron/jobs.json | python3 -m json.tool

# Step 4: Restart
openclaw gateway start
openclaw cron list
```

### Auto-deliver to paired Telegram without hardcoding chatId

If you want delivery to your paired bot with no chatId, use `--channel last` — this reuses the last route the agent replied to:

```bash
# First: send any message to your bot in Telegram (establishes the last route)

# Then add the job:
openclaw cron add \
  --name "morning-briefing" \
  --cron "0 8 * * *" \
  --tz "Asia/Seoul" \
  --session isolated \
  --message "Search for the top 3 AI and tech headlines from today and format them as the briefing template in SOUL.md." \
  --announce \
  --channel last
```

---

## 9. Testing the Cron Job

### Method 1 — `openclaw cron run` (recommended)

```bash
# Get your job ID
openclaw cron list

# Trigger immediately
openclaw cron run <jobId>

# Watch outcome
openclaw cron runs --id <jobId>
openclaw gateway logs --follow
```

### Method 2 — One-shot `--at` job (reliable workaround)

```bash
openclaw cron add \
  --name "briefing-test" \
  --at "2026-03-31T$(date -u -d '+2 minutes' '+%H:%M'):00Z" \
  --session isolated \
  --message "Search for the top 3 AI and tech headlines from today and format them as the briefing template in SOUL.md." \
  --announce \
  --channel last

openclaw gateway logs --follow
```

One-shot jobs delete themselves after a successful run.

### Method 3 — Chat-driven test (simplest)

Send this directly to your Telegram bot:

```
Run the morning briefing: search for the top 3 AI and tech headlines
from today, format as the briefing template in SOUL.md, and send back here.
```

If it works in chat, it will work in cron.

> **Known bug (Linux, v2026.3.8):** `openclaw cron run <jobId>` may return `{ enqueued: true }` but never execute — the job sits "running" until it times out with `lane wait exceeded`. Scheduled automatic execution works fine; only manual triggers via `cron run` are affected. Use Method 2 as a workaround.

---

## 10. Known Errors & Fixes

### Error: `Unrecognized key: "tasks"`

```
Gateway aborted: config is invalid.
<root>: Unrecognized key: "tasks"
```

**Cause:** `"tasks"` is not a valid key in `openclaw.json`. Cron jobs must be managed via `openclaw cron` CLI, not the main config.

**Fix:** Remove the `"tasks": [...]` block from `openclaw.json`, then restart:

```bash
openclaw gateway restart
```

---

### Error: `pairing required`

```
GatewayClientRequestError: Error: gateway closed (1008): pairing required
```

**Cause:** The CLI device has not been authorized yet. This is a security handshake — the gateway intentionally rejects unrecognized connections.

**Fix:**

```bash
# List pending pairing requests
openclaw devices list

# Approve your device
openclaw devices approve <request-id>

# If nothing shows up, force reinstall service identity
openclaw gateway install --force
openclaw gateway restart
```

---

### Error: `Delivering to Telegram requires target <chatId>`

**Cause:** The cron subagent spawned in isolated mode has no delivery context — it doesn't know which Telegram chat to send to.

**Fix options (in order of preference):**

1. Use `--session main` with default agent — inherits paired Telegram automatically, no chatId needed
2. Use `--channel last` — reuses the last route the agent replied to
3. Explicitly pass `--to "YOUR_NUMERIC_CHATID"` and nest `delivery` inside `payload` in jobs.json

---

### Error: `sessionTarget "main" is only valid for the default agent`

```
GatewayClientRequestError: Error: cron: sessionTarget "main" is only valid
for the default agent. Use sessionTarget "isolated" with payload.kind
"agentTurn" for non-default agents (agentId: daily-morning-brief-agent)
```

**Cause:** `sessionTarget: "main"` only works with the default agent. Named agents require `isolated` + `agentTurn`.

**Fix:** Update jobs.json:

```json
{
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "...",
    "delivery": {
      "channel": "telegram",
      "to": "987654321"
    }
  }
}
```

---

### General troubleshooting table

| Symptom | Cause | Fix |
|---|---|---|
| Bot doesn't reply | Gateway stopped | `openclaw gateway start` |
| "Unauthorized" error | User ID not in allowFrom | Re-run pairing or add ID to `allowFrom` |
| Node version error | Node < 22 | `nvm install 22 && nvm use 22` |
| OpenAI 401 error | Bad API key | Re-run `openclaw models auth paste-token --provider openai` |
| Cron fires but no Telegram message | `streamMode` misconfigured | Set `channels.telegram.streamMode: "partial"` |
| `jobs.json` edits lost on restart | Edited while gateway was running | Always stop gateway before editing, restart after |
| Cron run enqueues but never fires | Known Linux bug v2026.3.8 | Use `--at` one-shot job instead |

---

## 11. Daily Commands

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

## Appendix: Cron Expression Reference

| Expression | Meaning |
|---|---|
| `0 8 * * *` | Every day at 08:00 (local tz) |
| `0 23 * * *` | Every day at 23:00 UTC (= 08:00 KST) |
| `0 8 * * 1-5` | Weekdays only at 08:00 |
| `0 8,20 * * *` | Twice daily at 08:00 and 20:00 |
| `*/30 * * * *` | Every 30 minutes |

---

*Last updated: March 2026 | OpenClaw v2026.x | GPT-4o-mini | Telegram*

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
