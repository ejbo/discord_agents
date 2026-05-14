[**English**](README.md) · [简体中文](README.zh-CN.md)

# discord_agents

A multi-provider AI agent mesh, orchestrated on Discord. Every bot is
a **native provider agent** — not a thin LLM-API chat wrapper — so each
model contributes its full agentic strength (tool use, file system,
code execution, plugins, MCPs). Discord is the coordination plane:
specialists `@`-mention each other to hand off work, and you send /
receive tasks from any device.

```
                  ┌────────────────────────────────────────┐
                  │      Discord server / channel          │
                  └──┬──────────┬──────────┬───────────────┘
                     │          │          │
              ┌──────┴──┐  ┌────┴────┐  ┌──┴───────┐
              │  Bot A  │  │  Bot B  │  │  Bot C   │   ...
              │ Claude  │  │ Claude  │  │  Codex   │
              │  Code   │  │  Code   │  │   CLI    │
              │ (Orch.) │◀▶│(Synth.) │◀▶│ (Critic) │
              └─────────┘  └─────────┘  └──────────┘
                isolated     isolated     isolated
                config /     config /     config /
                memory       memory       memory
```

## Why this exists

A typical "LLM bot" wraps a provider's **completion endpoint** — text in,
text out, no tool use, no file system, no skills. This project wraps
each provider's **agent product**:

- **Claude bots → Claude Code**: full tool catalog (Bash, Read/Write,
  Edit, WebFetch, WebSearch), skills system, plugin ecosystem, MCP
  servers (Gmail, Drive, computer-use, …), `--dangerously-skip-permissions`
  for autonomous operation. The bot isn't just generating text — it's
  reading repo files, running tests, hitting external APIs, editing code.
- **Codex bots → OpenAI Codex CLI**: the same idea on the OpenAI side —
  Codex's native agent runtime, with its own tool use, file system access,
  and repository awareness.
- **(Roadmap) Gemini / Mistral / others**: pluggable as each provider's
  native agent runtime matures.

Each agent is a **specialist with its own native strengths**, not a
lowest-common-denominator chat interface. The mesh routes work at the
agent layer.

## Why per-bot isolation

Every bot is fully sandboxed from every other bot, even when they run
on the same machine and even when they share a Claude account:

- **Separate plugin cache.** Each bot has its own copy of installed
  plugins, with its own version. You can patch the Discord plugin
  differently for different bots, or run different versions side by
  side.
- **Separate memory.** Each bot maintains its own long-term memory
  under `.agent-config/projects/`. The Orchestrator never sees the
  Synthesizer's notes; the Critic doesn't inherit the Scout's research
  drafts. Personas don't bleed.
- **Separate Discord identity.** Each bot is a different Discord
  Application with its own token, its own allowlist, its own approved
  DM channels. One bot getting paired with a stranger doesn't affect
  the others.
- **Separate settings.** Permission allowlist, enabled plugins, theme,
  hooks — all per-bot. Lock one bot down tightly and leave another
  freer, in the same project.
- **Cross-provider coexistence.** A Claude Code bot and a Codex CLI
  bot can live as siblings under `agents/`; they don't share config
  files because each runs under its own `CLAUDE_CONFIG_DIR` (or
  `CODEX_HOME`, etc.).
- **Failures don't cascade.** If one bot's plugin server crashes, its
  Claude session exits, or its token gets revoked, the others keep
  running as normal.

Mechanically, isolation comes from three env vars exported by each
bot's launch alias:

```bash
CLAUDE_CONFIG_DIR=<agent-dir>/.agent-config
DISCORD_STATE_DIR=<agent-dir>/.agent-config/channels/discord
PEER_BOTS_FILE=<repo-root>/shared/peer-bots.json
```

That's the whole trick — point Claude Code (or any other agent
runtime) at a per-bot directory and it treats that as its full
universe. The `/agent-setup-mac` / `/agent-setup-wsl` skill wires
this up automatically when you add a new bot.

## Why Discord

- **One channel, all agents present.** `@`-mentions route work; the
  channel history is shared context; you can scroll up to see the full
  thread.
- **Multi-device sync for free.** Discord already runs on mobile / web /
  desktop / watch. Send a task from your phone in line at a coffee shop;
  read the result on your laptop at home — no custom dashboard required.
- **Asynchronous.** Agents wake on `@`-mention, do their work, hand off.
  No always-on UI needed.
- **Permissioned out of the box.** Per-bot allowlists, group-channel
  gating, DM pairing. Random people can't trigger your agents.
- **Bot-to-bot reachable.** A local patch to the Discord plugin lets
  agents `@` each other, gated by `shared/peer-bots.json` with
  per-sender rate limiting (kills runaway loops).

## How a task flows (4-agent example)

1. You DM or `@` the **Orchestrator** with a research question.
2. Orchestrator dispatches the **Scout** to gather evidence. Scout's
   native agent runs forward / counter / regional searches in parallel
   and returns an info packet.
3. Orchestrator routes to the **Synthesizer**. Synth produces a
   falsifiable thesis with explicit P (probability) and E (effect
   magnitude) estimates, plus an L0-L5 evidence tree.
4. Orchestrator routes to the **Critic**. Critic red-teams across five
   attack layers; flags fatal flaws.
5. Synth revises; Critic re-audits. Iterate until convergence or a
   3-round threshold triggers operator escalation.
6. Orchestrator delivers a three-part summary (technical / red-team /
   plain-language) — to your phone, in time for your next coffee break.

Each step is a **native agent** doing what it's best at, not a single
context-window juggling everything.

## Highlights

- **One slash command to add a bot**: `/agent-setup-mac <name>` (or
  `/agent-setup-wsl <name>` on WSL2 / Linux) scaffolds the agent
  directory, copies and patches the Discord plugin, writes
  permissions, and registers a launch alias.
- **Per-bot isolation.** Each agent's plugin cache, settings, memory,
  and Discord state are scoped to its own directory via
  `CLAUDE_CONFIG_DIR` and `DISCORD_STATE_DIR` env vars. Bots on the
  same machine never collide.
- **Bot-to-bot `@`-mention support.** The upstream Discord plugin drops
  all bot-authored messages; this repo's local patch allows peer bots
  listed in `shared/peer-bots.json` through, with per-sender rate
  limiting.
- **Cross-platform.** Ships with `/agent-setup-mac` for macOS and
  `/agent-setup-wsl` for WSL2 / Linux.
- **Migration-friendly.** Roles, memory, and allowlists are git-tracked;
  the only thing that needs an out-of-band transfer is the Discord bot
  token (never push that to git — Discord auto-revokes leaked tokens
  via GitHub Secret Scanning).

## Quickstart

After installing Claude Code and the Discord plugin (see
[SETUP.md § Part 1](SETUP.md#part-1--one-time-bootstrap-per-machine)):

```bash
cd ~/Projects && git clone <this-repo> agents_team && cd agents_team
claude
# in the Claude session:
/agent-setup-mac claude-a    # or /agent-setup-wsl on Linux
/exit

source ~/.bashrc                       # or ~/.zshrc on macOS
bot-a                                  # launches the new bot
# in the bot's Claude session:
/discord:configure <YOUR_BOT_TOKEN>
/exit
bot-a
/mcp                                   # plugin:discord:discord should be connected
```

Then DM the bot, run `/discord:access pair <code>` to allowlist
yourself, and you're live.

Full walkthrough — creating the Discord bot Application, inviting it
to a server, configuring roles, troubleshooting, migrating across
machines — in **[SETUP.md](SETUP.md)**.

## Repository layout

```
agents_team/
├── README.md                ← this file
├── README.zh-CN.md          ← 中文版
├── SETUP.md                 ← full deployment guide
├── .gitignore
├── .claude/skills/
│   ├── agent-setup-mac/         ← /agent-setup-mac (macOS, writes to ~/.zshrc)
│   └── agent-setup-wsl/         ← /agent-setup-wsl (WSL2 / Linux, writes to ~/.bashrc)
├── shared/
│   └── peer-bots.json       ← cross-bot allowlist (Application IDs)
└── agents/
    └── <name>/
        ├── CLAUDE.md            ← identity + role pointer + conventions
        └── .agent-config/
            ├── role.md          ← full role spec for this agent
            └── channels/discord/
                └── access.json  ← allowlist / groups / dmPolicy
```

See [SETUP.md § Reference: file layout](SETUP.md#reference-file-layout)
for the per-agent structure in full.

## Roles in this repo (as a starting point)

This repo is configured for a 4-agent research mesh. Each agent's
`CLAUDE.md` + `.agent-config/role.md` define the discipline:

| Agent | Role | Native runtime | Primary responsibility |
|-------|------|---------------|------------------------|
| `claude-a` | Orchestrator | Claude Code | Routes work, enforces handoff discipline, escalates to human at threshold |
| `claude-b` | Synthesizer | Claude Code | Produces opinionated, falsifiable judgments (L0/L1-L5/Steelman structure) |
| `claude-c` | Critic | Claude Code | Red-teams Synthesizer output across 5 attack layers |
| `claude-d` | Scout | Claude Code | Stateless information courier with mandatory counter-evidence self-check |

For your own use, edit each agent's `CLAUDE.md` and `role.md` to
redefine roles — nothing in the infrastructure assumes specific names.
Swap a Claude bot for a Codex bot by changing the launch command.

## Security notes

- Discord bot tokens (`.env` files) are never committed. Transfer them
  between machines via password manager / encrypted note / USB. Pushing
  a token to GitHub triggers automatic revocation by Discord (via the
  GitHub Secret Scanning partnership).
- Public Discord snowflakes (user IDs, channel IDs, bot Application
  IDs) are not credentials — committing them is fine.
- See [SETUP.md § Security model](SETUP.md#security-model) for the full
  threat model.

## License

MIT.

## Acknowledgements

Built on top of [Anthropic Claude Code](https://claude.ai/code) and the
official [discord plugin](https://github.com/anthropics/claude-plugins-official).
The bot-to-bot inbound patch in `server.ts` and the multi-bot isolation
scaffolding are local to this repo.
