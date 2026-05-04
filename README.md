# better-memory

> A cross-session personal memory system for Claude Code, with intent-based triggering, classification decision tree, and INDEX synchronization.
>
> Claude Code 跨会话个人记忆系统，意图触发、分类决策树、INDEX 自动同步。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Version](https://img.shields.io/badge/version-1.2.0-blue.svg)](https://github.com/FruityMaxine/better-memory/releases)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Plugin-orange.svg)](https://docs.claude.com/claude-code)

---

## 🇬🇧 English

### What it does

`better-memory` gives Claude Code a **structured, cross-project memory system**. When you tell Claude "remember this", "save this to memory", "记下这个", or any natural-language equivalent, the skill:

1. **Classifies** the information through a decision tree (rule? fact? about you? technical?)
2. **Routes** it to the right destination (CLAUDE.md / about_me/ / reference/ / rules/)
3. **Compresses** the content (preserve nuance, drop filler)
4. **Updates the INDEX** so future sessions know what's available

Result: Claude remembers your servers, preferences, accounts, conditional rules — across every project, every session — without bloating your CLAUDE.md or wasting tokens on irrelevant context.

### Why "better"

Built-in CLAUDE.md is a single file that loads into every session. That works for a few rules, but breaks down once you have:
- Multiple servers / accounts / external resources
- Personal preferences that only matter for some tasks
- Conditional rules that only fire in specific contexts

`better-memory` separates **storage** (cheap) from **loading** (expensive):

| Layer | Always loaded? | Cost | What goes here |
|---|---|---|---|
| `CLAUDE.md` | ✅ Every session | High | Always-on behavioral rules |
| `INDEX.md` | ✅ Each conversation start | Low (~500 tokens) | Catalog of available memory |
| Individual `.md` files | ❌ Only when relevant | Zero when unused | Facts, resources, conditional rules |

### Features

- **Intent-based triggering** — no rigid keyword matching; activates on natural language ("记下", "save this", "I want you to know X")
- **3 categorized folders** — `about_me/` (personal), `reference/` (technical), `rules/` (conditional behavior)
- **Auto-synchronized INDEX.md** — listing all memory files with one-line descriptions
- **Coarse-by-default file granularity** — one entity per file; lazy splitting only when needed
- **Conflict-aware updates** — never silently overwrites preferences; asks for choices
- **Compression rules** — strips filler, preserves nuance, keeps proper nouns verbatim
- **Atomic writes** — file + INDEX update in one cycle
- **Cross-platform** — works on Windows, macOS, Linux (path-independent)
- **Reminder hook** — `UserPromptSubmit` hook injects a one-line reminder on every prompt, nudging Claude to re-check `~/.claude/CLAUDE.md` rules before responding (since 1.2.0)

### Installation

In Claude Code, run:

```
/plugin marketplace add FruityMaxine/better-memory
/plugin install better-memory
```

That's it. The skill will auto-initialize `~/.claude/memory/` on first use and prompt you to add a 2-line pointer to your global CLAUDE.md.

### Usage

Just talk to Claude naturally. The skill activates by intent:

| You say | Skill action |
|---|---|
| `记下我的服务器 vega 在 178.x.x.x` | Saves to `reference/server_vega.md` |
| `我喜欢黑色幽默` | Saves to `about_me/preferences.md` |
| `以后做插件都要发到 marketplace` | Saves to `rules/marketplace_publishing.md` |
| `当我说 1+1 你回答 3` | Asks to add to global `CLAUDE.md` |
| `忘掉我对宠物的偏好` | Removes that section, confirms once |
| `你还记得 vega 服务器吗` | Reads `reference/server_vega.md`, summarizes |

### Architecture

```
~/.claude/                                    Global, cross-project
├── CLAUDE.md                                 (1) Behavior rules (always loaded)
└── memory/
    ├── INDEX.md                              Auto-managed catalog
    ├── about_me/                             (2) Personal facts
    │   └── <topic>.md
    ├── reference/                            (3) Technical resources
    │   └── <topic>.md
    └── rules/                                (4) Conditional behavior rules
        └── <topic>.md
```

### Classification decision tree

```
Q1: Will this information change Claude's output?
    ├─ YES → It's a RULE
    │   └─ Always-on?  ✅ → CLAUDE.md (asks first)
    │   └─ Conditional? → memory/rules/<topic>.md
    └─ NO  → It's a FACT
        └─ About user?    → memory/about_me/<topic>.md
        └─ External?      → memory/reference/<topic>.md
```

### Configuration

No configuration required. The skill works out of the box.

For advanced control, you can:
- Manually edit `~/.claude/memory/INDEX.md` to reorganize entries
- Add the memory folder to git (`cd ~/.claude/memory && git init`) for version history
- Move existing user-level skills aside if you previously installed a version manually

### Comparison

| | `CLAUDE.md` only | `better-memory` |
|---|---|---|
| Memory categories | None | 3 (about_me / reference / rules) |
| Cross-project | ✅ | ✅ |
| Conditional loading | ❌ (always loaded) | ✅ (loaded per task) |
| Conflict detection | ❌ | ✅ |
| Compression rules | ❌ | ✅ |
| Auto INDEX | N/A | ✅ |
| Token cost when idle | High | Minimal |

### License

[MIT](LICENSE) © FruityMaxine

### Contributing

Issues and PRs welcome at [github.com/FruityMaxine/better-memory](https://github.com/FruityMaxine/better-memory).

---

## 🇨🇳 中文

### 这是什么

`better-memory` 给 Claude Code 加上**结构化、跨项目的记忆系统**。当你说 "记下这个" / "save this" / "存到 memory" 或任何自然语言表达时，这个 skill 会：

1. **分类**：通过决策树判断信息类型（行为规则？事实？关于你？技术资料？）
2. **归位**：路由到正确的目的地（CLAUDE.md / about_me/ / reference/ / rules/）
3. **压缩**：去口水保核心（保留细微差异，砍掉填充词）
4. **更新 INDEX**：让未来 session 知道有哪些记忆可用

效果：Claude 记得你的服务器、偏好、账户、条件性规则——跨项目跨 session 持续可用——同时**不污染 CLAUDE.md，不浪费 token 在无关上下文上**。

### 为什么叫 "better"

内置 CLAUDE.md 是一个文件，每个 session 全量加载。少量规则 OK，但一旦你有：
- 多台服务器 / 账户 / 外部资源
- 只在某些任务才有意义的个人偏好
- 只在特定场景才触发的条件性规则

CLAUDE.md 就开始膨胀，token 浪费严重。

`better-memory` 把**存储**（便宜）和**加载**（贵）分开：

| 层级 | 总是加载？ | 成本 | 放什么 |
|---|---|---|---|
| `CLAUDE.md` | ✅ 每次 session | 高 | 总是生效的行为规则 |
| `INDEX.md` | ✅ 每次对话开始 | 低（~500 token） | 所有记忆的目录 |
| 单个 `.md` 文件 | ❌ 相关时才读 | 不用时 0 成本 | 事实、资源、条件规则 |

### 特性

- **意图触发** — 不靠硬关键词匹配；自然语言激活（"记下"、"save this"、"以后要记得"）
- **3 个分类文件夹** — `about_me/`（个人）、`reference/`（技术）、`rules/`（条件行为）
- **INDEX.md 自动同步** — 列出所有记忆文件 + 一句描述
- **粗粒度优先** — 同一实体一个文件；只在必要时拆分
- **冲突感知更新** — 偏好替换前必问，不静默覆盖
- **压缩规则** — 去填充词、保留细微差异、专有名词原样保留
- **原子写入** — 文件和 INDEX 同步更新
- **跨平台** — Windows、macOS、Linux 通用（不依赖具体路径）
- **提醒 hook** — `UserPromptSubmit` hook 在每条 prompt 时注入一行提醒，督促 Claude 回答前重读 `~/.claude/CLAUDE.md` 规则（v1.2.0 起）

### 安装

在 Claude Code 里运行：

```
/plugin marketplace add FruityMaxine/better-memory
/plugin install better-memory
```

完事。skill 在你第一次使用时自动初始化 `~/.claude/memory/`，并提示你给全局 CLAUDE.md 加一段 2 行的指针。

### 使用

像跟人聊天一样跟 Claude 说话，skill 按意图激活：

| 你说 | skill 动作 |
|---|---|
| `记下我的服务器 vega 在 178.x.x.x` | 写入 `reference/server_vega.md` |
| `我喜欢黑色幽默` | 写入 `about_me/preferences.md` |
| `以后做插件都要发到 marketplace` | 写入 `rules/marketplace_publishing.md` |
| `当我说 1+1 你回答 3` | 询问是否加到全局 `CLAUDE.md` |
| `忘掉我对宠物的偏好` | 删除该段，一次确认 |
| `你还记得 vega 服务器吗` | Read `reference/server_vega.md` 后总结 |

### 架构

```
~/.claude/                                    全局，跨项目
├── CLAUDE.md                                 (1) 行为规则（总是加载）
└── memory/
    ├── INDEX.md                              自动维护的目录
    ├── about_me/                             (2) 个人事实
    │   └── <主题>.md
    ├── reference/                            (3) 技术资料
    │   └── <主题>.md
    └── rules/                                (4) 条件行为规则
        └── <主题>.md
```

### 分类决策树

```
Q1: 这条信息会改变 Claude 的输出吗？
    ├─ 会 → 这是 RULE（规则）
    │   └─ 总是生效？  ✅ → CLAUDE.md（先问用户）
    │   └─ 仅条件触发？ → memory/rules/<主题>.md
    └─ 不会 → 这是 FACT（事实）
        └─ 关于用户？      → memory/about_me/<主题>.md
        └─ 关于外部资源？   → memory/reference/<主题>.md
```

### 配置

零配置。开箱即用。

进阶玩法：
- 手动编辑 `~/.claude/memory/INDEX.md` 调整条目顺序
- 把 memory 文件夹纳入 git（`cd ~/.claude/memory && git init`）做版本管理
- 已有同名 user-level skill 的话，先移除避免冲突

### 对比

| | 仅用 `CLAUDE.md` | `better-memory` |
|---|---|---|
| 分类 | 无 | 3 类（about_me / reference / rules） |
| 跨项目 | ✅ | ✅ |
| 条件加载 | ❌（总加载） | ✅（按任务加载） |
| 冲突检测 | ❌ | ✅ |
| 压缩规则 | ❌ | ✅ |
| 自动 INDEX | 无 | ✅ |
| 闲时 token 成本 | 高 | 极低 |

### License

[MIT](LICENSE) © FruityMaxine

### 贡献

欢迎在 [github.com/FruityMaxine/better-memory](https://github.com/FruityMaxine/better-memory) 提 issue 或 PR。
