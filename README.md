[Bilibili profile](https://space.bilibili.com/3493277319825652)

# goal-prompt-builder

> **A Claude Skill for writing high-quality `/goal` prompts that don't go off the rails.**

[简体中文](./README_CN.md) · English

---

OpenAI Codex CLI 0.128 introduced `/goal` — a persistent agent objective with runtime audit, token budget, and a soft-stop state machine. It can drive a single task for **20+ hours unattended**.

But it has one ugly failure mode: write the goal vaguely, and Codex will burn 100K tokens going in the wrong direction, then declare success.

This skill makes that failure mode hard to hit. It walks Claude (or you) through a **5-section golden template** (Objective / Scope / Constraints / Done when / Stop if), auto-detects your project type, reads your `AGENTS.md` / `CLAUDE.md`, and predicts audit-friendliness *before* you run the command.

---

## TL;DR

```bash
# Install (one-time)
curl -L -o /tmp/goal-prompt-builder.skill \
  https://github.com/win4r/goal-prompt-builder/raw/main/goal-prompt-builder.skill
mkdir -p ~/.claude/skills && unzip -o /tmp/goal-prompt-builder.skill -d ~/.claude/skills/

# Use (in any Claude conversation)
You: help me write a /goal for refactoring the auth module
Claude: [skill triggers, walks you through 6 steps, outputs a paste-ready /goal]
```

---

## What you get

A `.skill` bundle containing:

| File | Lines | Purpose |
|---|---|---|
| `SKILL.md` | ~250 | Main routing logic + 6-step workflow |
| `references/project-types.md` | ~260 | Defaults for Node / Python / Swift / Go / Rust / static |
| `references/scenarios.md` | ~330 | Skeletons for 7 scenarios (refactor / feature / batch / archaeology / UI audit / gatekeeper / custom) |
| `references/examples.md` | ~260 | 5 worked input→output transformations |

Total ~1100 lines of carefully designed prompt engineering — distilled from Codex's actual `continuation.md` audit prompt, the `/goal` PR stack (#18073-#18077), and the public issue tracker (#19910, #20656, #20792, #20536).

---

## Why this exists

`/goal` ships with a built-in audit prompt (`continuation.md`) that runs after every idle boundary. It tells the model:

> Build a prompt-to-artifact checklist that maps every explicit requirement, numbered item, named file, command, test, gate, and deliverable to concrete evidence. **Treat uncertainty as not achieved.**

This is one of the strongest anti-sandbagging mechanisms ever shipped in an AI coding tool. **But it only works if your goal text can be mapped to a checklist.**

Most users write goals like:
```
/goal Fix all the flaky tests and clean up the codebase
```

The audit can't construct a checklist from "all" or "clean up" — so it falls back to proxy signals like "tests passed" (which can mean "I ran them once and they didn't crash"). Result: the model declares done, you wake up to broken code.

This skill exists to make sure your goal text is **always** mappable to a real audit checklist.

---

## What the skill does

When triggered, Claude walks 6 steps:

1. **Pick interaction mode** — step-by-step, full-description, or hybrid (default)
2. **Auto-detect project type** — by inspecting filesystem (`package.json`, `Cargo.toml`, `*.xcodeproj`, etc.) or fetching repo URL. Reads `AGENTS.md` / `CLAUDE.md` if present.
3. **Pick scenario template** — 7 templates (refactor / SDD feature / batch / archaeology / UI audit / gatekeeper / custom)
4. **Gather 5 inputs** — Objective / Scope / Constraints / Done when / Stop if
5. **Predict audit-friendliness** — internal scoring; refuses to render if too vague
6. **Render** — copy-pasteable `/goal` + brief design rationale

Output looks like:

```
/goal <objective>.

First action: [if SDD-driven] read X, Y, Z and report counts. Wait for ack.

Scope: <files / subsystem>.

Constraints:
  - <project-type-specific defaults>
  - <rules from AGENTS.md / CLAUDE.md>

Done when:
  1. <verifiable artifact 1 — cite file or command>
  2. <verifiable artifact 2>
  ...

Stop if:
  - <mechanically detectable condition 1>
  ...

Use a token budget of <N> tokens for this goal.
```

---

## Installation

### Option 1: One-line install (Claude Code / Claude.ai with skill support)

```bash
curl -L -o /tmp/goal-prompt-builder.skill \
  https://github.com/win4r/goal-prompt-builder/raw/main/goal-prompt-builder.skill
mkdir -p ~/.claude/skills
unzip -o /tmp/goal-prompt-builder.skill -d ~/.claude/skills/
rm /tmp/goal-prompt-builder.skill
```

### Option 2: Clone and link

```bash
git clone https://github.com/win4r/goal-prompt-builder.git
ln -s "$(pwd)/goal-prompt-builder/goal-prompt-builder" ~/.claude/skills/goal-prompt-builder
```

### Option 3: Manual

Download `goal-prompt-builder.skill` from [the latest release](https://github.com/win4r/goal-prompt-builder/releases) (or copy the `goal-prompt-builder/` folder from this repo) into your Claude skills directory:

- **Claude Code**: `~/.claude/skills/`
- **Claude Desktop**: see [Anthropic's docs](https://docs.claude.com)
- **Cursor / other clients**: check the client's documentation for skill paths

---

## Usage

Once installed, the skill triggers automatically on phrases like:

- "help me write a /goal for ..."
- "design a goal for X"
- "make a /goal command for this repo"
- "review my goal command"
- "I want to use /goal to ..."
- (any mention of long-running Codex tasks)

You don't need to type "use the goal-prompt-builder skill" — Claude will pick it up.

### Example session

```
You: I want to use /goal to add Cohere rerank as a 5th provider to my retrieval pipeline.

Claude (with skill loaded): I'll help you build that. First — interaction mode?
  A. Step-by-step (safest)
  B. Full description (fastest)
  C. Hybrid (default, recommended)

You: C

Claude: [auto-detects Node/TS project from package.json, reads AGENTS.md]
  Detected: Node/TypeScript project. Found AGENTS.md with 3 iron rules.
  This sounds like a "feature implementation" scenario. Do you have a SDD spec
  already, or should we go with a freeform feature template?

You: I have an OpenSpec change at openspec/changes/add-cohere-rerank/

Claude: Perfect — that maps to scenario B (SDD-driven feature). A few questions:
  1. Token budget? (suggest 120K based on scope)
  2. Anything in AGENTS.md that I should pull into Constraints I might miss?
  ...

[after a few rounds]

Claude: Audit-friendliness: 94/100. Here's your /goal:

```
/goal Implement openspec/changes/add-cohere-rerank/ exactly as specified.

First action: read the 4 spec files + AGENTS.md and report counts.
Wait for my acknowledgment before implementation.

Scope: design.md "MUST NOT modify" list strictly enforced.

Constraints:
  - AGENTS.md iron rules are non-negotiable
  - No new npm dependencies
  - Mirror the existing 4 rerank providers' code style

Done when:
  1. Each task in tasks.md checked off, cite file path + commit
  2. Each SHALL has a passing test, cite test name
  3. Each GIVEN/WHEN/THEN scenario covered by integration test
  4. `npx tsc --noEmit` exits 0
  5. `npm test` exits 0; paste summary
  6. README.md updated with new provider row
  7. CHANGELOG.md has Unreleased entry

Stop if:
  - A task requires modifying a MUST NOT file
  - SHALLs conflict (escalate, don't decide)
  - npm install needed for new dep
  - Existing rerank provider tests fail
  - jiti cache wasn't cleared before plugin tests

Use a token budget of 120000 tokens.
```

Key design choices:
- First action enforces read+report — sidesteps `@filename` ref uncertainty
- Each AGENTS.md rule maps to both Constraint AND Stop-if (defense in depth)
- jiti cache rule pulled from CLAUDE.md — easily missed false-completion source

You: [pastes into Codex CLI, walks away for 3 hours, comes back to a green PR]
```

---

## Why three interaction modes?

| Mode | When to use | Time to render |
|---|---|---|
| **A. Step-by-step** | First time using `/goal`; high-stakes refactor | ~5 min |
| **B. Full description** | You've thought through the task; want speed | ~1 min |
| **C. Hybrid** | Daily use; balanced safety + speed | ~2-3 min |

The interaction mode affects **how** the skill collects your input, not the quality of the output. All three produce equally audit-friendly goals when you give them equal information.

---

## Why auto-detect project type?

Because the **Stop if** section is where most goals fail — and Stop-if rules are heavily project-type-specific:

- Swift project? Add "do not modify `project.pbxproj`" (PBXFileSystemSynchronizedRootGroup will pick up new files automatically)
- Node project? Add "no `npm install` for new deps"
- Python project? Add "no new entries in `requirements.txt`"
- iOS project with iPhone simulators? Add "halt if iPhone N simulator unavailable, run `xcrun simctl list` and let me decide"

Without auto-detection, the user has to remember these. With auto-detection, the skill loads the right defaults silently.

The detection priority:
1. Filesystem probe — `ls` for telltale files (`package.json`, `Cargo.toml`, etc.)
2. URL fetch — if user gave a GitHub link, fetch the README
3. Read `AGENTS.md` / `CLAUDE.md` if present (these contain project-specific rules that override defaults)
4. Fallback — ask the user

If detection succeeds, the skill **announces what it found** so you can correct it before proceeding.

---

## What the skill does NOT do

- ❌ Does not run `/goal` for you — only generates the prompt text
- ❌ Does not validate your project state (no `git status` checks, no test runs)
- ❌ Does not handle Codex versions older than 0.128 (`/goal` doesn't exist there)
- ❌ Does not generate prompts for `/plan`, `/compact`, or other Codex slash commands

---

## Hard rules baked into the skill

These come from analyzing Codex's actual `continuation.md` audit prompt:

1. **Reject vague verbs** — "improve", "optimize", "clean up", "all", "everything", "全部", "彻底" trigger pushback
2. **Force token budgets** — missing budget = no soft stop = potential runaway
3. **Force regression guards** — any goal touching tested code gets a "do not edit tests to make them pass" stop-if
4. **Force read+report first** for SDD-driven goals — sidesteps `@filename` reference uncertainty
5. **Force "MUST NOT modify" probing** for brownfield projects — #1 cause of scope creep
6. **Refuse to render** if audit-friendliness score < 70%

The full rule set is in `goal-prompt-builder/SKILL.md` under "Hard rules".

---

## Compatibility

- **Codex CLI**: 0.128.0+ (where `/goal` exists)
- **Claude**: any client that supports Claude Skills (Claude Code, Claude Desktop, Claude.ai with skill support)
- **Project types tested**: Node, Python, Swift / iOS, Go, Rust, static / docs sites
- **Languages**: skill is bilingual (English + Chinese); generates prompts in either based on user input

---

## Repository structure

```
goal-prompt-builder/
├── README.md                       (this file)
├── README_CN.md                    (中文版)
├── LICENSE                         (MIT)
├── goal-prompt-builder.skill       (packaged, ready to install)
└── goal-prompt-builder/            (source)
    ├── SKILL.md                    (main skill — routing + 6-step workflow)
    └── references/
        ├── project-types.md        (per-language defaults)
        ├── scenarios.md            (7 scenario skeletons)
        └── examples.md             (5 worked examples)
```

---

## Updating

The `.skill` file is just a zip of the `goal-prompt-builder/` folder. To rebuild after editing:

```bash
cd goal-prompt-builder
zip -r goal-prompt-builder.skill goal-prompt-builder/ \
  -x "*.DS_Store" "*__pycache__*"
```

Or use Anthropic's official `package_skill.py` from the [skill-creator skill](https://github.com/anthropics/skills) for validation + packaging in one step.

---

## Background reading

If you want to understand *why* the skill is shaped the way it is:

- **Codex 0.128 changelog** — official `/goal` announcement
- **PR #18073-#18077** — the 5-PR stack that built `/goal`
- **`continuation.md` template** — the audit prompt that runs every idle boundary ([commit `6014b66`](https://github.com/openai/codex))
- **Issue #20536** — `/goal` documentation gap
- **Issue #20656** — Plan-mode silently suppresses goal continuation
- **Issue #20792** — `/goal`-first sessions missing from `codex resume` lists
- **Issue #19910** — mid-turn `/compact` loses goal context
- **Simon Willison's writeup** — the best public explainer (Apr 30, 2026)

---

## License

MIT — see [LICENSE](./LICENSE). Use it however you want, including commercially.

---

## Contributing

PRs welcome, especially:

- New project type defaults (Elixir, Ruby, Java, Kotlin, etc.)
- New scenario skeletons not covered by the current 7
- Worked examples from your real production goals
- Translation of `SKILL.md` into other languages (Japanese, Korean, etc.)

For bugs or proposed changes to the core 6-step workflow, please open an issue first to discuss.

---

## Related

- **OpenAI Codex** — [github.com/openai/codex](https://github.com/openai/codex)
- **OpenSpec** — pairs beautifully with this skill ([openspec.dev](https://openspec.dev))
- **GitHub Spec Kit** — alternative SDD tool ([github.com/github/spec-kit](https://github.com/github/spec-kit))
- **Anthropic Skill Creator** — the official skill that builds skills

---

*Built for developers who got tired of `/goal` jobs running 4 hours and producing the wrong thing.*
