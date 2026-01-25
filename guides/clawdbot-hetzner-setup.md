# Clawdbot + Hetzner Setup Guide

> Deploy your personal AI assistant on a VPS with Telegram integration

## Overview

Clawdbot is a personal AI assistant that runs on your own server and communicates through Telegram, WhatsApp, or other messaging apps. Unlike ChatGPT or Claude's web interface, it's always on, remembers context, and can proactively reach out to you.

**What you'll get:**
- 24/7 AI assistant accessible via Telegram
- Full control over your data and configuration
- Claude Opus 4.5 with 200k context window
- ~€5.49/month total cost

**Time required:** 15-20 minutes

## Prerequisites

- Hetzner Cloud account ([sign up](https://www.hetzner.com/cloud))
- SSH key on your local machine
- Telegram account
- Anthropic API key OR Claude Max subscription with OAuth token

## Part 1: Create Hetzner Server

### 1.1 Generate SSH Key (if needed)

**Windows (PowerShell):**
```powershell
ssh-keygen -t ed25519 -C "your@email.com"
# Press Enter to accept default location
# View public key:
Get-Content ~/.ssh/id_ed25519.pub
```

**macOS/Linux:**
```bash
ssh-keygen -t ed25519 -C "your@email.com"
cat ~/.ssh/id_ed25519.pub
```

### 1.2 Create Server in Hetzner Console

1. Log into [Hetzner Cloud Console](https://console.hetzner.cloud/)
2. Create or select a project
3. Click **Add Server**
4. Configure:

| Setting | Value |
|---------|-------|
| **Location** | Falkenstein or Nuremberg (Germany) |
| **Image** | Ubuntu 24.04 |
| **Type** | Shared vCPU → Cost-Optimized → **CX33** (4 vCPU, 8GB RAM) |
| **Networking** | Public IPv4: ON, Public IPv6: ON |
| **SSH Key** | Paste your public key |
| **Name** | `clawdbot` (or your preferred name) |

5. Click **Create & Buy Now** (~€5.49/mo)
6. Wait ~30 seconds, then copy the IPv4 address

## Part 2: Server Setup

### 2.1 Connect via SSH

```bash
ssh root@YOUR_SERVER_IP
# Type 'yes' when asked about fingerprint
```

### 2.2 Update System

```bash
apt update && apt upgrade -y
```

### 2.3 Install Node.js 22

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs
```

Verify installation:
```bash
node -v  # Should show v22.x.x
```

### 2.4 Install Clawdbot

```bash
npm install -g clawdbot@latest
```

## Part 3: Configure Clawdbot

### 3.1 Get Your Credentials

**Telegram Bot Token:**
1. Open Telegram, search for `@BotFather`
2. Send `/newbot`
3. Follow prompts to name your bot
4. Copy the bot token (format: `123456789:ABC...XYZ`)

**Telegram User ID:**
1. Search for `@userinfobot` in Telegram
2. Send any message
3. Copy your numeric user ID

**Anthropic Auth (choose one):**

*Option A: API Key*
- Get from [console.anthropic.com](https://console.anthropic.com/)
- Starts with `sk-ant-api...`

*Option B: Claude Max OAuth Token (recommended for subscribers)*
- Run `claude setup-token` on your local machine with Claude Code CLI
- Copy the OAuth token (starts with `sk-ant-oat...`)

### 3.2 Run Setup

**With API Key:**
```bash
clawdbot onboard \
  --non-interactive \
  --accept-risk \
  --anthropic-api-key "sk-ant-api..." \
  --install-daemon \
  --skip-channels \
  --skip-skills
```

**With OAuth Token:**
```bash
clawdbot onboard \
  --non-interactive \
  --accept-risk \
  --auth-choice token \
  --token-provider anthropic \
  --token "sk-ant-oat01-..." \
  --install-daemon \
  --skip-channels \
  --skip-skills
```

### 3.3 Enable Telegram Plugin

```bash
clawdbot plugins enable telegram
clawdbot gateway restart
```

### 3.4 Add Telegram Channel

```bash
clawdbot channels add \
  --channel telegram \
  --token "YOUR_BOT_TOKEN"
```

### 3.5 Restrict Access (Security)

Only allow your Telegram user ID to interact with the bot:

```bash
clawdbot config set channels.telegram.allowFrom '[YOUR_USER_ID]'
clawdbot gateway restart
```

### 3.6 Verify Setup

```bash
clawdbot status
```

Expected output should show:
- Gateway: running
- Telegram: ON / OK

## Part 4: Test Your Bot

1. Open Telegram
2. Find your bot by username
3. Send a message
4. You should receive a response from Claude

## Useful Commands

### Server Management

```bash
clawdbot status              # Check overall status
clawdbot status --deep       # Detailed channel status
clawdbot logs --follow       # View live logs
clawdbot gateway restart     # Restart the bot
clawdbot health              # Run health checks
```

### Chat Commands (send in Telegram)

| Command | Description |
|---------|-------------|
| `/new` | Start a fresh conversation |
| `/model` | Switch AI models |
| `/compact` | Compress long conversations |
| `stop` | Cancel a running task |

### Configuration

```bash
clawdbot config get channels          # View channel config
clawdbot config set <path> <value>    # Update config
```

## Running Multiple Instances

Clawdbot supports **profiles** for running multiple isolated instances on the same server.

### Option 1: Multiple Profiles (Same Server)

```bash
# Create profile for project A
clawdbot --profile projectA onboard --non-interactive --accept-risk ...

# Create profile for project B
clawdbot --profile projectB onboard --non-interactive --accept-risk ...

# Manage specific profile
clawdbot --profile projectA status
clawdbot --profile projectB gateway restart
```

Each profile has isolated:
- Config: `~/.clawdbot-<profile>/clawdbot.json`
- State: `~/.clawdbot-<profile>/`
- Gateway port (configure different ports)

### Option 2: Separate Servers (Recommended for Production)

**When to use separate servers:**
- High load or resource-intensive tasks
- Complete isolation (different clients/projects)
- Different geographic regions
- Independent update/rollback cycles
- Different security requirements

**Cost comparison:**
| Setup | Monthly Cost |
|-------|-------------|
| 1 server, multiple profiles | ~€5.49 |
| 2 separate CX33 servers | ~€10.98 |
| 1 CX33 + 1 CX23 (lighter load) | ~€8.48 |

## Troubleshooting

### Bot not responding?

```bash
clawdbot status --all
clawdbot logs --follow
```

### Gateway won't start?

```bash
# Check if port is in use
netstat -tlnp | grep 18789

# Force restart
clawdbot gateway --force
```

### Need to redo setup?

```bash
clawdbot reset
clawdbot onboard --install-daemon
```

### SSH asking for password?

Your key isn't being used. Specify it explicitly:
```bash
ssh -i /path/to/your/key root@YOUR_SERVER_IP
```

### Token expired (OAuth)?

Regenerate token on your local machine:
```bash
claude setup-token
```

Then update on server:
```bash
clawdbot config set auth.profiles.anthropic:manual.token "new-token"
clawdbot gateway restart
```

## Security Best Practices

1. **Always set `allowFrom`** to restrict who can message your bot
2. **Use SSH keys**, never password authentication
3. **Keep Node.js updated**: `apt update && apt upgrade -y`
4. **Monitor logs** for unusual activity: `clawdbot logs --follow`
5. **Use firewall** if exposing additional ports

## Customization

### Personality (SOUL.md)

Edit `~/clawd/SOUL.md` to customize your bot's personality:

```bash
nano ~/clawd/SOUL.md
```

### User Context (USER.md)

Add personal info the bot should know:

```bash
nano ~/clawd/USER.md
```

### Skills

Browse and install skills from [clawdhub.com](https://clawdhub.com):

```bash
clawdbot skills install <skill-name>
```

## Resources

- [Clawdbot Docs](https://docs.clawd.bot/)
- [Clawdbot Discord](https://discord.gg/clawd)
- [Hetzner Cloud Docs](https://docs.hetzner.com/cloud/)
- [Anthropic Console](https://console.anthropic.com/)

---

*Guide created with Claude Code. Last updated: January 2026*
