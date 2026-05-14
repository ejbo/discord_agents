# discord_agents

Concurrent, isolated Claude Code agents that coordinate over Discord. Each
bot has its own Discord identity, its own config, its own memory, and its
own role spec. They talk to humans in shared channels and to each other
via `@`-mentions (gated by a per-sender rate-limited allowlist).

```
                 ┌────────────────────────────┐
                 │   Discord server / channel │
                 └──────┬────────────┬────────┘
                        │            │
              ┌─────────┴───┐   ┌────┴─────────┐
              │  Bot A      │   │   Bot B      │   ...
              │ Orchestrator│   │ Synthesizer  │
              │             │◀─▶│              │
              │ Claude Code │   │  Claude Code │
              │  session    │   │   session    │
              └─────────────┘   └──────────────┘
                  isolated         isolated
                  config/memory    config/memory
```

## Highlights

- **One slash command to add a bot**: `/agent-setup <name>` scaffolds the
  agent directory, copies and patches the Discord plugin, writes
  permissions, and registers a launch alias.
- **Bot-to-bot `@`-mention support**: the upstream Discord plugin drops
  all bot-authored messages; this repo's local patch allows peers listed
  in `shared/peer-bots.json` through, with per-sender rate limiting.
- **Per-bot isolation**: each agent's plugin cache, settings, memory, and
  Discord state are scoped to its own directory via `CLAUDE_CONFIG_DIR`
  and `DISCORD_STATE_DIR` env vars. Two bots on the same machine never
  collide.
- **Cross-platform**: ships with `/agent-setup` (macOS/Linux/WSL bash)
  and `/agent-setup-windows` (native PowerShell).
- **Migration-friendly**: roles, memory, and allowlists are git-tracked;
  the only things that need an out-of-band transfer are Discord bot
  tokens (which must never go through git — Discord auto-revokes leaked
  tokens via GitHub Secret Scanning).

## Quickstart

After installing Claude Code and the Discord plugin (see [SETUP.md
§ Part 1](SETUP.md#part-1--one-time-bootstrap-per-machine)):

```bash
cd ~/Projects && git clone <this-repo> agents_team && cd agents_team
claude
# In the Claude session:
/agent-setup claude-a
/exit

source ~/.bashrc                     # or ~/.zshrc
bot-a                                # launches the new bot
# In the bot's Claude session:
/discord:configure <YOUR_BOT_TOKEN>
/exit
bot-a
/mcp                                 # should show plugin:discord:discord connected
```

Then DM the bot, run `/discord:access pair <code>` to allowlist
yourself, and you're live.

Full walkthrough — including creating the Discord bot Application,
inviting it to a server, configuring roles, troubleshooting common
errors, and migrating across machines — is in **[SETUP.md](SETUP.md)**.

## Repository layout

```
agents_team/
├── README.md              ← you are here
├── SETUP.md               ← full deployment guide (read this for setup)
├── .gitignore
├── .claude/skills/
│   ├── agent-setup/             ← /agent-setup for macOS/Linux/WSL
│   └── agent-setup-windows/     ← /agent-setup-windows for native Windows
├── shared/
│   └── peer-bots.json     ← cross-bot allowlist
└── agents/
    └── <name>/            ← one directory per bot
        ├── CLAUDE.md      ← role pointer + conventions
        └── .agent-config/ ← isolated config (token gitignored; allowlist tracked)
```

See [SETUP.md § Reference: file layout](SETUP.md#reference-file-layout)
for the per-agent directory structure in full.

## Roles in this repo

This repo is configured for a 4-agent research mesh. Each agent's
`CLAUDE.md` and `role.md` define the discipline:

| Agent | Role | Primary responsibility |
|-------|------|------------------------|
| `claude-a` | Scout + Orchestrator | Routes work, enforces handoff discipline, escalates to human at threshold |
| `claude-b` | Synthesizer | Produces opinionated, falsifiable judgments (L0/L1-L5/Steelman structure) |
| `claude-c` | Critic | Red-teams Synthesizer output |
| `claude-d` | Reserved | Scout placeholder; reassign as needed |

For your own use, edit each agent's `CLAUDE.md` to redefine roles —
nothing in the infrastructure assumes specific role names.

## Security notes

- Discord bot tokens (`.env` files) are never committed; transfer them
  between machines via password manager, encrypted note, or USB.
- Memory files, allowlists, role specs, and channel IDs are committed —
  these are Discord public snowflakes, not credentials.
- See [SETUP.md § Security model](SETUP.md#security-model) for the full
  threat model.

## License

MIT.

## Acknowledgements

Built on top of [Anthropic Claude Code](https://claude.ai/code) and the
official [discord plugin](https://github.com/anthropics/claude-plugins-official).
The bot-to-bot inbound patch in `server.ts` is local to this repo.
