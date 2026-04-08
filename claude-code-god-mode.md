# Claude Code God Mode — 5 Unlocks Most Devs Ignore

You installed Claude Code. You ran `claude`. You typed prompts. It worked.

But you're using maybe 30% of what it can do.

The other 70% is hidden behind 5 features that Anthropic shipped quietly — and the engineers who use them ship 3x faster while spending half the tokens. This is the God Mode setup.

Every claim in this article is from official Anthropic docs, the Claude Code GitHub repo, or verified power-user blogs. Sources at the bottom.

---

## The Problem — Why Most Devs Burn Through Tokens

On March 31, 2026, Anthropic officially admitted that Claude Code users were "hitting limits way faster than expected" ([The Register](https://www.theregister.com/2026/03/31/anthropic_claude_code_limits/)). Reddit was full of devs reporting their $100 Max plan getting eaten in under an hour.

The problem isn't pricing. It's that most devs use Claude Code like a chatbot — one giant session, every model is Opus, every file gets read top-to-bottom, no `/clear`, no subagents, no skills. Token usage explodes.

GitHub issue [anthropics/claude-code#13579](https://github.com/anthropics/claude-code/issues/13579) documents over **700,000 wasted tokens** across common anti-patterns:

| Mistake | Tokens used | Tokens after fix | Saving |
|---|---|---|---|
| Build before grep | 70K | 2K | 97% |
| Parallel agents on shared files | 300K | 20K | 93% |
| Write 10 files then test | 124K | 10K | 92% |
| Overengineering beyond request | 112K | 2K | 98% |

The fix isn't a better plan. It's **using the features you already have**.

Here are the 5 unlocks.

---

## Unlock 1 — Plan Mode (Shift+Tab×2)

Press `Shift+Tab` twice. The status bar changes to "Plan Mode." Now Claude does read-only research with a Haiku-powered Explorer subagent — not Opus.

**What it does internally:**
- Spawns a Haiku subagent that searches the codebase
- Returns a structured plan to your main session
- No files get edited until you approve

**Why this saves tokens:**
- Haiku is roughly **1/12 the price of Opus** per token
- The exploration phase (which is usually the longest) happens in an isolated context — your main window stays clean
- You see the plan before any expensive write operation begins

**Anthropic's own rule:** *"If you could describe the diff in one sentence, skip the plan."* ([Anthropic best practices](https://code.claude.com/docs/en/best-practices)). Plan mode is for multi-file changes, unfamiliar code, or when you're not sure of the approach.

You can also press `Ctrl+G` to open the plan in your editor and edit it before executing — this is the cheapest way to course-correct.

> **Takeaway:** Plan Mode = Haiku-powered exploration + write-blocked safety. Use it whenever the task touches more than one file.

---

## Unlock 2 — Subagents (Parallel Fan-Out)

A subagent runs in its **own context window**. Only the final summary returns to your main session.

This is the difference between reading 50 files and getting one paragraph back, vs reading 50 files and dragging all 50 into your current context.

**The right way:**
```
Research X, Y, Z in parallel — three independent subagents.
Each agent owns its own files, returns its own summary.
```

**The wrong way:**
```
Spawn 5 parallel agents — all touching the same 3 files.
Result: 300,000 tokens wasted on redundant reads + coordination.
```

GitHub issue #13579 documents one case where 5 parallel agents on shared files burned **300K tokens**. The same task done sequentially used **20K tokens**. That's a **93% reduction** by NOT parallelizing.

**The rule from the field:** ([HN 45181577](https://news.ycombinator.com/item?id=45181577))
- Parallel = independent tasks only
- Shared files = sequential
- Don't spawn 40 subagents to "refactor everything" — that's costly overkill

Anthropic's official phrasing: *"Make agents for tasks, not roles."*

> **Takeaway:** Subagents are for keeping verbose research out of your main context — not for brute-forcing parallelism.

---

## Unlock 3 — Skills (Progressive Disclosure)

Anthropic announced Skills on **October 16, 2025** ([Equipping agents for the real world with Agent Skills](https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills)). Simon Willison wrote: *"Claude Skills are awesome, maybe a bigger deal than MCP"* ([simonwillison.net](https://simonwillison.net/2025/Oct/16/claude-skills/)).

A Skill is a folder. That's it.

```
my-skill/
├── SKILL.md           # required — frontmatter + instructions
├── reference.md       # optional — loaded on demand
├── examples/          # optional
└── scripts/helper.py  # optional — executed via bash
```

The frontmatter has just two required fields:
```yaml
---
name: my-skill
description: One sentence describing when to use this skill
---
```

**The killer feature is Progressive Disclosure** — a 3-tier loading model:

1. **Startup**: Only the `name` and `description` go into the system prompt. Cost = a few dozen tokens per skill.
2. **On match**: When Claude decides the skill is relevant, the full `SKILL.md` body loads.
3. **On demand**: Bundled reference files and scripts are read via bash only when needed.

This means you can have **100 skills installed** and pay almost nothing in token cost until they actually fire. Compare this to MCP servers, which inject full tool schemas into context on every call.

Built-in skills you may not know exist: `/batch`, `/claude-api`, `/debug`, `/loop`, `/simplify`. Try them.

### Real famous skills to install today

These three are the ones power users actually run:

| Skill | Author | Why |
|---|---|---|
| **[superpowers](https://github.com/obra/superpowers)** | Jesse Vincent (`obra`) | The #1 Claude Code plugin. ~13 composable skills that force disciplined TDD, brainstorming, planning, and systematic debugging. Released October 2025 alongside Jesse's viral "Superpowers" blog post. |
| **[anthropic/pdf](https://github.com/anthropics/skills)** | Anthropic (official) | Create, edit, fill forms, and extract content from PDFs. Part of the official `document-skills@anthropic-agent-skills` plugin (also includes `docx`, `pptx`, `xlsx`). |
| **[graphify](https://github.com/safishamsi/graphify)** | Safi Shamsi (`safishamsi`) | Turns any folder of code, docs, papers, or images into a queryable knowledge graph. Cuts token usage up to 71.5x per query. ~10.9K stars. |

> **Takeaway:** Skills give you reusable workflows with near-zero token overhead. Don't build everything from scratch — install the famous ones first.

---

## Unlock 4 — Token Discipline (3 Rules)

This is the lever that cuts your bill 40-70% with no quality loss.

### Rule 1 — Use `opusplan`
Set your model to `opusplan`. Opus plans the work, Sonnet writes the code. You get architectural reasoning at Opus quality and implementation at Sonnet cost. ([sabrina.dev](https://www.sabrina.dev/p/6-ways-i-cut-my-claude-token-usage)).

### Rule 2 — Treat `/clear` like punctuation
*"Make `/clear` a punctuation mark in your workflow: finish a task, commit, type `/clear`, then set fresh context."* ([Anthropic docs](https://code.claude.com/docs/en/best-practices))

Anthropic's specific rule: **if you've corrected Claude twice on the same issue, `/clear` and rewrite the prompt**. A clean session with a better prompt almost always beats a long one with accumulated corrections.

### Rule 3 — Use `/compact` at 70% context, not `/clear`
`/clear` nukes everything. `/compact` compresses while preserving intent. A 70K-token conversation compresses to roughly 4K tokens.

You can pass instructions: `/compact "keep API decisions, summarize debugging"`.

**Bonus rule — don't edit CLAUDE.md mid-session.** CLAUDE.md gets prompt-cached with a 90% discount on subsequent messages. Edit it mid-session and you bust the cache. ([claudefa.st](https://claudefa.st/blog/guide/development/usage-optimization)).

> **Takeaway:** Right model + ruthless `/clear` + smart `/compact` = your bill drops without your work slowing down.

---

## Unlock 5 — Hooks (Deterministic Guarantees)

The single best line from Anthropic's hooks documentation:
> *"Unlike CLAUDE.md instructions which are advisory, hooks are deterministic and guarantee the action happens."* ([Anthropic hooks guide](https://code.claude.com/docs/en/hooks-guide))

Putting "run prettier after edits" in CLAUDE.md is a polite request. Putting it in a hook means Claude has no choice — the harness fires the script after every Edit/Write.

**The most useful hook recipes:**

```jsonc
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "npx prettier --write $CLAUDE_FILE_PATH" }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "scripts/block-migrations.sh $CLAUDE_FILE_PATH" }
        ]
      }
    ]
  }
}
```

The 3 most-shared hook setups on Reddit and HN:
1. **PostToolUse + Edit|Write** → auto-format on every save
2. **PreToolUse + Edit** → block writes to `/migrations` or other protected paths
3. **SessionStart** → inject project context on every new session

Hooks turn Claude Code from "smart tool that occasionally listens" into "smart tool that always follows your safety rules."

> **Takeaway:** If a rule MUST happen, put it in a hook, not CLAUDE.md.

---

## The Settings.json + CLAUDE.md Template

Here's the minimal God Mode setup. Drop this in `.claude/settings.json`:

```jsonc
{
  "model": "opusplan",
  "permissions": {
    "allow": [
      "Bash(npm run lint)",
      "Bash(npm test)",
      "Bash(git status)",
      "Bash(git diff)",
      "Read(**)",
      "Grep(**)",
      "Glob(**)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(terraform destroy)",
      "Edit(.env*)",
      "Edit(secrets/*)"
    ]
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "npx prettier --write $CLAUDE_FILE_PATH 2>/dev/null || true" }
        ]
      }
    ]
  }
}
```

For your CLAUDE.md, **keep it under 100 lines**. Boris Tane (Anthropic engineer) keeps his at ~2,500 tokens ([boristane.com](https://boristane.com/blog/how-i-use-claude-code/)). HumanLayer's published guide says under 60 lines is ideal ([humanlayer.dev](https://www.humanlayer.dev/blog/writing-a-good-claude-md)).

**The rule from Anthropic's docs:**
> *"Bloated CLAUDE.md files cause Claude to ignore your actual instructions! If Claude keeps doing something you don't want despite having a rule against it, the file is probably too long."*

Add a line ONLY when Claude makes the exact mistake that line would prevent — never hypothetically.

---

## Summary

| Unlock | What | Why | How |
|---|---|---|---|
| **Plan Mode** | Shift+Tab×2 | Cheap Haiku exploration, no main-context pollution | Anthropic best practices |
| **Subagents** | Parallel fan-out | Verbose research stays out of main context | "Tasks not roles" |
| **Skills** | Folder + SKILL.md | Progressive disclosure = ~zero token cost until used | Skills launch Oct 2025 |
| **Token Discipline** | opusplan + /clear + /compact | 40-70% cost reduction with no quality loss | sabrina.dev guide |
| **Hooks** | settings.json | Deterministic guarantees, not advisory rules | Anthropic hooks docs |

These five together separate the 30% user from the power user. The features are already on your machine. You just have to turn them on.

---

## References

- [Anthropic — Claude Code best practices](https://code.claude.com/docs/en/best-practices)
- [Anthropic — Equipping agents with Agent Skills](https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills)
- [Anthropic — Hooks guide](https://code.claude.com/docs/en/hooks-guide)
- [Anthropic — Subagents in Claude Code](https://claude.com/blog/subagents-in-claude-code)
- [GitHub — claude-code issue #13579 (700K wasted tokens)](https://github.com/anthropics/claude-code/issues/13579)
- [The Register — Anthropic admits quota issues](https://www.theregister.com/2026/03/31/anthropic_claude_code_limits/)
- [Sabrina Ramonov — 6 ways I cut my Claude token usage](https://www.sabrina.dev/p/6-ways-i-cut-my-claude-token-usage)
- [HumanLayer — Writing a good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
- [Boris Tane — How I use Claude Code](https://boristane.com/blog/how-i-use-claude-code/)
- [Simon Willison — Claude Skills are awesome](https://simonwillison.net/2025/Oct/16/claude-skills/)
- [ClaudeFa.st — Usage optimization](https://claudefa.st/blog/guide/development/usage-optimization)

#claudecode #anthropic #ai #softwareengineer #systemdesign #coding #developertools #productivity #aiengineering #llm
