# SETUP — discord_agents end-to-end deployment guide

This document walks through every step needed to run a multi-bot Claude
Code research mesh over Discord, on a fresh machine. It covers macOS,
Linux, WSL2, and native Windows. Read top to bottom on first deploy;
skim later for the section you need.

## Table of contents

- [What you'll have at the end](#what-youll-have-at-the-end)
- [How it works in 60 seconds](#how-it-works-in-60-seconds)
- [Prerequisites](#prerequisites)
- [Quickstart (TL;DR for the impatient)](#quickstart-tldr-for-the-impatient)
- [Part 1 — One-time bootstrap (per machine)](#part-1--one-time-bootstrap-per-machine)
  - [1.A macOS / Linux](#1a-macos--linux)
  - [1.B Windows + WSL2 (recommended)](#1b-windows--wsl2-recommended)
  - [1.C Windows native (PowerShell)](#1c-windows-native-powershell)
- [Part 2 — Create a Discord bot Application](#part-2--create-a-discord-bot-application)
- [Part 3 — Invite the bot to your server](#part-3--invite-the-bot-to-your-server)
- [Part 4 — Run /agent-setup to scaffold the agent](#part-4--run-agent-setup-to-scaffold-the-agent)
- [Part 5 — First launch: configure token and pair](#part-5--first-launch-configure-token-and-pair)
- [Part 6 — Add the bot to peer-bots.json](#part-6--add-the-bot-to-peer-botsjson)
- [Part 7 — Roles and CLAUDE.md per agent](#part-7--roles-and-claudemd-per-agent)
- [Migrating to a new machine](#migrating-to-a-new-machine)
- [Token transfer between machines](#token-transfer-between-machines)
- [Troubleshooting](#troubleshooting)
- [Architecture deep-dive](#architecture-deep-dive)
- [Security model](#security-model)
- [Reference: file layout](#reference-file-layout)

---

## What you'll have at the end

- N concurrent Claude Code sessions, each one a separate Discord bot
  (separate identity, separate token, separate state)
- Each bot can be `@`-mentioned in shared Discord channels and replies
  through its own Claude Code instance
- Bots can `@`-mention each other (patched discord plugin), enabling
  multi-agent research workflows (Scout / Orchestrator / Synthesizer /
  Critic patterns)
- Each bot has its own long-term memory, role spec, and allowlist
- One slash command (`/agent-setup <name>`) scaffolds a new bot end-to-end

The architecture supports running 2 bots on one machine, or 4 bots
spread across two machines, etc. — they only need a shared
`peer-bots.json` to recognize each other.

## How it works in 60 seconds

Claude Code has a `--channels` flag that connects a session to an
external message channel (the Discord plugin in this case). The plugin
runs a Node.js child process that authenticates to Discord with a bot
token and pipes inbound messages into the Claude Code session.

To run multiple bots concurrently on the same machine, each Claude Code
session needs:

1. Its own working directory (project-level `CLAUDE.md` role definition).
2. Its own `CLAUDE_CONFIG_DIR` (separate plugins, memory, settings).
3. Its own `DISCORD_STATE_DIR` (separate Discord bot token, allowlist,
   approved DM mappings).
4. Its own Discord bot Application (separate token; Discord rejects
   concurrent gateway connections with the same token).

The `/agent-setup` skill in `.claude/skills/agent-setup/` automates the
local filesystem side of this — directories, plugin cache copy, SKILL.md
patches, settings.json, and the shell alias.

The plugin's `server.ts` is patched to (a) honor `$DISCORD_STATE_DIR`
instead of hardcoding `~/.claude/channels/discord/`, and (b) allow
inbound messages from bot accounts listed in `shared/peer-bots.json`
(the upstream plugin drops all bot-authored messages by default).

## Prerequisites

| What | Why | How |
|------|-----|-----|
| Discord account | Obvious | discord.com |
| Discord server you own or admin | Need to invite bots | Right-click → Create Server, or have admin on existing one |
| Claude Code | Runs the bots | `curl -fsSL https://claude.ai/install.sh \| bash` on macOS/Linux, `irm claude.ai/install.ps1 \| iex` in PowerShell on Windows |
| Claude Max or API plan | Long-running sessions | Subscribe at claude.ai |
| Node.js 18+ | Discord plugin runs on Node | nodejs.org or apt/brew |
| Git | Clone repo, push your fork | apt/brew or git-scm.com |

## Quickstart (TL;DR for the impatient)

After the one-time bootstrap (see Part 1):

```bash
cd ~/Projects && git clone <this-repo> agents_team && cd agents_team
claude
# inside the Claude session:
/agent-setup claude-a
/exit
# back in the shell:
source ~/.bashrc   # or ~/.zshrc
bot-a
# inside the new bot-a session:
/discord:configure <PASTE_BOT_TOKEN>
/exit
bot-a
/mcp   # should show plugin:discord:discord connected
```

Repeat the `/agent-setup <name>` + `bot-<name>` + `/discord:configure`
cycle for each additional bot.

Reading the rest of this doc gets you out of trouble when this
quickstart doesn't "just work."

---

## Part 1 — One-time bootstrap (per machine)

Pick the section matching your OS. You only do this once per physical
machine; after that, adding bots is a single `/agent-setup` call.

### 1.A macOS / Linux

```bash
# Tools
# macOS: brew install node git
# Debian/Ubuntu:
sudo apt update
sudo apt install -y curl git build-essential ca-certificates

# Node.js 20 LTS (Debian/Ubuntu)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Claude Code
curl -fsSL https://claude.ai/install.sh | bash
# Add ~/.local/bin to PATH if the installer says so:
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
claude --version    # should print 2.x.x

# Log in
claude
# follow the OAuth flow (paste callback code), then:
/plugin marketplace add anthropics/claude-plugins-official
/plugin install discord@claude-plugins-official
# scope: user
/exit

# Verify plugin landed
ls ~/.claude/plugins/cache/claude-plugins-official/discord/
# should show a version directory (e.g. 0.0.4)
```

You're done with bootstrap. Proceed to [Part 2](#part-2--create-a-discord-bot-application).

### 1.B Windows + WSL2 (recommended)

Native Windows works but WSL2 is the smoother path because the existing
`/agent-setup` skill uses POSIX paths and bash. If you don't already
have WSL2:

```powershell
# In PowerShell (administrator)
wsl --install -d Ubuntu
# Reboot if asked
```

Check it's there:

```powershell
wsl -l -v
# should list Ubuntu  Stopped  2
```

Enter Ubuntu:

```powershell
wsl
```

Critical: **don't stay as root.** Ubuntu on WSL sometimes drops you
into a root prompt if you skipped first-run user creation. If your
prompt shows `root@...`, create a non-root user before continuing:

```bash
adduser <username>             # e.g. adduser jbo
# follow prompts: set password, leave gecos fields blank
usermod -aG sudo <username>

# Make WSL default to this user from now on:
sudo tee /etc/wsl.conf <<EOF
[user]
default=<username>
EOF
```

Then in PowerShell:

```powershell
wsl --shutdown
wsl
```

Your prompt should now be `<username>@<machine>:~$`. **Why:** Claude
Code refuses `--dangerously-skip-permissions` when running as root,
which is the flag the autonomous bots need. Non-root user side-steps
this.

Inside the WSL user shell, follow [1.A macOS / Linux](#1a-macos--linux)
verbatim — apt + Node + Claude Code + plugin install. Then proceed to
Part 2.

### 1.C Windows native (PowerShell)

If you don't want WSL, the `/agent-setup-windows` skill mirrors
`/agent-setup` in pure PowerShell. Bootstrap differences:

```powershell
# Claude Code
irm claude.ai/install.ps1 | iex

# Node.js 20 LTS
winget install OpenJS.NodeJS.LTS

# Project root
New-Item -ItemType Directory -Force "$env:USERPROFILE\Projects\agents_team\shared" | Out-Null

# Plugin install (same as macOS)
claude
# session:
/plugin marketplace add anthropics/claude-plugins-official
/plugin install discord@claude-plugins-official
/exit
```

Native Windows ergonomics caveats:
- Launchers go to `$PROFILE` as PowerShell functions (not aliases),
  since PS aliases can't carry env vars.
- The plugin server.ts patch writes a Windows-shaped peer-bots path
  via `$env:USERPROFILE`.
- File ACLs replace `chmod 600`; the skill applies `icacls /inheritance:r`.
- If `$PROFILE` script doesn't load, run
  `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`.

After bootstrap, you use `/agent-setup-windows <name>` instead of
`/agent-setup <name>`. The rest of this doc shows the Unix flow; for
Windows, see `.claude/skills/agent-setup-windows/SKILL.md` for the
PowerShell equivalents.

---

## Part 2 — Create a Discord bot Application

Each bot needs its own Discord Application (and its own token —
Discord rejects two concurrent gateway connections with the same
token).

1. Visit <https://discord.com/developers/applications>.
2. Click **New Application** in the top right. Name it (e.g.
   `claude-a`). Accept the ToS.
3. Left sidebar → **Bot**.
4. Click **Reset Token**. Confirm. **Copy the token immediately** —
   you cannot see it again. Save it somewhere reachable from your
   machine (password manager / encrypted note).
5. On the same Bot page, scroll to **Privileged Gateway Intents**.
   Enable **MESSAGE CONTENT INTENT**. Without this, the bot can't
   read message text.
6. Optional: rename and avatar the bot, set "Public Bot" off if you
   only want yourself to invite it.

Treat the token like a password: it gives full control over the bot.
Anyone with it can impersonate the bot, scrape DMs, etc.

## Part 3 — Invite the bot to your server

1. Left sidebar → **Installation** (or **OAuth2** → **URL Generator**
   in older UI).
2. Set Install Link to **Discord Provided Link** or generate one with
   scopes:
   - **bot** (required)
   - **applications.commands** (only if you plan to add slash commands
     directly to Discord; not required for the channel plugin pattern)
3. Permissions (minimum useful set):
   - View Channels
   - Send Messages
   - Read Message History
   - Use External Emojis
   - Add Reactions
   - Attach Files
   - Embed Links
4. Copy the Install Link, open it in a browser. Select your server,
   authorize, complete the captcha. The bot should appear in your
   server's member list (offline until you launch Claude Code).

Repeat Parts 2 and 3 for each bot you want. They don't have to all be
created up front — `/agent-setup` doesn't care.

## Part 4 — Run /agent-setup to scaffold the agent

In any Claude Code session opened from the `agents_team` directory:

```
/agent-setup <name> [<alias>]
```

- `<name>`: agent directory name. Must match `[A-Za-z0-9_-]+`. Use
  `claude-a`, `claude-b`, … or descriptive names like `synthesizer`.
- `<alias>` (optional): shell command to launch this bot. If omitted,
  defaults to `bot-` + the name with a leading `claude-` stripped:
  - `/agent-setup claude-a` → alias `bot-a`
  - `/agent-setup synthesizer` → alias `bot-synthesizer`
  - `/agent-setup claude-a chat-with-a` → alias `chat-with-a`

What the skill does (idempotent — safe to re-run):

1. Creates `agents/<name>/.agent-config/channels/discord/{approved,}`
   and other subdirectories.
2. Copies the Discord plugin from your user-level Claude install
   (`~/.claude/plugins/...`) into the agent's local plugin cache.
3. Patches the agent's local `skills/access/SKILL.md` and
   `skills/configure/SKILL.md` to honor `$DISCORD_STATE_DIR` instead
   of hardcoding `~/.claude/channels/discord/`.
4. Patches the agent's local `server.ts` to allow inbound bot-to-bot
   messages from `peer-bots.json`, with per-sender rate limiting.
5. Writes plugin metadata (`installed_plugins.json`,
   `known_marketplaces.json`) with absolute paths.
6. Writes `settings.json` with:
   - `enabledPlugins.discord@claude-plugins-official: true`
   - `permissions.allow` listing all five `mcp__plugin_discord_discord__*`
     tools (so the bot doesn't prompt for permission on every reply)
7. Writes a `CLAUDE.md` role template (skipped if one already exists).
8. Appends a launch alias to `~/.bashrc` or `~/.zshrc` of the form:
   ```bash
   alias <alias>='cd <agent-dir> && \
       DISCORD_STATE_DIR="$PWD/.agent-config/channels/discord" \
       CLAUDE_CONFIG_DIR="$PWD/.agent-config" \
       PEER_BOTS_FILE="<project>/shared/peer-bots.json" \
       claude --dangerously-skip-permissions \
              --channels plugin:discord@claude-plugins-official'
   ```

When the skill prints its checklist, reload your shell:

```bash
source ~/.bashrc        # or ~/.zshrc
```

You should see the new alias in `alias | grep ^bot-`.

## Part 5 — First launch: configure token and pair

```bash
bot-<name>
```

Inside the Claude session that opens:

```
/discord:configure <PASTE_THE_TOKEN_FROM_PART_2>
```

This writes the token into the agent's
`.agent-config/channels/discord/.env`. The plugin server reads `.env`
at startup, so:

```
/exit
```

Re-launch:

```bash
bot-<name>
```

Verify the plugin connected:

```
/mcp
```

You should see `plugin:discord:discord` connected (or the equivalent
green/OK status). If it shows `Failed to reconnect ... -32000`, the
plugin server is exiting because of a misconfigured `.env` or a path
issue — jump to [Troubleshooting](#troubleshooting).

Next, pair yourself so the bot will reply to you:

1. In Discord, send the bot a DM (any message). The bot will reply
   with a 6-character code.
2. Back in the Claude session:
   ```
   /discord:access pair <code>
   ```
   This writes `approved/<your-user-id>` (mapping your user ID to your
   DM channel ID) and adds you to `allowFrom`.
3. Lock the policy:
   ```
   /discord:access policy allowlist
   ```
   From now on, new senders can't trigger pairing — only people you
   explicitly allow can DM the bot.

For each group channel where the bot should respond to `@`-mentions:

```
/discord:access group add <GROUP_CHANNEL_ID>
```

(Right-click the channel in Discord with Developer Mode on → Copy
Channel ID.)

Verify all of this with:

```
/discord:access
```

## Part 6 — Add the bot to peer-bots.json

For bot-to-bot `@`-mentions to work, each bot has to know the others
exist. Edit `shared/peer-bots.json` to include the new bot's Discord
**Application ID** (not the token!):

```json
{
  "_comment": "Discord bot IDs allowed to message each other.",
  "bots": [
    { "id": "<BOT_A_APPLICATION_ID>", "name": "claude-a", "role": "scout+orchestrator" },
    { "id": "<BOT_B_APPLICATION_ID>", "name": "claude-b", "role": "synthesizer" },
    { "id": "<BOT_C_APPLICATION_ID>", "name": "claude-c", "role": "critic" }
  ],
  "rateLimitPerMin": 30,
  "rateLimitNote": "Per-sender hard cap to prevent runaway loops."
}
```

You find the Application ID at <https://discord.com/developers/applications>
→ your app → **General Information** → Application ID (the long number
at the top). It's the same as the bot's user ID in chat.

The plugin re-reads `peer-bots.json` on every inbound message, so
**no restart needed** — just save the file and the change is live for
all bots that share it.

## Part 7 — Roles and CLAUDE.md per agent

Each `agents/<name>/CLAUDE.md` is loaded as project-level context when
that bot starts. The `/agent-setup` skill writes a minimal template;
customize it heavily.

Useful structure (adapt freely):

```markdown
# <agent-name>

You are <Role> in the discord_agents research mesh. Operator is
`<@<OPERATOR_USER_ID>>`. Other bots are siblings under `agents/`;
cross-agent reads of `.agent-config/` are permitted.

## Identity
- Config dir: `agents/<agent-name>/.agent-config/`
- Discord state: `$DISCORD_STATE_DIR`
- Shared resources: `../../shared/`

## Role
<paragraph of role definition — what this bot does, what discipline
it must follow, what it must NOT do>

Full spec: `.agent-config/role.md` (read it before any task work).

## Conventions
- Don't read or write other agents' `.agent-config/` casually.
- Cross-agent messages go through Discord with explicit `<@id>` mentions.
- Long-term memory is in `.agent-config/memory/`. Use it.
- Every Discord message ends with `NEXT: <@user-id>` + `REASON: <one line>`.
- Use raw `<@id>` mention syntax — plain "@Name" doesn't trigger
  notifications.
```

For the actual research-discipline content (Orchestrator handoff
rules, Synthesizer L0/L1-L5 structure, Critic protocol, etc.), see the
example role specs that ship in the repo:

- `agents/claude-a/.agent-config/role.md` (Orchestrator)
- `agents/claude-b/.agent-config/projects/.../memory/role_synthesizer.md` (Synthesizer)
- `agents/claude-d/.agent-config/projects/.../memory/role_scout.md` (Scout)

---

## Migrating to a new machine

Goal: take the whole setup from machine A to machine B with minimum
re-work.

What git carries cleanly (already committed in this repo):

- `agents/<name>/CLAUDE.md`
- `agents/<name>/.agent-config/role.md` (where present)
- `agents/<name>/.agent-config/projects/*/memory/*.md`
- `agents/<name>/.agent-config/channels/discord/access.json` (allowlist
  / groups — non-credential, public Discord IDs)
- `shared/peer-bots.json`
- `.claude/skills/`

What you regenerate on machine B (don't try to copy these):

- The plugin cache (`agents/*/.agent-config/plugins/`) — has machine-specific
  absolute paths in metadata; just re-run `/agent-setup`.
- The patched `server.ts` — the patch is re-applied with machine-B paths
  by `/agent-setup`.
- `~/.bashrc` aliases — re-added by `/agent-setup`.
- `settings.json` — re-written by `/agent-setup`.

What you must transfer out-of-band (NOT through git):

- The 4 (or N) Discord bot tokens in `agents/*/.agent-config/channels/discord/.env`.
- Your Claude Code account login (`claude` will prompt for OAuth on
  machine B — separate one-time login).

Migration recipe:

```bash
# On machine B, after the one-time bootstrap of Part 1:
mkdir -p ~/Projects && cd ~/Projects
git clone <your-repo-url> agents_team
cd agents_team
claude
# inside:
/agent-setup claude-a
/agent-setup claude-b
# ... (Resume / re-patch when prompted; this rewrites paths to machine B)
/exit
source ~/.bashrc
# bring in tokens (see next section), then for each bot:
bot-a
# inside:
/discord:configure <token>
/exit
bot-a
/mcp        # verify connected
```

## Token transfer between machines

**Never push tokens to GitHub.** GitHub Secret Scanning is partnered
with Discord; any token in a push is detected within minutes and
auto-revoked by Discord. Even private repos can be detected. After
revocation you'd have to Reset Token on each Application and reconfigure
the bots — strictly more work than just transferring the tokens
out-of-band.

The plain `.env` files together total < 1KB. Pick the simplest channel
you have access to on both machines:

| Method | Setup | Pros | Cons |
|--------|-------|------|------|
| Password manager (1Password, Bitwarden) | One-time install on both | Encrypted at rest, syncs automatically | Slowest first-time setup |
| Cloud-synced text file (iCloud, OneDrive, Google Drive) | One-time client install | Already there for most people | Plaintext in your account |
| Messenger to self (WeChat 文件传输助手, iMessage, Signal Note to Self) | None | Zero new setup | Plaintext in chat history |
| USB drive | None | Air-gapped | Physical step |
| Display once on screen, type on other machine | None | Nothing leaves your eyes | Painful for long tokens; typos easy |
| LAN file transfer (`scp`, `python -m http.server`) | Both machines online | One-shot, fast | Brief network exposure |

After tokens land on machine B and are configured into the bots,
**delete the transfer artifact from both machines**.

---

## Troubleshooting

### `/mcp` shows `Failed to reconnect to plugin:discord:discord: -32000`

`-32000` is "Connection closed" — the plugin server's Node.js process
exited at startup. Find out why:

```bash
# Inside the bot's Claude session:
/debug
/mcp                # triggers another reconnect attempt that gets logged
/exit

# In a normal shell:
LATEST=$(ls -t agents/<name>/.agent-config/debug/*.txt | head -1)
grep -B 1 -A 5 'Server stderr\|ENOENT\|DISCORD_BOT_TOKEN\|MCP error' "$LATEST"
```

Common causes:

| stderr message | Fix |
|----------------|-----|
| `DISCORD_BOT_TOKEN required, set in <some path>/.env` | The bot has no token. Run `/discord:configure <token>`, then `/exit` and re-launch. |
| `ENOENT: ... peer-bots.json` | The patched `server.ts` is reading from a path that doesn't exist. Verify `shared/peer-bots.json` is present at `$PROJECT_ROOT/shared/peer-bots.json`, and that `PEER_BOTS_FILE` env var in the alias matches. If you migrated from another OS, re-run `/agent-setup` to re-apply the patch with current-machine paths. |
| `EADDRINUSE` or "token already in use" | Two Claude Code sessions are trying to use the same bot token. Only one session per token. Quit the duplicate. |
| `Invalid token` | Discord revoked your token (often because it was pushed to GitHub). Reset Token in Developer Portal, `/discord:configure` the new one. |

### `--channels blocked by org policy`

You ran `bot-<name>` and saw `--channels blocked by org policy
(plugin:discord@claude-plugins-official). Inbound messages will be
silently dropped`.

Either there's an org-level managed-settings file disabling channels,
or your Claude Code login needs to opt in. Check:

```bash
find / -name 'managed-settings.json' 2>/dev/null
cat ~/.claude/settings.json 2>/dev/null
```

If `managed-settings.json` exists and sets `channelsEnabled: false`,
either edit it (`sudo`) or, if your account has no real org policy,
remove the file. If no managed settings, add to `~/.claude/settings.json`:

```json
{
  "channelsEnabled": true
}
```

then restart the bot session.

### `--dangerously-skip-permissions cannot be used with root/sudo privileges`

You're running Claude Code as root inside WSL2. Claude Code refuses
this combination as a safety guard. Fix: create a non-root WSL user
and run from that user (see [1.B](#1b-windows--wsl2-recommended)). After
migrating files and updating `/etc/wsl.conf`, restart WSL.

### Bot replies in DM fail: "I can't reply until you run /discord:access policy pairing"

The user ID is in `allowFrom` but the `approved/<userID>` mapping is
missing — the bot doesn't know which DM channel maps to that user.
Solution:

```
/discord:access policy pairing
```

Then DM the bot from Discord. It generates a pairing code. Approve:

```
/discord:access pair <code>
/discord:access policy allowlist   # lock back down
```

Group channels don't need this — they work directly from
`groups[<channel-id>]` configuration.

### Bot-to-bot `@`-mentions silently dropped

The upstream discord plugin drops every bot-authored message. The
patched `server.ts` allows bot accounts listed in `peer-bots.json`.
If `@`-mentions between bots aren't working:

1. Verify the patch is in place:
   ```bash
   grep 'LOCAL PATCH' agents/<name>/.agent-config/plugins/cache/claude-plugins-official/discord/0.0.4/server.ts
   ```
   Should match. If not, re-run `/agent-setup <name>`.
2. Verify the sending bot's Application ID is in `peer-bots.json`.
3. Verify `peer-bots.json` path resolves: in the alias, `PEER_BOTS_FILE`
   should point to an existing absolute path.
4. Check rate limiting — if a sender exceeded `rateLimitPerMin`,
   inbound is dropped silently until the window passes.

### Permission prompts on every Discord reply

The bot prompts "Allow `mcp__plugin_discord_discord__reply`?" each
time. `/agent-setup` should write a `settings.json` with these
pre-allowed; if you bypassed the skill or removed the file, restore:

```json
{
  "permissions": {
    "allow": [
      "mcp__plugin_discord_discord__reply",
      "mcp__plugin_discord_discord__react",
      "mcp__plugin_discord_discord__edit_message",
      "mcp__plugin_discord_discord__fetch_messages",
      "mcp__plugin_discord_discord__download_attachment"
    ]
  },
  "enabledPlugins": {
    "discord@claude-plugins-official": true
  }
}
```

at `agents/<name>/.agent-config/settings.json`.

### `/discord:configure` writes to the wrong location

The original discord plugin's `/discord:configure` skill hardcoded
`~/.claude/channels/discord/.env`. The `/agent-setup` patch rewrites
the local skill copy to honor `$DISCORD_STATE_DIR`. If after upgrading
the plugin the patch was lost, re-run `/agent-setup <name>` (it
detects unpatched files and re-applies).

### "Failed to clone repo" — repo private and no auth

If you set the repo private, you need either an SSH key on the new
machine (`ssh-keygen`, add the public key to GitHub) or an HTTPS
personal access token. For ephemeral migration use HTTPS:

```bash
git clone https://<your-username>:<github-pat>@github.com/<owner>/<repo>.git agents_team
```

Don't commit this URL — it embeds the PAT.

---

## Architecture deep-dive

Three layers of state, three env vars:

```
+----------------------------------------------------+
| working directory (agents/<name>/)                 |
|  - CLAUDE.md  ← project-level instructions         |
|                                                    |
|  +-------------------------------------------+     |
|  | CLAUDE_CONFIG_DIR (.agent-config/)        |     |
|  |  - settings.json (permissions, plugins)   |     |
|  |  - plugins/cache/  (Discord plugin code)  |     |
|  |  - memory/  (long-term)                   |     |
|  |  - role.md  (agent role spec)             |     |
|  |                                           |     |
|  |  +-------------------------------------+  |     |
|  |  | DISCORD_STATE_DIR                   |  |     |
|  |  | (channels/discord/)                 |  |     |
|  |  |  - .env  (BOT TOKEN — gitignored)   |  |     |
|  |  |  - access.json  (allowlist, groups) |  |     |
|  |  |  - approved/<userID>  (DM mapping)  |  |     |
|  |  +-------------------------------------+  |     |
|  +-------------------------------------------+     |
+----------------------------------------------------+

  External:
  - PEER_BOTS_FILE → shared/peer-bots.json
    (cross-bot allowlist for bot-to-bot @-mentions)
```

The launch alias exports all three env vars before invoking `claude`:

```bash
alias bot-x='cd <agent-dir> && \
  CLAUDE_CONFIG_DIR="$PWD/.agent-config" \
  DISCORD_STATE_DIR="$PWD/.agent-config/channels/discord" \
  PEER_BOTS_FILE="<project>/shared/peer-bots.json" \
  claude --dangerously-skip-permissions \
         --channels plugin:discord@claude-plugins-official'
```

Why each layer is separate:

- **working dir**: scopes `CLAUDE.md` (Claude Code auto-loads
  `CLAUDE.md` from the working dir and its parents).
- **CLAUDE_CONFIG_DIR**: isolates each agent's plugin cache, settings,
  memory. Without this all bots would share `~/.claude/`.
- **DISCORD_STATE_DIR**: isolates Discord identity (token, allowlist).
  The plugin's `server.ts` reads this env var:
  ```ts
  const STATE_DIR = process.env.DISCORD_STATE_DIR ??
      join(homedir(), '.claude', 'channels', 'discord')
  ```
- **PEER_BOTS_FILE**: cross-platform path to the shared bot allowlist.
  Defaults to a project-relative path in `server.ts`, overridable so
  the same patched code works on macOS, Linux, and Windows.

### The plugin patches

`/agent-setup` applies two patches to each agent's local plugin copy:

**Patch 1 — Path resolution in SKILL.md (access + configure)**

Original:
```
Manages access control for the Discord channel. All state lives in
`~/.claude/channels/discord/access.json`.
```

Patched:
```
## Path resolution (READ THIS FIRST)

State lives in `$DISCORD_STATE_DIR/` if that env var is set, otherwise
in `~/.claude/channels/discord/`. Before any read/write, capture:

```bash
STATE_DIR="${DISCORD_STATE_DIR:-$HOME/.claude/channels/discord}"
```
```

This makes Claude (which executes the skill instructions) write to the
agent-isolated directory, not the global default.

**Patch 2 — Bot-to-bot inbound allowlist in server.ts**

Original `messageCreate` handler:
```ts
client.on('messageCreate', msg => {
  if (msg.author.bot) return    // ← drops ALL bot messages
  handleInbound(msg).catch(...)
})
```

Patched: load `peer-bots.json` on each inbound, allow listed bot
senders with per-sender rate-limit, drop self and unknown bots:

```ts
const PEER_BOTS_FILE = process.env.PEER_BOTS_FILE ?? '...'
const recentBotInbound: Map<string, number[]> = new Map()
function loadPeerBots() { /* re-reads file every call */ }

client.on('messageCreate', msg => {
  if (msg.author.bot) {
    if (msg.author.id === client.user?.id) return  // never self-loop
    const peers = loadPeerBots()
    if (!peers.ids.has(msg.author.id)) return
    // rate limit per sender
    const now = Date.now()
    const recent = (recentBotInbound.get(msg.author.id) ?? [])
        .filter(t => now - t < 60_000)
    if (recent.length >= peers.rateLimitPerMin) return
    recent.push(now)
    recentBotInbound.set(msg.author.id, recent)
  }
  handleInbound(msg).catch(...)
})
```

Rate limiting is a hard cap to prevent runaway loops (e.g. two bots
that keep `@`-mentioning each other in a tight cycle). Tune
`rateLimitPerMin` in `peer-bots.json`; default 30/min.

---

## Security model

What's NEVER in git (enforced by `.gitignore`):

- `agents/*/.agent-config/channels/discord/.env` — Discord bot tokens.
  Discord considers these credentials; GitHub Secret Scanning will
  detect and notify Discord, who will revoke them.
- `agents/*/.agent-config/.claude.json` and `.claude.json.backup.*` —
  Claude Code's per-machine OAuth state.
- `agents/*/.agent-config/mcp-needs-auth-cache.json` — OAuth for other
  MCP servers (Gmail, Drive, etc. if connected).
- `agents/*/.agent-config/projects/*/*.jsonl` — session histories.
  These can contain anything you pasted into a session, including
  secrets.

What IS committed (Discord public snowflakes — not credentials):

- User IDs in `allowFrom`
- Channel IDs in `groups`
- Bot Application IDs in `peer-bots.json`
- DM channel IDs in `approved/<userID>`

These are visible to anyone interacting with the bot in any case; they
function like usernames, not passwords.

What you should review before publishing:

- `agents/<name>/CLAUDE.md` — may reference specific operator names or
  research topics. Redact or generalize if making the repo public.
- Memory files in `agents/<name>/.agent-config/projects/*/memory/` —
  may contain free-form notes about ongoing work.

If you suspect a token leaked (pushed to GitHub, posted in chat, etc.),
treat the bot as compromised:

1. Developer Portal → Bot → **Reset Token**.
2. `/discord:configure <new-token>` on every machine running that bot.
3. Restart each affected session.

---

## Reference: file layout

```
agents_team/
├── README.md
├── SETUP.md                          ← you are here
├── .gitignore                        ← excludes tokens, sessions, plugin caches
├── .claude/
│   ├── settings.json                 ← project-level: enabledPlugins
│   └── skills/
│       ├── agent-setup/SKILL.md           ← macOS/Linux/WSL slash command
│       └── agent-setup-windows/SKILL.md   ← native Windows slash command
├── shared/
│   └── peer-bots.json                ← bot-to-bot allowlist + rate limit
└── agents/
    └── <name>/
        ├── CLAUDE.md                 ← role pointer + conventions
        ├── workspace/                ← scratch dir for bot's working files
        └── .agent-config/
            ├── settings.json         ← per-bot: permissions, enabledPlugins
            ├── role.md (optional)    ← full role spec
            ├── channels/discord/
            │   ├── .env              ← BOT TOKEN — gitignored
            │   ├── access.json       ← allowlist, groups, dmPolicy
            │   └── approved/         ← <userID> → DM channel ID
            ├── plugins/cache/        ← copied + patched discord plugin
            ├── plugins/installed_plugins.json
            ├── plugins/known_marketplaces.json
            └── projects/*/memory/    ← long-term notes per agent
```

---

## Adapting this for your own use

This repo deploys a 4-agent research mesh (Orchestrator + Synthesizer +
Critic + Scout). You can:

- **Run fewer bots**: just `/agent-setup` only the ones you need.
- **Run different roles**: edit each agent's `CLAUDE.md` and
  `role.md` to define your own discipline.
- **Run more bots on more machines**: clone the repo on each, do the
  bootstrap once, then `/agent-setup` only the agents that machine
  should run. Share `peer-bots.json` via git so they recognize each
  other.
- **Run a single solo bot**: skip Parts 6 and 7; one CLAUDE.md and
  one `/agent-setup` invocation is enough.

Issues and improvements welcome — open a GitHub Issue or PR.
