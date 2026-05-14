[English](README.md) · [**简体中文**](README.zh-CN.md)

# discord_agents

一个**多 provider 原生 agent**协作网络，通过 Discord 协调。每个 bot 接入的
都是各家 provider 的**原生 agent 产品**——不是套 LLM API 的简单聊天封装——
这样每个模型都能发挥它的完整 agentic 能力（工具调用、文件系统、代码执行、
plugin、MCP）。Discord 是协调平面：各 specialist 通过 `@` 互相派单，你在
任何设备上发任务、收结果。

```
                  ┌────────────────────────────────────────┐
                  │      Discord 服务器 / 频道             │
                  └──┬──────────┬──────────┬───────────────┘
                     │          │          │
              ┌──────┴──┐  ┌────┴────┐  ┌──┴───────┐
              │  Bot A  │  │  Bot B  │  │  Bot C   │   ...
              │ Claude  │  │ Claude  │  │  Codex   │
              │  Code   │  │  Code   │  │   CLI    │
              │ 调度官  │◀▶│ 综合师  │◀▶│ 红队     │
              └─────────┘  └─────────┘  └──────────┘
                独立        独立         独立
                config /    config /     config /
                memory      memory       memory
```

## 项目目的

普通的 "LLM bot" 包装的是 provider 的 **completion 接口** —— 文字进，文字出，
没有工具调用、没有文件系统、没有 skill。本项目包装的是每个 provider 的
**agent 产品**：

- **Claude bot → Claude Code**：完整工具套件（Bash、Read/Write、Edit、
  WebFetch、WebSearch）、skill 体系、plugin 生态、MCP servers（Gmail、Drive、
  computer-use 等）、`--dangerously-skip-permissions` 自动化模式。Bot 不只是
  在生成文字——它在读项目文件、跑测试、调外部 API、改代码。
- **Codex bot → OpenAI Codex CLI**：OpenAI 那一侧同样思路 —— 用 Codex 自己的
  原生 agent runtime，带它自己的工具调用、文件系统访问、仓库感知能力。
- **（规划中）Gemini / Mistral / 其他**：等每个 provider 的原生 agent runtime
  成熟后即可插入。

每个 agent 都是带自己**原生强项的 specialist**，不是被压平成"最小公约数"的
chat 接口。整个 mesh 在 **agent 层**做路由，不是 API 层。

## 为什么用 Discord 当协调层

- **单 channel 全员在场**：`@` 用来派单，channel 历史是共享上下文，往上翻就能
  看到完整线程。
- **多端同步免费拿**：Discord 本身就跨手机/Web/桌面/手表同步。你在咖啡店排队
  时手机发任务，回家在笔记本上看结果——不用自己搭面板。
- **异步运行**：agent 只在被 `@` 时唤醒，干完活就交棒。不需要常驻 UI。
- **权限开箱即用**：每个 bot 独立 allowlist、群组 channel gating、DM pairing。
  陌生人不能触发你的 agent。
- **bot 互相可达**：本仓库 patch 过 Discord plugin，让 agent 之间可以互相 `@`，
  通过 `shared/peer-bots.json` 控制白名单 + 每发件人 rate limit（防失控循环）。

## 一个任务的完整流程（4-agent 示例）

1. 你 DM 或 `@` **调度官**（Orchestrator），抛一个研究问题。
2. 调度官派 **侦察兵**（Scout）出去搜证据。Scout 的原生 agent 并行跑正向 /
   反向 / 地域多元三类搜索，返回信息包。
3. 调度官交给 **综合师**（Synthesizer）。Synth 产出一个可证伪的判断，带显式
   概率估计（P）和影响幅度估计（E），加 L0-L5 证据树。
4. 调度官交给 **红队**（Critic）。Critic 在 5 个攻击层级展开攻击，标致命漏洞。
5. Synth 修订，Critic 复审。直到收敛或者撞到 3 轮阈值触发用户升级。
6. 调度官给出三件套总结（技术版 / 红队总结 / 白话版）——推到你手机上，正好赶上
   你下次喝咖啡的休息时间。

每一步都是**原生 agent 干自己擅长的事**，不是一个 context window 硬扛全部任务。

## 特点

- **一条 slash 命令加一个 bot**：`/agent-setup-mac <name>`（macOS）或者
  `/agent-setup-wsl <name>`（WSL2 / Linux）会自动建好 agent 目录、复制并 patch
  Discord plugin、写好 permissions、注册启动 alias。
- **bot 级隔离**：每个 agent 的 plugin cache、settings、memory、Discord state
  通过 `CLAUDE_CONFIG_DIR` 和 `DISCORD_STATE_DIR` 环境变量 scope 到自己的目录。
  同机器多个 bot 不会冲突。
- **bot-to-bot `@` 支持**：上游 Discord plugin 默认丢弃所有 bot 发的消息；
  本仓库本地 patch 改成允许 `peer-bots.json` 里列出的 bot 互相通信，带
  per-sender rate limit。
- **跨平台**：提供 `/agent-setup-mac`（macOS）和 `/agent-setup-wsl`
  （WSL2 / Linux）两个 skill。
- **方便迁移**：角色规范、memory、allowlist 都进 git；只有 Discord bot token
  需要走带外通道（不能进 git——GitHub Secret Scanning 跟 Discord 是合作伙伴，
  token 一旦推上去会被自动 revoke）。

## 快速开始

装好 Claude Code 和 Discord plugin 之后（详见
[SETUP.md § Part 1](SETUP.md#part-1--one-time-bootstrap-per-machine)）：

```bash
cd ~/Projects && git clone <repo 地址> agents_team && cd agents_team
claude
# 在 Claude session 里：
/agent-setup-mac claude-a    # 或 /agent-setup-wsl 如果你在 Linux
/exit

source ~/.bashrc                       # macOS 上是 ~/.zshrc
bot-a                                  # 启动新 bot
# 在 bot 的 Claude session 里：
/discord:configure <你的 BOT TOKEN>
/exit
bot-a
/mcp                                   # 应该看到 plugin:discord:discord connected
```

然后在 Discord 上 DM bot，跑 `/discord:access pair <code>` 把自己加进
allowlist，就能用了。

完整流程——建 Discord bot Application、邀请到服务器、配置 role、排错、跨机器
迁移——都在 **[SETUP.md](SETUP.md)** 里。

## 仓库结构

```
agents_team/
├── README.md                ← 英文版
├── README.zh-CN.md          ← 你现在看的
├── SETUP.md                 ← 完整部署指南
├── .gitignore
├── .claude/skills/
│   ├── agent-setup-mac/         ← /agent-setup-mac（macOS，写 ~/.zshrc）
│   └── agent-setup-wsl/         ← /agent-setup-wsl（WSL2 / Linux，写 ~/.bashrc）
├── shared/
│   └── peer-bots.json       ← bot 互相 @ 的白名单（Application ID）
└── agents/
    └── <name>/
        ├── CLAUDE.md            ← 身份 + 角色指针 + 约定
        └── .agent-config/
            ├── role.md          ← 该 agent 的完整角色规范
            └── channels/discord/
                └── access.json  ← allowlist / groups / dmPolicy
```

每个 agent 的目录详细结构见
[SETUP.md § Reference: file layout](SETUP.md#reference-file-layout)。

## 仓库自带的角色配置（起步模板）

仓库预置了一个 4 agent 的研究 mesh。每个 agent 的 `CLAUDE.md` +
`.agent-config/role.md` 定义了它的纪律：

| Agent | 角色 | 原生 runtime | 主要职责 |
|-------|------|-------------|----------|
| `claude-a` | 调度官 Orchestrator | Claude Code | 路由派单、强制接力纪律、必要时升级人类 |
| `claude-b` | 综合师 Synthesizer | Claude Code | 产出带立场、可证伪的判断（L0/L1-L5/反方最强论结构） |
| `claude-c` | 红队 Critic | Claude Code | 在 5 个攻击层级上 red-team Synthesizer 产出 |
| `claude-d` | 侦察兵 Scout | Claude Code | 无状态信息搬运工，强制对照搜索 + 反向证据自检 |

要换成你自己的用法，编辑每个 agent 的 `CLAUDE.md` 和 `role.md` 即可——
infrastructure 不依赖具体的角色名。把 Claude bot 换成 Codex bot 只需改启动命令。

## 安全须知

- Discord bot token (`.env` 文件) 永远不进 git。换机器时用密码管理器 / 加密
  笔记 / U 盘等带外通道传。**Token 一旦推到 GitHub，Discord 会通过 GitHub
  Secret Scanning 合作渠道自动 revoke**。
- Discord 公开 snowflake（user ID、channel ID、bot Application ID）不是
  credential，进 git 没问题。
- 完整威胁模型见
  [SETUP.md § Security model](SETUP.md#security-model)。

## License

MIT.

## 致谢

基于 [Anthropic Claude Code](https://claude.ai/code) 和官方
[discord plugin](https://github.com/anthropics/claude-plugins-official) 构建。
`server.ts` 的 bot-to-bot inbound patch 以及多 bot 隔离脚手架是本仓库特有。
