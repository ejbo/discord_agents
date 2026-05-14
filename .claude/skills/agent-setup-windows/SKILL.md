---
name: agent-setup-windows
description: PowerShell-native variant of /agent-setup for Windows machines. Creates a new isolated Claude bot under agents/<name>/ with patched Discord plugin, settings.json, peer-bots.json wiring, plugin metadata, and a PowerShell launcher function added to $PROFILE. Use on native Windows; for macOS/Linux use /agent-setup; for WSL2 use /agent-setup (paths are Linux-side).
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - AskUserQuestion
---

# /agent-setup-windows — Create a new isolated Claude bot (Windows)

Usage: `/agent-setup-windows <name> [<alias>]`

PowerShell mirror of `/agent-setup`. Same semantics, Windows paths and shell.

- `<name>` — agent directory name, `^[A-Za-z0-9_-]+$` (e.g. `claude-b`).
- `<alias>` — optional PowerShell function name (`bot-b` style). If omitted,
  derived: strip leading `claude-`, otherwise full name, prefixed `bot-`.

Example: `/agent-setup-windows claude-c` → `agents\claude-c\` + `bot-c` function.

---

## Prerequisites on the Windows machine (one-time bootstrap)

Run these once before the first `/agent-setup-windows` invocation:

1. **Install Claude Code** — `iwr -useb https://claude.ai/install.ps1 | iex`
   (or follow the official Windows installer).
2. **Install Node.js 18+** — required by the Discord plugin's server. Get
   from <https://nodejs.org/> (LTS) or `winget install OpenJS.NodeJS.LTS`.
3. **Create the project tree:**
   ```powershell
   New-Item -ItemType Directory -Force "$env:USERPROFILE\Projects\agents_team\shared" | Out-Null
   New-Item -ItemType Directory -Force "$env:USERPROFILE\Projects\agents_team\agents" | Out-Null
   ```
   Optionally `git clone` the existing macOS repo to bring over `CLAUDE.md`,
   `shared/peer-bots.json`, the `.claude/skills/` folder, and any role specs.
4. **Install the Discord plugin globally** (so its cache exists at
   `%USERPROFILE%\.claude\plugins\cache\claude-plugins-official\discord\<ver>`):
   ```powershell
   claude
   # then inside the session:
   /plugin marketplace add anthropics/claude-plugins-official
   /plugin install discord@claude-plugins-official
   /exit
   ```
5. **Seed `peer-bots.json`** — if you didn't clone from the Mac repo:
   ```powershell
   Set-Content -Encoding UTF8 "$env:USERPROFILE\Projects\agents_team\shared\peer-bots.json" '{
     "_comment": "Discord bot IDs allowed to message each other via the patched discord plugin. Update when adding new agents.",
     "bots": [],
     "rateLimitPerMin": 30
   }'
   ```

After bootstrap, `/agent-setup-windows <name>` is the only thing you run
to add bots.

---

## What you (Claude) do, step by step

All shell commands below are PowerShell — invoke them via the Bash tool as
`powershell -NoProfile -Command "..."`. This works regardless of whether the
user's default Bash tool shell is pwsh, cmd, or Git Bash.

### Step 0 — Parse arguments and set vars

Parse `$ARGUMENTS` as two space-separated tokens (NAME, optional ALIAS).
Reject NAME if it doesn't match `^[A-Za-z0-9_-]+$`.

```powershell
$NAME = "<first arg>"
$SHORT = $NAME -replace '^claude-',''
$ALIAS = if ($args[1]) { $args[1] } else { "bot-$SHORT" }
$ROOT = "$env:USERPROFILE\Projects\agents_team"
$AGENT_DIR = "$ROOT\agents\$NAME"
$AGENT_CONF = "$AGENT_DIR\.agent-config"
$PLUGIN_CACHE = "$AGENT_CONF\plugins\cache\claude-plugins-official"
$PLUGIN_MP = "$AGENT_CONF\plugins\marketplaces\claude-plugins-official"
$DISCORD_PLUGIN = "$PLUGIN_CACHE\discord\0.0.4"
$STATE_DIR = "$AGENT_CONF\channels\discord"
$PEER_BOTS = "$ROOT\shared\peer-bots.json"
$SOURCE_PLUGIN = "$env:USERPROFILE\.claude\plugins\cache\claude-plugins-official\discord\0.0.4"
$SOURCE_MP = "$env:USERPROFILE\.claude\plugins\marketplaces\claude-plugins-official"
```

### Step 1 — Preconditions

```powershell
if (-not (Test-Path $ROOT)) { throw "agents_team repo not found at $ROOT — run bootstrap first" }
if (-not (Test-Path $SOURCE_PLUGIN)) { throw "Discord plugin not installed globally. Bootstrap step 4." }
```

If `$AGENT_DIR` already exists, use AskUserQuestion: *Resume/re-patch* (keep
existing files, re-run patching + metadata) | *Abort*. Don't auto-overwrite.

### Step 2 — Make the directory tree

```powershell
New-Item -ItemType Directory -Force "$STATE_DIR\approved" | Out-Null
New-Item -ItemType Directory -Force "$PLUGIN_CACHE" | Out-Null
New-Item -ItemType Directory -Force "$AGENT_CONF\plugins\marketplaces" | Out-Null
New-Item -ItemType Directory -Force "$AGENT_DIR\workspace" | Out-Null
# Windows ACLs aren't required for personal use, but for hardening:
icacls "$STATE_DIR" /inheritance:r /grant:r "${env:USERNAME}:(F)" | Out-Null
```

### Step 3 — Seed the Discord plugin (copy, don't link)

```powershell
if (-not (Test-Path $DISCORD_PLUGIN)) {
  Copy-Item -Recurse -Force "$SOURCE_PLUGIN" "$PLUGIN_CACHE\discord"
}
if (-not (Test-Path $PLUGIN_MP)) {
  Copy-Item -Recurse -Force "$SOURCE_MP" "$PLUGIN_MP"
}
```

### Step 4 — Patch the local discord skills to honor $DISCORD_STATE_DIR

Same content rewrite as the macOS skill. Use Edit (with `replace_all: true`)
on these two files — the patch is content-based, not OS-specific:

- `$DISCORD_PLUGIN\skills\access\SKILL.md`
- `$DISCORD_PLUGIN\skills\configure\SKILL.md`

**Skip if already patched** — grep for `DISCORD_STATE_DIR:-` in the file
first; if present, skip.

For each file:
1. Insert the "Path resolution (READ THIS FIRST)" preamble right after the
   first prose block (before `Arguments passed: $ARGUMENTS`). Use this exact
   block verbatim (keep literal `$` and backticks):

   ```
   ## Path resolution (READ THIS FIRST)

   State lives in `$DISCORD_STATE_DIR/` if that env var is set, otherwise
   in `~/.claude/channels/discord/`. Before any read/write, capture:

   ```bash
   STATE_DIR="${DISCORD_STATE_DIR:-$HOME/.claude/channels/discord}"
   ```

   Every `$STATE_DIR/...` reference below means that resolved absolute path.
   ```

2. Replace literals:
   - `~/.claude/channels/discord/` → `$STATE_DIR/`
   - `~/.claude/channels/discord` → `$STATE_DIR`

### Step 4b — Apply the bot-to-bot inbound patch to `server.ts`

The upstream plugin drops every bot-authored message. Patch `server.ts` so
peer bots listed in `peer-bots.json` get through, with a per-sender rate cap.

**Skip if already patched** — grep for `LOCAL PATCH .* bot-to-bot` in
`$DISCORD_PLUGIN\server.ts`; if present, skip.

For Windows the only difference from the macOS skill's Step 4b is the path
written into `PEER_BOTS_FILE`. Use forward slashes (Node.js accepts them
on Windows) and the resolved Windows path:

```ts
// === LOCAL PATCH (YYYY-MM-DD): bot-to-bot inbound allowlist ===
const PEER_BOTS_FILE = process.env.PEER_BOTS_FILE ?? 'C:/Users/<USERNAME>/Projects/agents_team/shared/peer-bots.json'
```

Replace `<USERNAME>` with `$env:USERNAME` resolved at patch time. The
`process.env.PEER_BOTS_FILE ?? ...` form is **important** — it lets a launch
function override the path without re-patching, which matters when you
clone this repo across machines with different user names. Keep the rest of
the patch (rate limiter, peer-bot lookup, allowlist gate) identical to the
macOS version — copy it from `agents/claude-a/.agent-config/plugins/cache/claude-plugins-official/discord/0.0.4/server.ts` if you have access to the Mac
checkout; otherwise mirror the structure from the macOS `/agent-setup`
SKILL.md Step 4b body.

### Step 5a — Write settings.json (plugin enabled + tools pre-allowed)

```powershell
$settings = @'
{
  "theme": "auto",
  "enabledPlugins": {
    "discord@claude-plugins-official": true
  },
  "permissions": {
    "allow": [
      "mcp__plugin_discord_discord__reply",
      "mcp__plugin_discord_discord__react",
      "mcp__plugin_discord_discord__edit_message",
      "mcp__plugin_discord_discord__fetch_messages",
      "mcp__plugin_discord_discord__download_attachment"
    ]
  }
}
'@
if (-not (Test-Path "$AGENT_CONF\settings.json")) {
  Set-Content -Encoding UTF8 -Path "$AGENT_CONF\settings.json" -Value $settings
} else {
  # Merge: keep existing keys, add missing entries to enabledPlugins +
  # permissions.allow (dedupe). Use ConvertFrom-Json / ConvertTo-Json.
  # See "Merge logic" at bottom for details.
}
```

### Step 5b — Write plugin metadata (Windows-shaped paths)

`$AGENT_CONF\plugins\installed_plugins.json`:
```json
{
  "version": 2,
  "plugins": {
    "discord@claude-plugins-official": [
      {
        "scope": "local",
        "installPath": "<$DISCORD_PLUGIN with forward slashes>",
        "version": "0.0.4",
        "installedAt": "<ISO now>",
        "lastUpdated": "<ISO now>",
        "projectPath": "<$AGENT_DIR with forward slashes>"
      }
    ]
  }
}
```

`$AGENT_CONF\plugins\known_marketplaces.json`:
```json
{
  "claude-plugins-official": {
    "source": { "source": "github", "repo": "anthropics/claude-plugins-official" },
    "installLocation": "<$PLUGIN_MP with forward slashes>",
    "lastUpdated": "<ISO now>"
  }
}
```

Get the ISO timestamp with:
```powershell
$now = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ss.000Z")
```

Convert backslashes to forward slashes:
```powershell
$installPath = $DISCORD_PLUGIN -replace '\\','/'
$projectPath = $AGENT_DIR -replace '\\','/'
$mpPath = $PLUGIN_MP -replace '\\','/'
```

### Step 6 — Write CLAUDE.md template (skip if exists)

If `$AGENT_DIR\CLAUDE.md` already exists, skip. Otherwise write a template
analogous to the macOS skill's Step 6, parameterized with the agent name.

### Step 7 — Add the launcher to PowerShell $PROFILE

PowerShell aliases can't carry env-var setup; use a function instead.

```powershell
# Ensure $PROFILE exists
if (-not (Test-Path $PROFILE)) {
  New-Item -ItemType File -Path $PROFILE -Force | Out-Null
}

$launcherName = $ALIAS
$launcherBody = @"
function $launcherName {
    Set-Location -Path '$AGENT_DIR'
    `$env:DISCORD_STATE_DIR = "`$PWD\.agent-config\channels\discord"
    `$env:CLAUDE_CONFIG_DIR = "`$PWD\.agent-config"
    `$env:PEER_BOTS_FILE = '$PEER_BOTS'
    claude --channels plugin:discord@claude-plugins-official
}
"@

# Append only if not already present
$existing = Get-Content -Raw $PROFILE -ErrorAction SilentlyContinue
if (-not ($existing -match "function $launcherName ")) {
  Add-Content -Path $PROFILE -Value "`n$launcherBody"
}
```

Note: the embedded `` ` `` backticks before `$env:`, `$PWD`, `$launcherName`
in the here-string are PowerShell's literal-`$` escapes — they prevent
expansion at template time so the variables resolve when the function is
*invoked*. `$ALIAS`, `$AGENT_DIR`, `$PEER_BOTS` should expand at template
time (no backtick prefix).

### Step 8 — Final summary

Print:

```
✓ Agent <name> ready at agents\<name>\
✓ Plugin seeded + bot-to-bot patched + skills patched
✓ Metadata written
✓ CLAUDE.md template (edit the Role section)
✓ Function `<alias>` added to $PROFILE

To bring this bot online:
1. Get a new Discord bot token: https://discord.com/developers/applications
   → New Application → Bot → Reset Token → copy
2. Invite to your server (Installation tab → Install Link)
3. Reload PowerShell:  . $PROFILE   (dot-space-$PROFILE; or just open new window)
4. Launch:  <alias>
5. In the Claude session:  /discord:configure <paste-token>
6. DM the bot → it sends a 6-char pairing code
7. In the Claude session:  /discord:access pair <code>
8. Lock it:  /discord:access policy allowlist
9. Add this bot's Discord ID to shared\peer-bots.json so other bots can @ it.
```

---

## Merge logic for existing settings.json (Step 5a)

```powershell
$existing = Get-Content -Raw "$AGENT_CONF\settings.json" | ConvertFrom-Json
if (-not $existing.enabledPlugins) {
  $existing | Add-Member -NotePropertyName enabledPlugins -NotePropertyValue (@{
    'discord@claude-plugins-official' = $true
  })
} else {
  $existing.enabledPlugins.'discord@claude-plugins-official' = $true
}
if (-not $existing.permissions) {
  $existing | Add-Member -NotePropertyName permissions -NotePropertyValue (@{ allow = @() })
}
$needed = @(
  'mcp__plugin_discord_discord__reply',
  'mcp__plugin_discord_discord__react',
  'mcp__plugin_discord_discord__edit_message',
  'mcp__plugin_discord_discord__fetch_messages',
  'mcp__plugin_discord_discord__download_attachment'
)
$existing.permissions.allow = @($existing.permissions.allow + $needed | Select-Object -Unique)
$existing | ConvertTo-Json -Depth 10 | Set-Content -Encoding UTF8 "$AGENT_CONF\settings.json"
```

## Cross-platform peer-bots.json strategy

The Windows bot-to-bot patch uses `process.env.PEER_BOTS_FILE ?? <default>`
so the SAME plugin can run on Mac and Windows. The launch function exports
`$env:PEER_BOTS_FILE` to the Windows-shaped absolute path. The macOS skill's
Step 4b currently hardcodes a Mac path — if you maintain bots on both OSes,
refactor the macOS patch the same way: `process.env.PEER_BOTS_FILE ?? '<mac path>'`,
and have the macOS `bot-*` alias export `PEER_BOTS_FILE` too. Then `server.ts`
becomes truly portable.

## Caveats

- **Long paths**: Windows has a 260-char path limit by default. Plugin cache
  paths can blow past that under deep nesting. Enable long paths via
  `New-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem' -Name 'LongPathsEnabled' -Value 1 -PropertyType DWORD -Force` (requires admin).
- **PowerShell version**: targets PowerShell 7+ (`pwsh`). Windows PowerShell
  5.1 mostly works but some `ConvertTo-Json` defaults differ — pass
  `-Depth 10` everywhere.
- **Execution policy**: if `$PROFILE` script doesn't load, run
  `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`.
- **Antivirus**: real-time AV scanning of `.agent-config\plugins\cache\` can
  noticeably slow plugin launch. Consider adding `%USERPROFILE%\Projects\agents_team`
  to Windows Defender exclusions.

## Alternative: WSL2 (recommended if you can use it)

If you have WSL2 with Ubuntu/Debian, the macOS `/agent-setup` skill runs
identically inside WSL — same bash, same paths (just under WSL's `$HOME`).
Steps:

1. `wsl --install` (one-time, in admin PowerShell)
2. Inside WSL: install Claude Code, clone the repo, run `/plugin install discord@...`
3. Run `/agent-setup claude-b` exactly as on Mac — `bot-b` alias goes to
   WSL's `~/.zshrc` or `~/.bashrc`
4. Launch `bot-b` from inside WSL terminal

Discord network from WSL works fine. The only quirk: Claude Code's IDE
integration with Windows-side editors needs the `\\wsl$\Ubuntu\...` path
or `code .` from inside WSL.

Native PowerShell is more "feels right" for Windows; WSL2 is much less code
to maintain because you reuse the existing skill verbatim.

## Implementation notes

- Be idempotent — re-runnable on existing agents (re-patch, re-write meta).
- Never log or echo Discord tokens. Token writes happen via
  `/discord:configure` inside the new session, not by this skill.
- The `$PROFILE` path differs by PowerShell version and host:
  - PS7 console: `Documents\PowerShell\Microsoft.PowerShell_profile.ps1`
  - PS5.1 console: `Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1`
  - VSCode integrated: `Documents\PowerShell\Microsoft.VSCode_profile.ps1`
  Test with `echo $PROFILE` to see which file. Add the function to the
  console one; for VSCode prompt the user.
- After Step 7, the new function only takes effect in new PS sessions or
  after `. $PROFILE` (dot-source).
