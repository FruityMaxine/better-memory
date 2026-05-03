---
name: better-memory
description: Cross-session personal memory system for storing user facts, preferences, technical resources, and conditional behavior rules across all projects. Use whenever the user expresses intent to save / record / remember / store / recall / forget / update any piece of information across sessions — detect intent by meaning, NOT keyword matching. Triggers include phrases like "记下", "记一下", "save this", "remember this", "存到 memory", "以后要记得", "I want you to know X", "from now on...", "忘掉", "去除", "delete this memory", as well as less explicit intents (the user describing a fact about themselves or a resource they want preserved). The skill maintains 3 categorized folders under ~/.claude/memory/ with an auto-synchronized INDEX.md, and routes content into one of four destinations: CLAUDE.md (always-loaded behavioral rules), about_me/ (personal facts that don't affect default behavior), reference/ (cross-project technical resources), or rules/ (conditional behavioral rules loaded only when relevant context appears).
---

# better-memory

Cross-project personal memory management. Acts when the user wants to save, update, recall, or delete information across sessions. Decides where each piece of information belongs, writes it cleanly, and keeps the index synchronized.

## When to activate

Activate by **intent**, not keyword. The user is asking to save / update / forget / recall personal information whenever the intent — expressed naturally — falls into one of these:

- **Save** anything the user wants preserved across sessions: facts, preferences, technical info, behavior rules
- **Update** existing memory when something changed
- **Delete** information the user no longer wants kept
- **Search / Recall** when the user asks "what do you remember about X"

Examples of activation:

| User says | Activate? |
|---|---|
| "记下我的服务器 IP 是 X" | ✅ |
| "save this to memory" | ✅ |
| "以后我说部署都默认部署到 vega" | ✅ (rule type) |
| "我喜欢猫" | ✅ (about_me type) |
| "忘掉那个偏好" | ✅ (delete) |
| "你还记得我那个服务器吗" | ✅ (recall — read-only) |
| "1+1 等于几" | ❌ |
| "帮我写代码" | ❌ |

Do not require exact phrasing. Detect intent.

## Architecture (4 destinations)

```
~/.claude/                                    Global, cross-project
├── CLAUDE.md                                 (1) Behavior rules (always loaded)
└── memory/
    ├── INDEX.md                              Auto-managed catalog
    ├── about_me/                             (2) Personal facts (no behavior impact)
    │   └── <topic>.md
    ├── reference/                            (3) Technical resources (cross-project)
    │   └── <topic>.md
    └── rules/                                (4) Conditional behavior rules
        └── <topic>.md
```

**Path is always `~/.claude/memory/`** (cross-project). Never write project-scoped memory under this skill — those belong to project-level memory systems handled by other plugins.

## Classification decision tree

When a user wants to save a piece of information, run this tree before writing:

```
Q1: Will this information change Claude's output in some answer?
    │
    ├─ YES → It's a RULE
    │   │
    │   ├─ Q2: Does it apply to >50% of sessions / always-on?
    │   │   ├─ YES → CLAUDE.md (offer to add a line, ask user before editing CLAUDE.md)
    │   │   └─ NO  → memory/rules/<topic>.md
    │
    └─ NO  → It's a FACT
        │
        ├─ Q3: Is it about the user as a person?
        │   ├─ YES → memory/about_me/<topic>.md
        │   └─ NO  → memory/reference/<topic>.md
```

**Why this matters**: misclassification causes either token waste (rules in CLAUDE.md that rarely fire) or missed enforcement (rules buried in reference/ that never trigger).

**Edge cases**:
- "I prefer cats" — open creative tasks (story names, recommendations) might use this. → about_me. The skill caller will Read about_me on relevant tasks.
- "When publishing my Claude plugin, always upload to marketplace" — only relevant during plugin publishing → rules/.
- "When I say 1+1, you reply 3" — affects every literal answer → CLAUDE.md (always-on rule).

When ambiguous: **ask the user**, do not guess.

## File granularity

**Default: coarse.** One real-world entity or one coherent concept = one file. Split lazily.

| Group into one file when… | Split into separate files when… |
|---|---|
| Same physical entity (one server = one file) | File grows >2000 words AND has clear sub-topics |
| Same coherent concept (all dietary preferences) | Sub-topics are independently asked about |
| Several related small facts | User explicitly asks to split |

**Examples**:
- ✅ One server → `reference/server_vega.md` (IP, SSH, hardware, billing, all in one)
- ✅ Lifestyle preferences → `about_me/lifestyle.md` (pets, food, schedule)
- ✅ One GitHub account → `reference/account_github.md`
- Splitting `basic_math.md` into `addition.md` + `multiplication.md` is **wrong** unless user has explicitly demonstrated they query addition and multiplication separately enough to justify

When the file grows, raise the question: *"This file is getting long. Would you like me to split out [sub-topic]?"* Don't split silently.

## File naming

Topic-based, no type prefix. The folder is the type.

- ✅ `reference/server_vega.md`
- ✅ `about_me/basics.md`
- ✅ `rules/marketplace_publishing.md`
- ❌ `reference/reference_server_vega.md` (redundant prefix)
- ❌ `reference/vega.md` (too generic — server_vega is unambiguous)

Lowercase + underscore. ASCII only when possible (helps cross-platform shell handling).

## frontmatter template

Every memory file uses this format. The `type` field must be exactly one of three values: `about_me`, `reference`, or `rules`.

```markdown
---
name: <human-readable title>
description: <one sentence — this goes into INDEX.md verbatim>
type: about_me   # pick exactly one of: about_me | reference | rules
---

# <title>

<body>
```

**No timestamps.** Version history lives in git if the user chooses to git-init the memory folder.

## INDEX.md format

`~/.claude/memory/INDEX.md` is the catalog. Always loaded by Claude at the start of each conversation (per CLAUDE.md instruction). Keep it tight — only file path + description.

```markdown
# Memory Index

## about_me/
- [basics.md](about_me/basics.md) — name, gender, profession, personal background
- [lifestyle.md](about_me/lifestyle.md) — pets, food, schedule, hobbies

## reference/
- [server_vega.md](reference/server_vega.md) — Hetzner CX23 main server (IP, SSH, billing, deployment)
- [account_hetzner.md](reference/account_hetzner.md) — Hetzner Cloud account details

## rules/
- [marketplace_publishing.md](rules/marketplace_publishing.md) — auto-upload Claude plugins to marketplace after creation
```

INDEX uses the `description` field from each file's frontmatter — keep them concise (no length limit, but aim for one informative sentence).

**INDEX update triggers**:
- New file created → add entry
- File deleted → remove entry
- A file's `description` field changed substantively → update its entry
- Content added to a file but description unchanged → no INDEX change

## Write protocol

1. **Search first.** `grep -ri <key topic>` under `~/.claude/memory/`. If a related file exists, append/update; don't create a duplicate.
2. **Compress.** Strip filler ("嗯", "you know", "I mean"). Distill to facts/preferences. See compression rules below.
3. **Classify.** Run the decision tree.
4. **Pick filename**: `<type-folder>/<topic>.md`.
5. **Write or update file** (create folder lazily if missing).
6. **Update INDEX.md** atomically — same write cycle.
7. **Confirm to user** — one short line: "Saved [topic] to memory/[type]/[file]."

When the destination is **CLAUDE.md** (a behavior rule), do NOT write it directly. Tell the user: *"This is a behavior rule — I'll need to add it to your global CLAUDE.md. Want me to do that?"* and only edit on confirmation.

## Compression rules

User speaks naturally; memory must be terse. **Preserve nuance, drop noise.**

| Information type | What to keep |
|---|---|
| Hard facts (numbers, names, IPs, dates) | Verbatim |
| Preferences and attitudes | Core meaning + relative ranking ("猫优先 / 狗也行") |
| Reasoning / explanation | Drop, keep only the conclusion |
| Conversational scaffolding ("嗯", "我那次") | Drop unless essential to interpretation |

**Example**:

User said: *"嗯就是说我那个我比较喜欢猫但是我也不讨厌狗就是说狗也行猫更好你懂的"*

Memory writes:
```markdown
## 宠物偏好
- 猫（首选）
- 狗（也接受）
```

Don't paraphrase important nouns (product names, person names) — keep them verbatim. Don't strip nuance ("猫更好" vs "也行" is meaningful — both must survive).

## Update protocol

When updating existing memory, the action depends on **what kind of change**:

| Change type | Action |
|---|---|
| Objective fact updated (server IP changed, price changed) | **Silent overwrite**. Replace the old value. |
| Preference replaced ("was cats, now dogs") | **Ask** — "替换成狗，还是猫和狗都喜欢？" |
| Preference added ("also like dogs, still like cats") | Append, no question needed |
| Major rewrite (>30% of file content) | Show diff preview, confirm before writing |

Never preserve "originally / now" history in memory — that bloats the file. If the user wants history, they can git-init the memory folder.

## Conflict / ambiguity protocol

**Never guess.** If any of these hold, ask the user:

- Classification is unclear (could be about_me or rules)
- The new info contradicts existing memory
- The user's phrasing has multiple plausible meanings
- The destination filename is ambiguous (could append to file A or B)

When asking, **give choices**: *"你是指 A 还是 B?"* — better than open-ended *"能澄清一下吗?"*

Until clarified, **do not write**.

## Recall / read protocol

When the user asks "what do you know about X" or "do you remember Y":

1. Read INDEX.md (if not already in context)
2. Identify candidate files by description
3. Read relevant file(s)
4. Summarize back to user, citing source: *"From `reference/server_vega.md`: IP is X, SSH alias is `vega`."*

Don't dump the whole file unless the user asks for it.

## Delete protocol

Triggered by **intent** — detect by meaning, not exact keyword. Example signals (not exhaustive): "忘掉", "去除", "删掉", "forget", "remove", "don't remember X anymore", "丢掉那条记忆".

1. Search memory for matching content
2. Show user the candidate(s): *"Found these matches: [list]. Which to remove?"*
3. **Confirm once** — then delete (no double confirmation; that's annoying)
4. Delete granularity:
   - Whole file is about this thing → delete file + remove INDEX entry
   - Just one section → remove section, keep file, INDEX unchanged (unless description shifts)
5. **CLAUDE.md content is never auto-deleted** — only suggest the user remove it manually, since CLAUDE.md is their hard control surface

## Initialization (first use)

When the skill triggers and `~/.claude/memory/` doesn't exist:

1. Create `~/.claude/memory/` and an empty `INDEX.md` immediately. **Don't ask** — folder creation is harmless.
2. Subdirectories (`about_me/`, `reference/`, `rules/`) are created **lazily** the first time content is written to that type.
3. Check global CLAUDE.md for the memory pointer paragraph. If absent, **ask the user**:
   > *"To make this memory system work across sessions, I need to add a short pointer paragraph to your global CLAUDE.md so I know to read INDEX.md at the start of each conversation. Without it, future sessions won't know your memory exists. Add it now?"*
4. On confirmation, append the pointer paragraph to `~/.claude/CLAUDE.md`. On refusal, proceed but warn that future sessions won't auto-discover memory.

The pointer paragraph to add (verbatim):

```markdown
## 个人记忆系统

- **必读索引**：每次对话开始前 Read `~/.claude/memory/INDEX.md`，按其中描述决定是否读取具体记忆文件。
- **必用 skill**：所有记忆相关操作（记录 / 更新 / 删除 / 查询）使用 `better-memory` skill。
```

## Anti-patterns

Don't:

- **Auto-write CLAUDE.md without asking** — it's the user's primary control surface; ask first
- **Hard-keyword-match for triggers** — detect intent (you're an LLM, behave like one)
- **Create files without searching first** — duplicates pollute the system
- **Silently overwrite preferences** — preference changes deserve explicit confirmation
- **Put behavior rules in about_me/ or reference/** — they won't get enforced
- **Put facts in rules/** — they'll trigger inappropriate behavior
- **Update INDEX on every micro-edit** — INDEX should be stable; only update when "what's available" changes
- **Quote user's raw words verbatim** — compress to facts, except for proper nouns
- **Compress so aggressively that nuance is lost** — "猫优先 / 狗也行" carries meaning; "喜欢动物" doesn't
- **Pre-create all subdirectories** — lazy creation only; clean filesystem
- **Skip user confirmation on deletes** — one confirmation is required; one only
- **Try to retain "originally / now" history in memory** — git is for that

## Examples

### Example 1: Personal preference (about_me)

User: *"记下我喜欢黑色幽默"*

1. Search: no related file in `about_me/`.
2. Classify: not a behavior rule → about_me/ → topic = humor preferences.
3. Filename: `about_me/preferences.md` (start a general preferences file rather than `about_me/humor.md`, since more preferences will likely accumulate).
4. Write:
   ```markdown
   ---
   name: 个人偏好
   description: 用户的口味、风格、内容偏好
   type: about_me
   ---
   # 个人偏好
   ## 幽默
   - 喜欢黑色幽默
   ```
5. Update INDEX.md.
6. Confirm: *"Saved 黑色幽默偏好 to memory/about_me/preferences.md."*

### Example 2: Technical resource (reference)

User: *"记下我的服务器 vega，IP 是 178.x.x.x，SSH 别名 vega 和 vega-443，私钥在 ~/.ssh/id_ed25519"*

1. Search: no `server_vega.md` exists.
2. Classify: technical resource, cross-project → reference/.
3. Filename: `reference/server_vega.md`.
4. Write file with all the details + frontmatter.
5. Update INDEX.md with: `[server_vega.md] — Hetzner 主服务器 (vega, IP, SSH, 部署路径)`.
6. Confirm.

### Example 3: Conditional rule (rules)

User: *"以后我做 Claude 插件的时候都要上传到 marketplace"*

1. Search: no related rules file.
2. Classify: behavior rule, but conditional (only when working on Claude plugins) → rules/.
3. Filename: `rules/claude_plugin_publishing.md`.
4. Write rule with frontmatter.
5. INDEX entry: *"做 Claude 插件后必须上传到 marketplace"*.
6. Confirm.

### Example 4: Always-on rule (CLAUDE.md)

User: *"以后我说 1+1 你要回答 3"*

1. Classify: behavior rule, always-on → CLAUDE.md.
2. **Do not auto-write CLAUDE.md.** Ask:
   > *"This is a behavior rule that affects every answer — it should go in your global CLAUDE.md. Want me to add it?"*
3. On confirmation, append to `~/.claude/CLAUDE.md`.

### Example 5: Preference change (update with question)

User (later): *"现在我更喜欢狗"*

1. Search: find `about_me/preferences.md` containing "猫优先".
2. Detect potential conflict.
3. **Ask**: *"你之前说猫优先，狗也行。是改成狗优先（猫降级），还是改成猫狗并列？"*
4. Update according to answer.
5. INDEX unchanged (description still "用户的口味、风格、内容偏好").

### Example 6: Delete

User: *"忘掉我对宠物的偏好"*

1. Search: find `about_me/preferences.md` containing pet preferences.
2. **Show**: *"Found 宠物偏好 in `about_me/preferences.md`. Remove just this section? (rest of preferences file stays)"*
3. On confirmation, remove the `## 宠物偏好` section from the file.
4. INDEX unchanged (file still exists, description still applicable).

## Why this design

The system separates **what's loaded** from **what's stored**:

- CLAUDE.md is loaded into every session → expensive → only put always-relevant rules there
- INDEX.md is loaded at the start of each conversation → cheap (small) → tells Claude what specific knowledge is *available*
- Individual memory files are loaded only when their description matches the task → zero cost when irrelevant

This three-tier loading is the core economic argument: storing knowledge has near-zero cost; loading it on every prompt has real cost. The skill optimizes for "store everything; load only what's needed".
