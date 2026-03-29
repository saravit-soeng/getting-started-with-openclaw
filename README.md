# OpenClaw — Complete Guide
> Self-hosted AI agent gateway for WhatsApp, Telegram, Discord, iMessage, and more.

---

## Table of Contents
1. [What is OpenClaw?](#what-is-openclaw)
2. [Key Concepts](#key-concepts)
3. [Setup on WSL2 (Windows)](#setup-on-wsl2-windows)
4. [Build a Simple AI Agent with Telegram](#build-a-simple-ai-agent-with-telegram)
5. [Useful Commands](#useful-commands)

---

## What is OpenClaw?

OpenClaw is a **self-hosted, open-source AI agent gateway** that connects your
favorite chat apps to AI models (Anthropic Claude, OpenAI, Google, Ollama, etc.).
You run a single **Gateway** process on your own machine or server — it becomes
the bridge between your messaging apps and an always-available AI assistant.

**Who is it for?** Developers and power users who want a personal AI assistant
they can message from anywhere without giving up control of their data.

### Why OpenClaw?

| Feature | Description |
|---|---|
| **Self-hosted** | Runs on your hardware, your rules, your data |
| **Multi-channel** | One Gateway serves WhatsApp, Telegram, Discord, Signal, Slack, and more simultaneously |
| **Agent-native** | Built for AI agents with tool use, sessions, memory, and multi-agent routing |
| **Open source** | MIT licensed, community-driven |
| **Model-agnostic** | Works with Anthropic, OpenAI, Google, Ollama, and any OpenAI-compatible server |

### Supported Channels

WhatsApp, Telegram, Discord, Slack, Signal, iMessage (BlueBubbles), Google Chat,
Microsoft Teams, IRC, Matrix, LINE, Mattermost, Twitch, and more.

---

## Key Concepts

### Architecture

```
WhatsApp / Telegram / Slack / Discord / Signal / iMessage
                        │
                        ▼
              ┌─────────────────────┐
              │       Gateway        │  ws://127.0.0.1:18789
              └──────────┬──────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     AI Agent          CLI          WebChat UI
    (RPC mode)    (openclaw …)     (browser)
```

### Workspace Bootstrap Files

Inside `~/.openclaw/workspace/`, these files are injected into every session:

| File | Purpose |
|---|---|
| `AGENTS.md` | Operating instructions and persistent memory |
| `SOUL.md` | Persona, tone, and boundaries |
| `TOOLS.md` | Notes on how you want tools used |
| `IDENTITY.md` | Agent name, vibe, emoji |
| `USER.md` | Your profile and preferred address |

### Skills

Skills are `SKILL.md` files with YAML frontmatter that teach the agent how and
when to use tools. They live in `<workspace>/skills/` and are loaded at runtime.

```
~/.openclaw/workspace/skills/
└── my-skill/
    └── SKILL.md    ← YAML frontmatter + markdown instructions
```

---

## Setup on WSL2 (Windows)

### Prerequisites

- Windows 10 (build 19041+) or Windows 11
- Node.js 24 (installed inside WSL)
- An API key from Anthropic, OpenAI, Google, or another provider

---

### Phase 1 — Install WSL2 + Ubuntu

Open **PowerShell as Administrator**:

```powershell
# Install WSL2 with Ubuntu (default)
wsl --install

# Or pick a specific version
wsl --install -d Ubuntu-24.04

# Reboot Windows if prompted
```

---

### Phase 2 — Enable systemd (required for the Gateway service)

Open your **Ubuntu (WSL) terminal**:

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true

[network]
generateResolvConf=false
EOF

# Set reliable DNS (fixes common WSL2 network issues)
sudo rm -f /etc/resolv.conf
sudo tee /etc/resolv.conf >/dev/null <<'EOF'
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF

sudo chattr +i /etc/resolv.conf
```

Shut down WSL from PowerShell, then reopen Ubuntu:

```powershell
wsl --shutdown
```

Verify systemd is running:

```bash
systemctl --user status
```

---

### Phase 3 — Install Node.js via nvm (Recommended)

Using `nvm` avoids APT network issues entirely:

```bash
# Install nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# Reload shell
source ~/.bashrc

# Install and use Node 24
nvm install 24
nvm use 24
nvm alias default 24

# Verify
node --version    # v24.x.x
npm --version
```

---

### Phase 4 — Install OpenClaw

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

---

### Phase 5 — Run Onboarding

```bash
openclaw onboard --install-daemon
```

This wizard:
- Lets you choose a model provider (Anthropic, OpenAI, etc.)
- Prompts for your API key
- Registers the Gateway as a **systemd user service** that auto-starts

Verify it's running:

```bash
openclaw gateway status    # should show port 18789
openclaw dashboard         # opens browser Control UI
```

---

### Phase 6 — Auto-Start on Windows Boot (Optional)

For headless setups where no one logs into Windows:

```bash
# Inside WSL — keep user services alive without login
sudo loginctl enable-linger "$(whoami)"

# Install the Gateway service
openclaw gateway install
```

In **PowerShell as Administrator** — start WSL at Windows boot:

```powershell
# Check your distro name
wsl --list --verbose

# Create scheduled task
schtasks /create /tn "WSL Boot" /tr "wsl.exe -d Ubuntu --exec /bin/true" /sc onstart /ru SYSTEM
```

Verify after reboot:

```bash
systemctl --user is-enabled openclaw-gateway
systemctl --user status openclaw-gateway --no-pager
```

---

### Common WSL2 Troubleshooting

| Problem | Fix |
|---|---|
| `apt update` fails with `NOSPLIT` | Fix DNS (Phase 2) or use `nvm` to bypass APT |
| `systemctl` not found | Re-add `systemd=true` to `/etc/wsl.conf`, then `wsl --shutdown` |
| Gateway won't start | Run `openclaw doctor` |
| Can't open dashboard from Windows browser | See portproxy forwarding below |

**Expose Gateway to Windows browser** (PowerShell as Admin):

```powershell
$WslIp = (wsl -d Ubuntu -- hostname -I).Trim().Split(" ")[0]
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 `
  listenport=18789 connectaddress=$WslIp connectport=18789
New-NetFirewallRule -DisplayName "OpenClaw Gateway" -Direction Inbound `
  -Protocol TCP -LocalPort 18789 -Action Allow
```

---

## Build a Simple AI Agent with Telegram

We'll build **"Briffy"** — a cheerful daily briefing agent accessible from Telegram.

### Step 1 — Create a Telegram Bot

1. Open Telegram and message **[@BotFather](https://t.me/botfather)**
2. Send `/newbot` and follow the prompts
3. Copy your bot token: `123456789:ABC-xxxxxxxxxxxxxxxxxxxxxxxxxxxx`

---

### Step 2 — Initialize Your Workspace

```bash
openclaw setup
ls ~/.openclaw/workspace/
# AGENTS.md  SOUL.md  TOOLS.md  IDENTITY.md  USER.md
```

---

### Step 3 — Configure the Agent Identity

**`~/.openclaw/workspace/IDENTITY.md`**
```markdown
name: Briffy
emoji: ☀️
vibe: cheerful morning assistant
```

**`~/.openclaw/workspace/SOUL.md`**
```markdown
# Soul

You are Briffy — warm, efficient, and positive without being over the top.
You speak like a knowledgeable friend, not a corporate chatbot.
Never use filler phrases like "Certainly!" or "Of course!".
Keep replies short unless detail is specifically asked for.
```

**`~/.openclaw/workspace/USER.md`**
```markdown
# User Profile

Name: [Your Name]
Timezone: Asia/Seoul (KST, UTC+9)
Preferred format: bullet points for summaries, plain prose for explanations
```

**`~/.openclaw/workspace/AGENTS.md`**
```markdown
# Briffy — Daily Briefing Agent

You are Briffy, a cheerful morning assistant.

## Core behaviours
- Always greet the user by name (see USER.md)
- Keep responses concise and upbeat
- When asked for a briefing, use the `daily_briefing` skill
- Never run commands that modify files unless explicitly asked
- If you don't know something, say so clearly
```

---

### Step 4 — Create the Daily Briefing Skill

```bash
mkdir -p ~/.openclaw/workspace/skills/daily-briefing
```

**`~/.openclaw/workspace/skills/daily-briefing/SKILL.md`**
```markdown
---
name: daily_briefing
description: Produces a morning briefing with current time, date, and workspace summary.
---

# Daily Briefing Skill

When the user asks for a "briefing", "morning update", or "daily summary", do:

1. Use the `exec` tool to run `date` and get the current date and time.
2. Use the `exec` tool to run `ls -lh ~/` to list home directory contents.
3. Compose a short briefing in this format:

   **☀️ Good morning, [Name]!**

   📅 **Date & Time:** [result from step 1]

   📁 **Workspace snapshot:**
   [3–5 notable files/dirs from step 2 with sizes]

   🗒️ **Today's focus:** [one encouraging sentence]

Keep it under 150 words. Be warm and brief.
```

---

### Step 5 — Configure `openclaw.json`

Edit `~/.openclaw/openclaw.json`:

```json
{
  "identity": {
    "name": "Briffy",
    "theme": "cheerful morning assistant",
    "emoji": "☀️"
  },
  "agent": {
    "workspace": "~/.openclaw/workspace",
    "model": {
      "primary": "anthropic/claude-sonnet-4-6"
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "YOUR_TELEGRAM_BOT_TOKEN_HERE"
    }
  },
  "messages": {
    "ackReaction": "☀️"
  },
  "logging": {
    "level": "info"
  }
}
```

---

### Step 6 — Restart and Test

```bash
# Reload skills and config
openclaw gateway restart

# Verify the skill loaded
openclaw skills list

# Quick terminal test
openclaw agent --message "Give me my morning briefing"
```

Then open Telegram, find your bot, and send:
```
Give me my morning briefing
```

Expected reply:
```
☀️ Good morning, [Your Name]!

📅 Date & Time: Mon Mar 30 09:15:42 KST 2026

📁 Workspace snapshot:
- AGENTS.md (1.2K)
- SOUL.md (0.4K)
- skills/ (dir)

🗒️ Today's focus: You've got a great setup — let's make it count!
```

---

### Step 7 — Automate a Daily Briefing at 8am (Optional)

Add to `openclaw.json`:

```json
{
  "automation": {
    "cron": [
      {
        "schedule": "0 8 * * *",
        "message": "Give me my morning briefing",
        "agent": "main"
      }
    ]
  }
}
```

Then restart:

```bash
openclaw gateway restart
```

---

## Useful Commands

| Command | Description |
|---|---|
| `openclaw gateway status` | Check Gateway health and port |
| `openclaw gateway restart` | Reload config and skills |
| `openclaw gateway install` | Register as systemd service |
| `openclaw dashboard` | Open browser Control UI |
| `openclaw setup` | Initialize workspace files |
| `openclaw onboard` | Re-run setup wizard |
| `openclaw doctor` | Diagnose and auto-repair issues |
| `openclaw skills list` | List all loaded skills |
| `openclaw agent --message "..."` | Chat from the terminal |
| `openclaw config set <key> <val>` | Edit config non-interactively |
| `openclaw gateway logs --follow` | Watch live Gateway logs |

### In-Chat Commands (send from Telegram/WhatsApp)

| Command | Description |
|---|---|
| `/status` | Show model, tokens, cost |
| `/model` | Switch the active model |
| `/reset` | Clear the current session |
| `/new` | Start a fresh session |

---

## Next Steps

- **Add more skills** — web search, git summary, weather via `exec` + `curl`
- **Connect more channels** — WhatsApp, Discord, Slack simultaneously
- **Memory plugin** — install `memory-lancedb` so the agent remembers across sessions
- **Multi-agent routing** — create a separate `work` agent with stricter tool policies
- **Browse ClawHub** — community skill registry at [clawhub.com](https://clawhub.com)

---

*Generated from the OpenClaw documentation — [docs.openclaw.ai](https://docs.openclaw.ai)*
