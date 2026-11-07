# [🚀哔哩哔哩主页](https://space.bilibili.com/3493277319825652)

# goal-prompt-builder

> **一个让你的 `/goal` 不跑偏的 Claude Skill。**

简体中文 · [English](./README.md)

---

OpenAI Codex CLI 0.128 上线了 `/goal` 命令——一个带运行时审计、token 预算、状态机软停的持久化 Agent 目标。它能让 Codex **连续跑 20+ 小时无人值守**完成一件事。

但它有个非常致命的失败模式：**goal 写糊了，Codex 会烧 100K token 把方向跑偏，然后宣告完成**。

这个 skill 就是为了堵住这个失败模式。它会引导 Claude（或者你自己）按 **5 段式黄金模板**（Objective / Scope / Constraints / Done when / Stop if）一步步把 goal 写对，自动检测项目类型、读取 `AGENTS.md` / `CLAUDE.md`、并在你 paste 之前**预测审计友好度**。

---

## 一句话上手

```bash
# 装（一次性）
curl -L -o /tmp/goal-prompt-builder.skill \
  https://github.com/win4r/goal-prompt-builder/raw/main/goal-prompt-builder.skill
mkdir -p ~/.claude/skills && unzip -o /tmp/goal-prompt-builder.skill -d ~/.claude/skills/

# 用（在任何 Claude 对话里说）
你: 帮我写一个 /goal，用来重构鉴权模块
Claude: [skill 自动触发，引导你 6 步，输出可直接 paste 的 /goal]
```

---

## 你拿到的是什么

一个 `.skill` 包，里面装了：

| 文件 | 行数 | 作用 |
|---|---|---|
| `SKILL.md` | ~250 | 主路由 + 6 步 workflow |
| `references/project-types.md` | ~260 | Node / Python / Swift / Go / Rust / 静态项目的默认值 |
| `references/scenarios.md` | ~330 | 7 种场景的骨架（重构 / 新功能 / 批量 / 考古 / UI audit / 守门员 / 自定义） |
| `references/examples.md` | ~260 | 5 个完整的 input → output 案例 |

总共 ~1100 行精心设计的 prompt engineering——从 Codex 真实的 `continuation.md` 审计提示词、`/goal` 的 5 个 PR（#18073-#18077）、以及社区已经踩过的坑（issue #19910 / #20656 / #20792 / #20536）反推出来。

---

## 为什么要这个 skill

`/goal` 自带一段叫 `continuation.md` 的审计提示词，每次 Codex 进入空闲边界就会注入。这段提示词告诉模型：

> 把 objective 翻译成「需求 → 证据」对照表，把每个显式要求、编号项、文件名、命令、测试、关卡、可交付物**映射到具体证据**。**不确定即视为未达成**。

这是 AI coding 工具历史上最有杀伤力的反 sandbagging 机制之一。**但它有一个前提：你的 goal 文本必须能被翻译成对照表**。

大多数人写 goal 的方式是：
```
/goal 修一下所有 flaky 测试，顺便整理一下代码
```

审计期模型没法从"所有"和"整理"里构造出可枚举的对照表——它只能 fall back 到代理信号（"测试跑过了"≈ "我跑了一次没崩"）。结果就是：模型宣告完成、你醒来发现一团糟。

这个 skill 的存在，就是为了让你写出来的 goal **永远能被映射成真正的 audit checklist**。

---

## skill 做什么

触发后，Claude 走 6 步：

1. **选交互模式** — 询问式 / 全描述式 / 混合式（默认）
2. **自动检测项目类型** — 探测 filesystem（看 `package.json`、`Cargo.toml`、`*.xcodeproj` 这些）或者抓 GitHub URL。读取 `AGENTS.md` / `CLAUDE.md`（如果有）。
3. **选场景模板** — 7 种（重构 / SDD 新功能 / 批量任务 / 代码考古 / UI audit / 守门员 / 自定义）
4. **收集 5 段输入** — Objective / Scope / Constraints / Done when / Stop if
5. **预测审计友好度** — 内部打分；分数太低拒绝 render
6. **Render** — 可直接 paste 的 `/goal` + 几句关键设计选择说明

输出长这样：

```
/goal <一句话目标>。

First action: [如果是 SDD] 先读 X、Y、Z 文件并报告计数，等我确认。

Scope: <文件 / 子系统>。

Constraints:
  - <项目类型默认约束>
  - <从 AGENTS.md / CLAUDE.md 提取的规则>

Done when:
  1. <可验证的产出 1 — cite 文件或命令>
  2. <可验证的产出 2>
  ...

Stop if:
  - <可机械检测的条件 1>
  ...

Use a token budget of <N> tokens for this goal.
```

---

## 安装

### 方式 1：一键安装（Claude Code / 支持 Skill 的 Claude.ai）

```bash
curl -L -o /tmp/goal-prompt-builder.skill \
  https://github.com/win4r/goal-prompt-builder/raw/main/goal-prompt-builder.skill
mkdir -p ~/.claude/skills
unzip -o /tmp/goal-prompt-builder.skill -d ~/.claude/skills/
rm /tmp/goal-prompt-builder.skill
```

### 方式 2：clone + 软链接

```bash
git clone https://github.com/win4r/goal-prompt-builder.git
ln -s "$(pwd)/goal-prompt-builder/goal-prompt-builder" ~/.claude/skills/goal-prompt-builder
```

### 方式 3：手动

从 [Releases 页面](https://github.com/win4r/goal-prompt-builder/releases) 下载 `goal-prompt-builder.skill`（或者直接把本仓库的 `goal-prompt-builder/` 目录拷贝过去），放到你的 Claude skills 目录：

- **Claude Code**：`~/.claude/skills/`
- **Claude Desktop**：见 [Anthropic 官方文档](https://docs.claude.com)
- **Cursor / 其他客户端**：查对应客户端的 skills 路径

---

## 怎么用

装好之后，skill 会在你说这些话时自动触发：

- "帮我写一个 /goal …"
- "给 X 设计一个 goal"
- "为这个 repo 做个 /goal"
- "帮我 review 一下这个 goal"
- "我想用 /goal 来 …"
- 任何提到长程 Codex 任务的场景

你**不需要**说"用 goal-prompt-builder 这个 skill"——Claude 会自己识别。

### 完整对话示例

```
你: 我想用 /goal 给我的检索流水线加 Cohere rerank 作为第 5 个 provider。

Claude（载入 skill 后）: 我来帮你做。先选交互模式：
  A. 询问式（最稳）
  B. 全描述式（最快）
  C. 混合式（默认推荐）

你: C

Claude: [自动从 package.json 检测到 Node/TS 项目，读取 AGENTS.md]
  检测到：Node/TypeScript 项目。找到 AGENTS.md，里面有 3 条铁律。
  这听起来是"新功能实现"场景。你已经有 SDD spec 了吗，还是用 freeform 模板？

你: 我有 OpenSpec change，路径是 openspec/changes/add-cohere-rerank/

Claude: 完美——走场景 B（SDD 驱动的新功能）。几个问题：
  1. Token 预算？（基于 scope 大小建议 120K）
  2. AGENTS.md 里有没有我可能漏掉的关键约束需要进 Constraints？
  ...

[几轮对话后]

Claude: 审计友好度 94/100。这是你的 /goal：

```
/goal 严格按照 openspec/changes/add-cohere-rerank/ 的全部规范实现。

First action: 先读 4 个 spec 文件 + AGENTS.md，回报计数。
等我确认后再开始实现。

Scope: design.md 的 "MUST NOT modify" 清单严格遵守。

Constraints:
  - AGENTS.md 的铁律是非协商
  - 不引入新 npm 依赖
  - 实现风格 mirror 既有 4 个 rerank provider

Done when:
  1. tasks.md 每项打勾，cite 文件路径 + commit hash
  2. 每条 SHALL 至少一个通过测试，cite 测试名
  3. 每个 GIVEN/WHEN/THEN scenario 有对应集成测试
  4. `npx tsc --noEmit` 退出码 0
  5. `npm test` 退出码 0；粘贴 summary
  6. README.md 新增 provider 行
  7. CHANGELOG.md 在 Unreleased 加条目

Stop if:
  - 某 task 要求改 MUST NOT 文件
  - SHALL 互相冲突（升级，不要自己决定）
  - 需要 npm install 新依赖
  - 现有 rerank provider 测试开始失败
  - 跑插件测试前没清 jiti 缓存

Use a token budget of 120000 tokens.
```

几个关键设计选择：
- First action 强制先读再报数 — 绕开 `@filename` 引用语法的不确定性
- AGENTS.md 每条铁律同时进 Constraints 和 Stop if（双重防御）
- jiti 缓存这条是从 CLAUDE.md 提取的 — 这是个常被忽略的 false-completion 来源

你: [paste 进 Codex CLI，去开 3 个小时会，回来看到 PR 已经在 GitHub 上等 review 了]
```

---

## 三种交互模式怎么选

| 模式 | 什么时候用 | 多久能 render |
|---|---|---|
| **A. 询问式** | 第一次写 /goal；高风险重构 | 约 5 分钟 |
| **B. 全描述式** | 你已经想透了；要快 | 约 1 分钟 |
| **C. 混合式** | 日常使用；安全和速度平衡 | 约 2-3 分钟 |

交互模式只影响 skill **怎么收集你的输入**，不影响输出质量。三种模式给同样的信息，会产出同样审计友好的 goal。

---

## 为什么要自动检测项目类型

因为 **Stop if** 是大多数 goal 失败的真正源头——而 stop-if 规则**严重依赖项目类型**：

- Swift 项目？要加"不许改 `project.pbxproj`"（用 PBXFileSystemSynchronizedRootGroup 会自动 pick up 新文件）
- Node 项目？要加"不许 `npm install` 新依赖"
- Python 项目？要加"不许往 `requirements.txt` 加新条目"
- iOS 项目用模拟器？要加"iPhone N 模拟器不可用时停下来跑 `xcrun simctl list` 让我决定换哪个"

没有自动检测，这些规则得靠用户记。有了自动检测，skill 默默帮你 load 进去。

检测优先级：
1. Filesystem 探测——`ls` 找标志性文件（`package.json` / `Cargo.toml` / `*.xcodeproj` 等）
2. URL 抓取——如果用户给了 GitHub 链接，去抓 README
3. 读 `AGENTS.md` / `CLAUDE.md`（如有），里面的项目级规则会**覆盖默认值**
4. fallback——问用户

检测成功后，skill **会先 announce 它检测到了什么**，给你一个机会纠正：

> 检测到这是一个 Swift / iOS 项目（找到 lingolearn.xcodeproj + CLAUDE.md）。
> 我会按 Swift 项目默认约束适配。如果不对告诉我。

---

## skill 不做什么

- ❌ 不会替你跑 `/goal` —— 只生成 prompt 文本
- ❌ 不会校验项目状态（不跑 `git status`、不跑测试）
- ❌ 不支持 0.128 之前的 Codex（那时候没 `/goal`）
- ❌ 不为 `/plan`、`/compact` 等其他 Codex 斜杠命令生成 prompt

---

## skill 内置的硬规则

这些规则全部从分析 Codex 真实的 `continuation.md` 审计提示词反推出来：

1. **拒绝虚词**——"改进"、"优化"、"清理"、"全部"、"彻底"、"all"、"everything" 触发推回
2. **强制 token 预算**——没预算 = 没软停 = 可能跑飞
3. **强制 regression 兜底**——任何涉及测试代码的 goal 都自动加上"不许靠改测试通过"的 stop-if
4. **SDD goal 强制先读再报数**——绕开 `@filename` 引用语法是否被 Codex 解析的不确定性
5. **brownfield 项目强制问 "MUST NOT modify" 清单**——这是 scope 失控的头号原因
6. **审计友好度低于 70% 拒绝 render**——逼用户先把 goal 改好

完整规则在 `goal-prompt-builder/SKILL.md` 的 "Hard rules" 段落。

---

## 兼容性

- **Codex CLI**：0.128.0+（再老的版本没 `/goal`）
- **Claude**：任何支持 Claude Skills 的客户端（Claude Code / Claude Desktop / 支持 skill 的 Claude.ai）
- **测试过的项目类型**：Node、Python、Swift / iOS、Go、Rust、静态 / 文档站点
- **语言**：skill 双语（中 + 英），根据用户输入生成对应语言的 goal

---

## 仓库结构

```
goal-prompt-builder/
├── README.md                       (英文版)
├── README_CN.md                    (这份)
├── LICENSE                         (MIT)
├── goal-prompt-builder.skill       (打包好的，可直接装)
└── goal-prompt-builder/            (源码)
    ├── SKILL.md                    (主 skill — 路由 + 6 步 workflow)
    └── references/
        ├── project-types.md        (各语言默认值)
        ├── scenarios.md            (7 种场景骨架)
        └── examples.md             (5 个完整案例)
```

---

## 修改后重新打包

`.skill` 文件就是 `goal-prompt-builder/` 文件夹的 zip。改完之后重新打包：

```bash
cd goal-prompt-builder
zip -r goal-prompt-builder.skill goal-prompt-builder/ \
  -x "*.DS_Store" "*__pycache__*"
```

或者用 Anthropic 官方的 `package_skill.py`（在 [skill-creator](https://github.com/anthropics/skills) 里），一步搞定校验 + 打包。

---

## 想搞懂背后的设计

如果你想理解 skill **为什么**长这样：

- **Codex 0.128 changelog** —— `/goal` 的官方发布说明
- **PR #18073-#18077** —— 构建 `/goal` 的 5 个 PR
- **`continuation.md` 模板** —— 每次空闲边界注入的审计提示词（[commit `6014b66`](https://github.com/openai/codex)）
- **Issue #20536** —— `/goal` 的官方文档缺口
- **Issue #20656** —— Plan 模式静默压制 goal 续跑（最致命的坑）
- **Issue #20792** —— `/goal` 作为线程第一条消息会让 codex resume 列表找不到
- **Issue #19910** —— 续跑期手动 `/compact` 会丢失 goal 上下文
- **Simon Willison 的解读** —— 目前公开的最佳解释（2026 年 4 月 30 日）

---

## License

MIT —— 见 [LICENSE](./LICENSE)。随便用，包括商用。

---

## 欢迎 PR

特别欢迎以下方向的贡献：

- 新的项目类型默认值（Elixir、Ruby、Java、Kotlin 等）
- 现有 7 种场景之外的新场景骨架
- 你在生产环境真实跑过的 goal 案例（脱敏后）
- 把 `SKILL.md` 翻译成其他语言（日文、韩文等）

如果是想改核心的 6 步 workflow，请先开 issue 讨论。

---

## 相关项目

- **OpenAI Codex** —— [github.com/openai/codex](https://github.com/openai/codex)
- **OpenSpec** —— 跟这个 skill 配合极佳（[openspec.dev](https://openspec.dev)）
- **GitHub Spec Kit** —— 另一种 SDD 工具（[github.com/github/spec-kit](https://github.com/github/spec-kit)）
- **Anthropic Skill Creator** —— 用来构建 skill 的官方 skill

---

*献给那些受够了 `/goal` 跑 4 小时然后做错东西的开发者。*
