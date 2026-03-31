# Personal AI Agent — Setup Guide

### Build your own always-on personal AI agent with OpenClaw, Claude, and a native iOS app

---

This is the companion guide to [*The Era of Personalized Software Is Here*](https://medium.com/@craigoley). It walks through the full stack: a Mac mini running an AI agent that updates you throughout the day, talks to you via iMessage, and powers a native iOS app you build with Claude Code.

The result is a personal AI system that costs about $5–10/month, runs continuously, and does what you actually want — because you built it.

---

> **Before you start: use this guide with Claude**
>
> This guide is designed to be read alongside a Claude conversation, not in isolation. Paste the guide into Claude at the start, then work through each section with it as context. When a step doesn't work as expected on your specific machine, Claude can help you adapt it. When you hit an error message you don't understand, paste it in and ask. The guide gives you the path — Claude helps you walk it.

---

## What this actually is

Before listing what you'll build, it's worth saying what this *is* at a higher level — because it's easy to get lost in the tools and miss the point.

This guide is about building a personal feedback loop. One where you can try an idea in the morning, have it running on your phone by afternoon, and iterate based on how it actually works for your life. The tools described here — OpenClaw, Claude Code, ClawApp — are the current implementation of that loop. They'll change. The principle won't.

The system you're building mixes two things that don't usually live together: **deterministic algorithms** (scripts that do the same thing every time, can't be reasoned with, don't cost money per run) and **non-deterministic AI** (language models that reason, synthesize, and handle ambiguity, but can't be trusted for exact repeatability). The art is knowing which to reach for. Your morning briefing: a script fetches your calendar (always the same way), Haiku reads it and writes what you need to know (AI). Your build pipeline: a shell script runs the same commands in the same order (deterministic), Claude Code wrote and maintains that script (AI). You're not replacing one with the other. You're composing them.

## What you'll end up with

In concrete terms: a Mac mini running an AI agent 24/7 that talks to you via your real iMessage number. A native iOS app with a chat screen, a morning briefing, system status, and a build pipeline — including a screen where you tap "build" and watch the agent compile and distribute a new version of the app to your phone. A morning briefing that synthesizes your calendar, reminders, and news every day without you asking. A model architecture that costs $5–10/month, with most background work running free on local models.

This is not a beginner tutorial. You'll need to be comfortable with the command line, running scripts, and sitting with an error message long enough to actually read it. But you don't need to be a professional iOS developer, and you don't need to understand everything here before you start — just work through it in order.

---

## What you'll need

A few things to have in place before you start:

- **Apple Silicon Mac mini** — M4 with 16GB is the sweet spot. This machine runs 24/7, so power efficiency matters. Cheaper Intel machines work but don't run local AI models well.
- **iPhone** running iOS 16 or later
- **Apple Developer account** — $99/year, required to distribute the iOS app to your own phone
- **Anthropic API key** — separate from your Claude subscription (explained in Part 3). Get one at [console.anthropic.com](https://console.anthropic.com).
- **Claude subscription** — Max or Max 20x recommended. Claude Code sessions use your subscription, not the API.
- **GitHub account** — free. Your app code lives here.
- **Expo account** — free. Handles iOS app builds and distribution.
- **Tailscale account** — free tier is enough. Lets you SSH into your Mac mini from anywhere.

---

## The Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    ClawApp (Your iPhone)                  │
│                                                           │
│  Chat ←──── WebSocket (port 18790 via proxy)            ┐│
│  Briefing ←─ Direct fetch (calendar, weather, podcasts)  ││
│  Home ←───── RPC (cron.list, usage.cost, health JSON)    ││
│  Build ←──── HTTP API (port 18795, build-status endpoint)││
│  Podcasts ←─ HTTP (port 18793, scored JSON file)         ││
└────────────────────────────┬─────────────────────────────┘│
                             │                              │
┌────────────────────────────▼──────────────────────────────┤
│                     Mac mini M4                           │
│                                                           │
│  ┌─────────────────┐   ┌──────────────────────────────┐  │
│  │  OpenClaw        │   │  Direct API Calls            │  │
│  │  (message bus)   │   │  (intelligence on demand)    │  │
│  │                  │   │                              │  │
│  │  WebSocket auth  │   │  Haiku: podcast scoring      │  │
│  │  Session state   │   │  Haiku: briefing synthesis   │  │
│  │  RPC gateway     │   │  Haiku: judgment tasks       │  │
│  │  iMessage bridge │   │                              │  │
│  │  Cron scheduler  │   │  ~$0.02/turn                 │  │
│  └────────┬─────────┘   └──────────────────────────────┘  │
│           │                                               │
│  ┌────────▼──────────────────────────────────────────┐   │
│  │              Shell Scripts & Cron Jobs             │   │
│  │                                                    │   │
│  │  build-clawapp.sh  · score-podcasts.py             │   │
│  │  export-reminders.sh · podcast-digest.js           │   │
│  │  Ollama (heartbeats, watchdog — $0)                │   │
│  └────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────┘
```

**The key architectural insight:** OpenClaw is the message bus and conversation layer. The Anthropic API is intelligence called directly when you need reasoning. Shell scripts handle deterministic processes. ClawApp is the interface that pulls from the right source for each piece of data — never routing something through the agent when a direct call is more reliable.

**Cost breakdown:**
| What | Model | Cost |
|------|-------|------|
| Your conversations | Claude Haiku | ~$0.02/turn |
| Morning briefing | Claude Haiku | ~$0.05/day |
| Heartbeat checks | Ollama (local) | $0 |
| Background crons | Ollama (local) | $0 |
| **Monthly total** | | **~$5–10** |

---

## The one thing to understand before you start

AI is non-deterministic. The same instruction can produce slightly different behavior each time — a different interpretation, a creative shortcut, occasionally something that breaks what was working. That's not a bug in the models. It's what makes them useful for reasoning. It's just fatal for a build pipeline.

The system described in this guide handles this by separating what AI does from what scripts do:

```
Scripts   → collect data, execute processes, maintain state
AI (LLMs) → reason about data, synthesize prose, handle ambiguity
```

Your morning briefing: a scheduled script fetches your calendar, a Haiku API call synthesizes the output. Your podcast scoring: a SQL query reads your app's database, Haiku scores each episode against your interests. Your build pipeline: a shell script runs the same steps every time, Claude Code wrote and maintains that script.

The agent triggers scripts. Scripts don't trigger agents (except to send results). AI reasons about data. Scripts collect it.

This isn't a philosophical preference — it's the thing that makes the system reliable. When you break this pattern, things fail in confusing ways. The agent rewrites the build script and runs a slightly different version. The monitoring cron burns API tokens on every check even when nothing's new. The scheduled briefing comes out differently every morning because the agent is making judgment calls that should be deterministic.

If a task needs to succeed 100% of the time, write a script. If it requires understanding, use AI. Most of what you're building here needs both.

---

## Part 1: Mac mini Setup

### 1.1 — Two Accounts

You need two macOS accounts on the same machine:

**Admin account** — your personal account. This is where your development tools, SSH keys, and personal apps live. Signed into your personal Apple ID.

**Agent account** — a standard (non-admin) user account dedicated entirely to the agent. This account runs OpenClaw. It cannot touch the admin account's files. If the agent does something wrong — runs a bad command, gets confused by a clever prompt — it should be working with the minimum possible access to your machine.

> ⚠️ Never run the agent on your admin account. This was the root cause of the group chat incident described in the companion post.

**Create the agent account:**

1. System Settings → Users & Groups → Add Account
2. Type: **Standard** (not Administrator)
3. Give it a name you'll remember — something like `agent` or `claw`
4. Create a strong password and store it in your password manager

**Create a dedicated Apple ID for the agent:**

Before setting up iMessage, create a fresh Gmail address and use it to register a new Apple ID at [appleid.apple.com](https://appleid.apple.com). This keeps the agent's iMessage identity completely separate from your personal messages. You'll sign into this Apple ID in Messages.app on the agent account in Part 5.

### 1.2 — Always-On Configuration

Your Mac mini needs to run without interruption:

```bash
# Prevent sleep
sudo pmset -a sleep 0 disksleep 0 displaysleep 0

# Wake on network access (for remote management)
sudo pmset -a womp 1

# Auto-restart after power failure
sudo pmset -a autorestart 1
```

Enable SSH for remote access:
System Settings → General → Sharing → Remote Login → On

### 1.3 — Tailscale

Install [Tailscale](https://tailscale.com) on both the Mac mini and your iPhone. This gives you a private VPN so you can SSH into the Mac mini from anywhere as if you're on the local network.

```bash
brew install tailscale
sudo tailscaled install-system-daemon
tailscale up
```

Note your Mac mini's Tailscale IP — you'll use it for SSH and ClawApp connections.

### 1.4 — Core Developer Tools

Install these as your admin account:

```bash
# Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Developer essentials
brew install node@22 git gh jq

# Node setup
brew link --overwrite node@22
node --version  # should show v22+

# GitHub CLI auth
gh auth login
```

---

## Part 2: OpenClaw Installation

Everything in Part 2 runs as the agent account — not your admin account. Switch to it via Fast User Switching (the user menu in your menu bar) or SSH:

```bash
ssh agent@your-tailscale-ip
```

### 2.1 — Install Homebrew for the Agent Account

The agent account needs its own Homebrew installation:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
source ~/.zprofile
```

### 2.2 — Install Ollama (Free Local Model)

Ollama runs language models locally. You'll use it for background tasks to keep API costs near zero.

```bash
brew install ollama
```

Configure Ollama as a system service so it starts at boot (not just when you're logged in). Create a LaunchDaemon plist at `/Library/LaunchDaemons/homebrew.ollama.plist` — run this as admin:

```bash
sudo tee /Library/LaunchDaemons/homebrew.ollama.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key><string>homebrew.ollama</string>
    <key>UserName</key><string>AGENT_ACCOUNT_NAME</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/ollama</string>
        <string>serve</string>
    </array>
    <key>EnvironmentVariables</key>
    <dict>
        <key>HOME</key><string>/Users/AGENT_ACCOUNT_NAME</string>
        <key>PATH</key><string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
    </dict>
    <key>RunAtLoad</key><true/>
    <key>KeepAlive</key><true/>
</dict>
</plist>
EOF

sudo launchctl load /Library/LaunchDaemons/homebrew.ollama.plist
```

Replace `AGENT_ACCOUNT_NAME` with your agent account's short username.

Pull a local model for background tasks:

```bash
ollama pull qwen2.5:7b
```

Verify:
```bash
curl -s http://127.0.0.1:11434/api/tags | jq '.models[].name'
```

### 2.3 — Install OpenClaw

As the agent account, configure npm to use a local prefix (the agent account may not have write access to Homebrew's global directories):

```bash
mkdir -p ~/.npm-global
npm config set prefix ~/.npm-global
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.zprofile
source ~/.zprofile
```

Install OpenClaw:

```bash
npm install -g openclaw@latest
openclaw --version
```

### 2.4 — Configure OpenClaw

Create `~/.openclaw/openclaw.json`:

```json
{
  "models": {
    "providers": {
      "anthropic": {
        "apiKey": "YOUR_ANTHROPIC_API_KEY",
        "models": [
          {
            "id": "anthropic/claude-haiku-4-5-20251001",
            "contextWindow": 200000,
            "maxTokens": 8096
          },
          {
            "id": "anthropic/claude-sonnet-4-6",
            "contextWindow": 200000,
            "maxTokens": 8096
          }
        ]
      },
      "ollama": {
        "baseUrl": "http://127.0.0.1:11434",
        "apiKey": "ollama-local",
        "api": "ollama",
        "models": [
          {
            "id": "ollama/qwen2.5:7b",
            "contextWindow": 32768,
            "maxTokens": 4096,
            "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0}
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-haiku-4-5-20251001",
        "fallbacks": ["ollama/qwen2.5:7b", "anthropic/claude-sonnet-4-6"]
      },
      "heartbeat": {
        "every": "60m",
        "model": "ollama/qwen2.5:7b"
      }
    }
  }
}
```

> **Critical:** The `"api": "ollama"` key in the Ollama provider block is required. Without it, OpenClaw defaults to the OpenAI-compatible endpoint, which Ollama doesn't support, causing silent 404 failures.

### 2.5 — Run OpenClaw as a System Service

OpenClaw needs to start at boot, before any user logs in. A system-level service (LaunchDaemon) handles this correctly. Run as admin:

```bash
sudo tee /Library/LaunchDaemons/ai.openclaw.gateway.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key><string>ai.openclaw.gateway</string>
    <key>UserName</key><string>AGENT_ACCOUNT_NAME</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/node</string>
        <string>/Users/AGENT_ACCOUNT_NAME/.npm-global/lib/node_modules/openclaw/dist/index.js</string>
        <string>gateway</string>
        <string>--port</string>
        <string>18789</string>
    </array>
    <key>EnvironmentVariables</key>
    <dict>
        <key>HOME</key><string>/Users/AGENT_ACCOUNT_NAME</string>
        <key>PATH</key><string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
    </dict>
    <key>RunAtLoad</key><true/>
    <key>KeepAlive</key><true/>
    <key>StandardOutPath</key><string>/tmp/openclaw/openclaw.log</string>
    <key>StandardErrorPath</key><string>/tmp/openclaw/openclaw-error.log</string>
</dict>
</plist>
EOF

mkdir -p /tmp/openclaw
sudo launchctl load /Library/LaunchDaemons/ai.openclaw.gateway.plist
```

Verify:
```bash
sudo launchctl list | grep openclaw
openclaw gateway status
```

---

## Part 3: Anthropic API Setup

### 3.1 — Two Separate Systems

Your Claude subscription (Pro, Max, Max 20x) and the Anthropic API are **completely separate billing systems**.

- **Your subscription** covers interactive use: claude.ai, Claude Code, Claude Desktop
- **The Anthropic API** is pay-per-token, billed separately, with its own key

When OpenClaw makes a model call, it uses the API. Your subscription doesn't cover it. Get an API key at [console.anthropic.com](https://console.anthropic.com) → API Keys → Create Key.

**Set a spending cap immediately.** Go to console.anthropic.com → Billing → Usage Limits → set a hard cap at $15–25/month to start. Without a cap, using Sonnet as the primary model for everything will run you $200+ per week.

Store your API key in `~/.openclaw/.env`:

```bash
echo 'ANTHROPIC_API_KEY=sk-ant-...' > ~/.openclaw/.env
chmod 600 ~/.openclaw/.env
```

### 3.2 — Right Model for the Right Task

This is the most important cost optimization in the entire setup:

| Use case | Model | Why |
|----------|-------|-----|
| Your conversations | Haiku | Fast, cheap, handles tool calling |
| Morning briefing | Haiku | Reliable output, tool use |
| Heartbeats, crons | Ollama | Free, tolerable failure rate |
| Complex debugging | Sonnet or Opus | Worth the cost |

> **The $200 lesson:** Running Sonnet as the primary model for *everything* — conversations, heartbeats, cron jobs, all of it — cost roughly $200 in the first week. Haiku handles 95% of what an agent needs at a fraction of the cost. Sonnet is for when you explicitly need heavier reasoning. Opus is for debugging sessions where you're genuinely stuck.

---

## Part 4: Claude Code Setup

Claude Code is Anthropic's agentic coding tool. You'll use it to build and iterate on ClawApp, and eventually the agent itself will use it to build new features.

### 4.1 — Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
claude --version
```

### 4.2 — Authenticate

```bash
claude auth login
```

This uses your Claude subscription (Max/Pro), not the API. Claude Code sessions are covered by your flat-rate subscription.

### 4.3 — GitHub Integration

Claude Code works best when it can read your full codebase and push changes. Configure GitHub:

```bash
gh auth login
git config --global user.email "your@email.com"
git config --global user.name "Your Name"
```

---

## Part 5: iMessage Channel (BlueBubbles)

BlueBubbles bridges iMessage so your agent can send and receive real iMessages to your iPhone.

### 5.1 — Install BlueBubbles

Download from [bluebubbles.app](https://bluebubbles.app). 

> ⚠️ **Run BlueBubbles exclusively on the agent account.** Never on your admin account. If BlueBubbles runs on your admin account, it will access your entire personal Messages database — every conversation you've ever had. The group chat incident in the companion post happened exactly this way.

On the agent account:
1. Sign into Messages.app with the dedicated Apple ID you created in Part 1
2. Open BlueBubbles → complete setup wizard
3. Set port: `1234`, create a strong password
4. Enable Private API (follow BlueBubbles documentation for SIP disable/re-enable)
5. Add Full Disk Access: System Settings → Privacy & Security → Full Disk Access → add BlueBubbles
6. Set BlueBubbles to auto-launch: BlueBubbles Settings → Startup → "Launch Agent (Crash Persistent)"

### 5.2 — Connect BlueBubbles to OpenClaw

Add the BlueBubbles channel to `~/.openclaw/openclaw.json`:

```json
"channels": {
  "bluebubbles": {
    "enabled": true,
    "serverUrl": "http://YOUR_LAN_IP:1234",
    "password": "YOUR_BLUEBUBBLES_PASSWORD",
    "webhookPath": "/bluebubbles-webhook",
    "dmPolicy": "allowlist",
    "allowFrom": ["YOUR_PHONE_NUMBER"],
    "groupPolicy": "disabled"
  }
}
```

Use your LAN IP address (not `localhost`) for the server URL. Get it with `ipconfig getifaddr en0`.

Register the webhook in BlueBubbles Settings → Webhooks:
```
http://127.0.0.1:18789/bluebubbles-webhook?password=YOUR_GATEWAY_TOKEN
```

Get your gateway token from `~/.openclaw/openclaw.json` under `gateway.token`.

### 5.3 — macOS Tahoe: Private API Fix

If you're running macOS Tahoe (26+), there's a known issue where OpenClaw's BlueBubbles plugin falls back to AppleScript for message sends, which is broken on Tahoe. The fix requires two patches to the OpenClaw source code.

> ⚠️ These patches are overwritten when you run `npm update`. You must re-apply them after any OpenClaw update. The full patch script is in this repo at `scripts/patch-bluebubbles-tahoe.sh`.

Find the send file:
```bash
find ~/.npm-global -name "send.ts" -path "*/bluebubbles/*"
```

Apply two patches:
1. Around the `needsPrivateApi` declaration: force it to `true`
2. Around the `canUsePrivateApi` check: force the condition to always enter the private API branch

The exact lines change between OpenClaw versions — refer to the patch script in this repo for the current implementation.

Restart the gateway after patching:
```bash
sudo launchctl kickstart -k system/ai.openclaw.gateway
```

Test:
```bash
openclaw gateway status  # should show RPC probe: ok
```

Send yourself an iMessage. You should receive a response from the agent.

---

## Part 6: The Agent's Identity — AGENTS.md

Before giving the agent tools, define who it is. OpenClaw uses a workspace directory (`~/.openclaw/workspace/`) containing files the agent reads at the start of every session.

The most important is `AGENTS.md`. Create `~/.openclaw/workspace/AGENTS.md`:

```markdown
# Agent Configuration

## Who I am
I am a personal AI assistant running on your Mac mini. My primary job is to help you stay on top of your life — calendar, tasks, information — and to help you build and improve the apps and tools you use.

## How I communicate
- Keep responses concise. Respect the person's time.
- For anything taking more than 2 minutes, send a status update.
- Never silently fail. If something is blocked, say so immediately.
- One question at a time. Never stack multiple asks.

## What I can access
- Google Calendar (via gog CLI)
- Apple Reminders (via remindctl)
- Web search
- Local file system under my home directory
- GitHub repos I've been given access to

## Rules that exist because something went wrong
- Never delete databases or state files. Use targeted edits instead.
- Never complete financial transactions without explicit confirmation.
- Never install software or tools without explicit approval.
- Always verify a configuration key exists before spending time debugging it.

## This file is a living document
When you learn something important, add it here.
```

> **The AGENTS.md pattern** is one of the most valuable practices in this entire setup. Every rule in that file exists because something went wrong. It's the institutional memory of the system, written so that every new Claude session starts knowing what the previous ones already learned.

---

## Part 7: Adding Tools

An agent without tools is just a chatbot that charges you per token.

### 7.1 — Google Calendar

```bash
brew install gog  # Google Calendar CLI
gog auth keyring file  # use file-based keyring for headless operation
gog auth add your@email.com --services calendar --readonly
gog calendar list --json  # verify auth worked
```

### 7.2 — Apple Reminders Export

Create `~/scripts/export-reminders.sh`:

```bash
#!/bin/bash
/opt/homebrew/bin/remindctl show > /tmp/reminders-today.txt 2>&1
```

Add to the admin account's crontab (`crontab -e`):
```
0 6 * * * /Users/ADMIN_ACCOUNT/scripts/export-reminders.sh
```

### 7.3 — Morning Briefing

The briefing architecture evolves over time. The common path:

**Start simple: OpenClaw cron**

The easiest starting point is an OpenClaw cron that has the agent gather and format everything. It's two commands to set up and teaches you what the agent can actually access. Expect reliability to be imperfect — the agent may miss calendar events, produce generic headlines, or format things inconsistently. That's fine for learning; it's not fine if you're depending on it every morning.

```bash
openclaw cron add   --name "morning-briefing"   --cron "0 7 * * *"   --tz "America/New_York"   --model "anthropic/claude-haiku-4-5-20251001"   --session "isolated"   --agent "main"   --timeout 120   --message "Run the morning briefing: calendar, reminders, weather, top 3 headlines, podcasts. Send via iMessage. Under 20 lines."
```

Note `--session isolated` — this runs in a fresh session rather than appending to your main chat history.

**Graduate to: direct parallel fetching**

The more reliable architecture fetches each data source independently, in parallel, and only calls Haiku when you need synthesis. The ClawApp briefing screen can do this directly using `Promise.allSettled`:

- **Calendar** — use the device's native calendar API (`expo-calendar`) rather than routing through the agent
- **Reminders** — read from a pre-exported file written by a 6 AM cron on the admin account
- **Weather** — call `wttr.in` directly (free, no auth required: `https://wttr.in/YOUR_CITY?format=j1`)
- **Headlines** — an hourly cron writes fresh news to a shared JSON file; the app fetches it directly
- **Podcasts** — the scoring script writes to a shared JSON file; same pattern

Each source that fails is isolated — a broken headlines fetch doesn't affect calendar display. The screen renders as data arrives rather than waiting for everything.

**Google Calendar OAuth gotcha**

If you're using Google Calendar via a CLI tool (like `gog`), watch for this: Google projects in "Testing" mode revoke refresh tokens after 7 days. Your calendar access will silently stop working on a weekly cadence. Fix: publish your Google Cloud project (even if it's just for personal use). Go to Google Cloud Console → OAuth consent screen → Publish App. This gets you long-lived refresh tokens.

---

## Part 8: ClawApp — Your Native iOS Interface

ClawApp is a React Native/Expo iOS app that gives you a native interface to the agent. More importantly: it has a build screen that lets the agent build and deploy new versions of itself.

### 8.1 — Create the Repo

```bash
mkdir ClawApp && cd ClawApp
gh repo create YOUR_GITHUB_USERNAME/ClawApp --private --confirm
```

### 8.2 — Scaffold with Claude Code

Start a Claude Code session and use this prompt to generate the initial app:

```
Build a React Native Expo iOS app called ClawApp that connects to an OpenClaw agent gateway.

Tech stack: React Native, Expo SDK 52+, TypeScript, expo-router for navigation.

The app needs four screens accessible via a bottom tab navigator:

1. **Chat screen** — Main screen. Connects to the OpenClaw gateway via WebSocket at 
   ws://[TAILSCALE_IP]:18789. Shows messages in a chat interface (user right, agent left). 
   Streaming responses with a blinking cursor while the agent is typing. Haptic feedback 
   on send. Inverted FlatList so newest messages are at the bottom. Empty state with a 
   friendly illustration and "Message your agent to get started."

2. **Home screen** — System status dashboard. Shows: gateway connection status (green/red), 
   current model in use, today's cost from usage.cost RPC, and upcoming scheduled jobs from 
   cron.list RPC. Pull to refresh.

3. **Build screen** — App build management. A "Start Build" button that sends a message to 
   the agent to trigger a build. Polls a JSON file at http://[TAILSCALE_IP]:18793/clawapp-build.json 
   every 3 seconds and displays a stage timeline: Provisioning → Compiling → Packaging → 
   Distributing. When complete, shows an "Install Update" button that opens the EAS install link.

4. **Settings screen** — Shows gateway URL, connection status, app version.

Gateway connection:
- Connect on app launch
- Reconnect automatically on disconnect
- Show connection status in the header

Build the complete app with all four screens. Use Tailwind core utility classes for styling 
(no compile step). Export a default component from each screen. Create a README with setup 
instructions. Commit to main.
```

### 8.3 — EAS Setup

```bash
npm install -g eas-cli
eas login
eas build:configure  # select iOS
```

Configure `eas.json` for local builds:

```json
{
  "build": {
    "preview": {
      "ios": {
        "simulator": false,
        "distribution": "internal"
      }
    }
  }
}
```

### 8.4 — Local Builds (Free)

EAS cloud builds cost money. `eas build --local` builds on your Mac mini instead, but still distributes via EAS:

```bash
# Install build dependencies on Mac mini
brew install fastlane
xcode-select --install  # if Xcode isn't installed

# Trigger a local build
eas build --local --platform ios --profile preview --non-interactive
```

> **The key insight:** `eas build --local` uses your Mac mini's CPU and doesn't consume EAS build minutes. The artifact still uploads to EAS and you get the same install link. Zero compute cost, identical distribution experience.

**Tahoe codesigning — two files, not one**

If you hit codesigning errors on macOS Tahoe, `scripts/patch-eas-tahoe.sh` in this repo handles the fix. One thing not immediately obvious: there are **two** `keychain.js` files in the EAS cache that both need patching — one in `dist/ios/credentials/` and one in `dist/steps/utils/ios/credentials/`. The second has a different class structure. The patch script handles both, but if you're patching manually, don't stop at the first one.

**Build script self-overwrite bug**

If your build script runs `git checkout -- .` to reset the working directory, it can overwrite itself mid-execution since the script is tracked in git. Fix: have a wrapper script (`run-build.sh`) copy the build script to `/tmp` before executing it. The copy runs from `/tmp` and the git reset can't touch it.

### 8.5 — The Canonical Build Script

> ⚠️ **Do not have the agent manage the build process directly.** See the design principle at the top of this guide. AI agents are non-deterministic; build pipelines must not be.

The build script lives in the ClawApp repo at `scripts/build-clawapp.sh` and is version-controlled. The agent runs it with one command. It never rewrites it.

Here's what the canonical build script does:
1. Pulls latest from GitHub
2. Runs `npm install`
3. Applies macOS Tahoe codesigning patches if needed (`scripts/patch-eas-tahoe.sh`)
4. Runs `eas build --local --platform ios --profile preview --non-interactive`
5. Writes structured JSON progress at each stage to `/Users/Shared/clawapp-build.json`
6. Reads the install URL from `serve-build.js` (which manages the distribution tunnel)
7. Sends an iMessage with the install link

The JSON format the app polls every 3 seconds:

```json
{
  "buildId": "build-1711700000",
  "status": "in_progress",
  "branch": "main",
  "commitHash": "abc1234",
  "commitMessage": "feat: add haptic feedback",
  "startedAt": 1711700000000,
  "stage": "compile",
  "stages": {
    "git":      { "status": "complete",     "duration": 12 },
    "deps":     { "status": "complete",     "duration": 38 },
    "patch":    { "status": "complete",     "duration": 4  },
    "compile":  { "status": "in_progress",  "startedAt": 1711700054000 },
    "install":  { "status": "pending" }
  },
  "easUrl": null,
  "error": null
}
```

### 8.6 — serve-build.js: The Distribution Server

Rather than the build script managing its own cloudflare tunnel (which creates instability — tunnels crash, URLs change, the script kills tunnels it doesn't own), run a persistent Node.js server that manages the tunnel with auto-restart:

```javascript
// scripts/serve-build.js
// Serves build status JSON and manages cloudflare tunnel with auto-restart
// Run as a persistent background process, not invoked per-build
```

The full implementation is in this repo. It:
- Serves the OTA install page with HTTPS on port 18794 (for `itms-services://` iOS install links)
- Exposes a build HTTP API on port 18795: `/trigger-build`, `/build-status`, `/build-log`, `/tunnel-url`
- Manages a cloudflared tunnel with automatic restart on crash

> ⚠️ **Port conflict**: If you're running a Python server for podcast data on port 18793, keep it separate from serve-build.js. These two must not share a port — serve-build.js restarting will kick off the data server if they're on the same port. Use dedicated ports for each service.

The app's build screen polls `/build-status` (the API endpoint) — not a raw JSON file. The API returns a `_buildInProgress` flag from server memory, which means a newly triggered build shows as in-progress immediately even if the JSON file on disk still shows "complete" from the last build.

The build script reads the tunnel URL at the end:
```bash
INSTALL_URL=$(curl -sk https://localhost:18794/tunnel-url)
```

This decouples tunnel management from build execution. The tunnel is always running. The build script never touches it.

### 8.7 — Telling the Agent to Build

The agent's build instruction in `AGENTS.md` is deliberately simple:

```markdown
## Building ClawApp
When asked to "update ClawApp" or "build ClawApp":

Run EXACTLY:
  bash ~/Developer/ClawApp/scripts/build-clawapp.sh 2>&1

Do NOT:
- Rewrite the build script
- Run a different path or version of the script
- Manage cloudflared directly
- Run eas build commands manually

If something needs changing in the build process:
  Edit ~/Developer/ClawApp/scripts/build-clawapp.sh
  Commit to a branch → open a PR
  Merge the PR → then the next build uses the updated script

After running: read the install URL from the build JSON or from:
  curl -sk https://localhost:18794/tunnel-url

Send an iMessage with that URL.
```

This single rule — "run this script, never rewrite it" — is the most important thing you can put in AGENTS.md for the build pipeline. Everything else is recoverable. An agent rewriting a build script mid-session and running its own version is not.

**How to verify the agent is actually running your build**

A real iOS build takes 10-20 minutes. If a "build" completes in under 2 minutes, the agent is not running the canonical script — it's either faking the build (writing success status without executing anything), reusing an old artifact, or running inline steps it invented. The test: write a unique canary file before triggering a build and verify it exists in the compiled artifact. If the canary isn't there, the agent didn't build what you thought it built.

**How to verify the agent is actually running your build**

A real iOS build takes 10-20 minutes. If a "build" completes in under 2 minutes, the agent is not running the canonical script — it's either faking the build (writing success status without executing anything), reusing an old IPA, or running inline steps it invented. The test: write a unique canary file before triggering a build and verify it exists in the compiled artifact. If the canary isn't there, the agent didn't build what you thought it built.

---

## Part 9: Model Selection Guide

The right mental model for working with multiple Claude models:

**Haiku** — your everyday agent. Fast, cheap, handles tool calling, follows instructions reliably. Use for: all conversations, morning briefings, scheduled crons, anything where you need a consistent response.

**Sonnet** — your implementer. Give it a well-specified problem and it produces excellent code, documentation, and solutions. Use for: Claude Code sessions, complex one-off tasks where quality matters more than cost.

**Opus** — your diagnostician. Slower, more expensive, but genuinely better at reasoning through complex ambiguous problems. Use for: debugging sessions where you've been stuck for a while, architectural decisions, anything where Sonnet has given you wrong confident answers.

> **The pattern:** Haiku runs your life. Sonnet builds your tools. Opus untangles your messes.

**Ollama** — your free background worker. Handles heartbeats, watchdog crons, simple scheduled checks. Accepts that it will occasionally fail silently (thinking token bugs exist in some models). Never use for conversations a human is waiting on.

### When to Bypass the Agent Entirely

You'll eventually find yourself calling the Anthropic API directly — bypassing the agent entirely — for certain tasks. That's not a sign something is wrong. It's the right call.

**Call the Anthropic API directly when:**
- You have structured data that needs scoring or classification (podcast episodes, search results)
- You need consistent, repeatable output format
- The task is part of a script that runs on a schedule
- You want the call to be free of session context and conversation history

```javascript
// From podcast-digest.js — calling Haiku directly, not through OpenClaw
const response = await fetch('https://api.anthropic.com/v1/messages', {
  method: 'POST',
  headers: { 'x-api-key': process.env.ANTHROPIC_API_KEY, 'anthropic-version': '2023-06-01' },
  body: JSON.stringify({
    model: 'claude-haiku-4-5-20251001',
    max_tokens: 2048,
    messages: [{ role: 'user', content: scoringPrompt }]
  })
});
```

**Read from OpenClaw via RPC when:**
- You need structured system data: cron status, usage cost, session health
- ClawApp needs to display agent state without starting a conversation
- You want data without consuming a conversation turn

**Use the agent (OpenClaw conversation) when:**
- The task requires judgment, context, or reasoning about ambiguous things
- You're asking an open question without a known structure to the answer
- The response needs to draw on memory, prior context, or persona

The pattern that emerges: OpenClaw is the message bus and conversation layer. Direct API calls are intelligence on demand. Scripts are deterministic infrastructure. ClawApp is the interface that reaches for the right one.

---

## Part 10: The AGENTS.md Pattern

Honestly, this is the thing I'd tell people about first if I were explaining this setup in person.

Every time you discover something important — a config key that doesn't exist, a failure mode you didn't anticipate, a rule that prevents something from going wrong — write it to `AGENTS.md`. Every Claude session starts by reading this file. Every lesson you encode there is a lesson the next session doesn't have to learn the hard way.

Every rule in that file exists because something went wrong. The more specific you are about the incident, the better the file works — a vague reminder like 'be careful with databases' doesn't help. 'Never delete chat.db — use sqlite3 for targeted cleanup' does.

Some examples of what ends up in a mature `AGENTS.md`:

- "Never delete databases. Use targeted queries instead."
- "Always verify a configuration key is real before spending time debugging it."
- "When a scheduled job doesn't arrive, assume it's failing silently before assuming you missed it."
- "AI will sometimes choose the nuclear option when you needed a scalpel. Slow down for irreversible actions."

---

## When Things Break

They will. Here's where to start.

### Agent stopped responding to iMessages

```bash
# Check if the gateway is running
sudo launchctl list | grep openclaw

# Check what the last response was
cat ~/.openclaw/agents/main/sessions/*.jsonl | grep '"assistant"' | tail -3

# Look for content:[] (Ollama thinking bug) or NO_REPLY (session issue)
tail -50 /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log
```

### Messages come from the wrong account (wrong email instead of your phone number)

BlueBubbles is probably running on the wrong account (your admin account). Verify it's running on the agent account only:

```bash
# Check which account is running BlueBubbles
ps aux | grep BlueBubbles
```

If it shows your admin account username, stop it there, switch to the agent account, and start it there.

### Silent cron failures

Check the cron list for error reasons:
```bash
openclaw cron list
```

Common causes:
- Wrong model name (e.g., `claude-haiku-4-5` instead of `claude-haiku-4-5-20251001`)
- Missing `agentId` or `sessionKey` in the cron configuration
- Ollama thinking token bug on complex tasks — switch to Haiku for that cron
- Model timeout on complex multi-tool crons — increase `--timeout` and verify the cron isn't silently succeeding with empty output

### Podcast/media sync not updating on headless Mac mini

If you're using a media app that syncs via iCloud and the Mac mini runs headless, the app may never sync because it only syncs when actively running in the foreground. Fix: add a cron that briefly opens the app before your processing script runs, waits a few minutes for sync, then closes it.
- Model timeout on complex multi-tool crons — increase `--timeout` and verify the cron isn't silently succeeding with empty output

### Podcast/media sync not updating on headless Mac mini

If you're using a media app that syncs via iCloud (podcasts, music, etc.) and the Mac mini runs headless (no GUI session), the app may never sync because it only syncs when actively running in the foreground. Fix: add a cron that opens the app briefly before your processing script runs, waits a few minutes for sync to complete, then closes it. This is inelegant but reliable.

### EAS local build fails

```bash
# Clean old build artifacts first
rm -rf ~/Developer/ClawApp/ios/build

# Check Xcode is installed
xcode-select -p

# Re-run patches if on macOS Tahoe
bash ~/Developer/clawapp-dist/scripts/patch-eas-tahoe.sh

# Try the build
eas build --local --platform ios --profile preview
```

---

## Keeping Costs Under Control

Check your API spend at [console.anthropic.com](https://console.anthropic.com) → Usage. Do this weekly until you trust the setup.

What to expect when things are running right: under $0.10 on a quiet day, under $0.50 on a day where you actually use it, $5–10 for the month. If you're spending more than that, the most likely causes are a cron job that should run on Ollama but is hitting Haiku, or a MEMORY.md file that's grown so large it consumes tokens on every session start.

The single biggest cost mistake is leaving Sonnet as the primary model. Switch to Haiku for everything except sessions where you specifically need heavier reasoning.

---

## What to Build Next

Every capability below follows the same basic pattern: a script gathers the data, Haiku (called directly or through the agent) does the reasoning, the result shows up in your app or via iMessage. Once you've built one of these, the rest get easier fast.

### Briefings and Digests

**Morning briefing** — the one covered in this guide. Calendar, reminders, weather, top 3 headlines, podcast recommendations. Runs at 7 AM without asking.

**Evening wind-down** — a 9 PM cron that reads what happened today, surfaces anything unfinished, and tells you what's on tomorrow's calendar. Two minutes before bed instead of checking your phone manually.

**Weekly review** — Sunday evening summary. What you did this week, what carried over, what's coming next week. Useful if you keep any kind of personal or professional tracking.

**Pre-meeting prep** — send the agent a calendar event name, get back a quick briefing: who's in the meeting, what you've discussed before, any relevant context from your notes. Build this once and you'll use it constantly.

### Monitors and Alerts

**Podcast relevance scoring** — Query your podcast app's database for unplayed episodes, score them against your interests via Haiku, surface the top few in your morning briefing. The post describes this in detail. It's one of the most satisfying things to get working.

Once the basic scorer is working, three improvements make it significantly more accurate:

- **Listening history feedback loop** — Query your podcast app's play history to compute per-show engagement rates. A show you subscribe to but never finish shouldn't score the same as one you listen to religiously. Feed this as behavioral context into the Haiku scoring prompt. The scoring stack becomes: `AI relevance × age decay × episode length penalty × listening affinity multiplier`.
- **Age decay** — Newer episodes should score higher, but not so aggressively that a 3-week-old episode from a show you love gets buried. A gentle curve works well for a 30-day digest window.
- **Dismiss feature** — Let users swipe away episodes they've decided to skip. Store dismissed IDs in a shared file; the scoring script filters them out on the next run.

**Podcast discovery** — A separate weekly cron can search for new episodes from shows you don't subscribe to but would probably like, scored with the same algorithm. Surface them in a "Discover" section separate from your regular queue.

Once the basic scorer is working, three improvements make it significantly more accurate:

- **Listening history feedback loop** — Query your podcast app's play history to compute per-show engagement rates (episodes started vs. completed). A show you subscribe to but never finish shouldn't score the same as one you listen to religiously. Feed this data as a behavioral context block into the Haiku scoring prompt. The scoring stack becomes: `AI relevance score × age decay × episode length penalty × listening affinity multiplier`.
- **Age decay** — Newer episodes should score higher than old ones, but not so aggressively that a 3-week-old episode from your favorite show gets buried. A gentle curve (full score for days 1-5, tapering to a 0.5x floor by day 20) works well for a 30-day digest window.
- **Dismiss feature** — Let users swipe away episodes they've already decided to skip. Store dismissed episode IDs in a shared file; the scoring script filters them out on the next run. This keeps the queue clean without requiring a database.

**Podcast discovery** — A separate weekly cron searches for new episodes from shows you don't subscribe to but would probably like, based on your interest profile. Score them with the same algorithm. Surface them in a "Discover" section separate from your regular queue. The best finds come from shows adjacent to ones you already love at high engagement rates.

**Price drop alerts** — script that checks specific product pages or uses a price tracking API. Alert when something you're watching drops below your threshold.

**News monitor** — instead of generic "top headlines," give it specific topics or companies. Run it nightly, only alert when something genuinely relevant appears.

**Calendar gap detector** — script that looks at your week and surfaces open time you might want to protect or use intentionally. "You have three hours free Tuesday afternoon. Want me to suggest something?"

**Location-aware context** — have ClawApp send your current city every time you open it. Your morning briefing adjusts for local weather. Your agent knows if you're traveling and can factor that in.

### Summarizers

**Email digest** — a daily or twice-daily summary of your inbox, surfacing emails that need a response and filtering out noise. Works well as a direct Haiku call: pass in subject lines and snippets, get back a prioritized summary.

**PR / code review summary** — if you work with GitHub, a morning digest of open PRs needing attention, recent commits, and CI status. Five minutes of context instead of opening GitHub and clicking around.

**Document summarizer** — a quick-action in ClawApp that lets you paste in a long document or article and get the key points back. Useful more often than you'd think.

**Meeting notes cleanup** — paste rough notes from a meeting, get back a structured summary with action items. Saves the "I need to process these notes later" procrastination.

### Recommenders

**Book recommendations** — feed the agent your reading history and interests, ask it weekly for what to read next. It'll get better at your taste over time as you react to its suggestions.

**Restaurant suggestions** — when you're in a city (your location file knows), ask it to recommend somewhere based on what you're in the mood for. It can factor in time of day, who you're with, cuisine preferences.

**Weekend plans** — Friday afternoon cron that looks at your weekend calendar gaps, checks the weather, and suggests a few things worth doing. It should know enough about you by this point to make it personal.

**Vision-based inventory** — Claude's vision API (Haiku, ~$0.02/image) can identify objects from photos. Take a photo of a shelf or a collection — Haiku identifies what's there and can make recommendations based on what it finds. Useful for anything where "what do I have?" is a recurring question.

**Vision-based inventory** — Claude's vision API (Haiku, ~$0.02/image) can identify objects from photos. Take a photo of a shelf, a pantry, a collection — Haiku identifies what's there, estimates quantities, and can make recommendations based on what it finds. The latency is low enough to use in real time from the app. Useful for anything where "what do I have?" is a recurring question.

---

Every one of these starts the same way: open a Claude conversation, describe what you want, let it help you design the approach, then use Claude Code to build the script. The first one takes the longest. After that, you know the pattern.

---

## The real payoff

None of the capabilities above are the point. The point is that once the infrastructure is working, the cost of trying something new is almost zero. You have an idea Saturday morning. You describe it to Claude, design the approach together, let Claude Code build the script, add a cron, and by Saturday evening it's running. Sunday you see how it actually works in practice. Monday you adjust it.

That feedback loop — measured in hours, not sprints — is what changes how you think about what's worth building. Things you'd never propose in a work setting because the effort isn't justified become worth trying on a Saturday afternoon. Most of them won't stick. Some will become things you use every day. You won't know which is which until you try.

That's the experiment this guide makes possible.

---

## Contributing

If you build your own version and run into something not covered here — a failure mode, a better approach, a tool integration that worked well — pull requests are welcome. This guide got better by being used, and the same will be true of yours.

---

*Built by Craig Oley. Companion post: [The Era of Personalized Software Is Here](https://medium.com/@craigoley)*
