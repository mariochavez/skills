---
name: claude-md-auditor
description: Audit, optimize, and maintain CLAUDE.md (or AGENTS.md) instruction files for AI coding agents. Use this skill whenever the user wants to review, trim, or improve a CLAUDE.md or AGENTS.md file, asks whether their instruction file is too long, wants to reduce token usage in their AI context, or says something like "review my CLAUDE.md", "optimize my agent instructions", "my context file is getting bloated", or "help me clean up CLAUDE.md". Also triggers when the user pastes a CLAUDE.md and asks what to cut or improve.
---

# CLAUDE.md Auditor

This skill helps review and optimize CLAUDE.md (or AGENTS.md) files — the instruction files that AI coding agents read at the start of every session. The goal is to maximize **behavioral signal per token**: every line should change how the agent acts, not just describe the project.

## Why this matters

CLAUDE.md content is injected between the system prompt and the first user message on **every single request**. Bad lines don't sit idle — they dilute the agent's attention across everything it produces. The file sits at the apex of a leverage cascade: a bad instruction multiplies through every plan, every implementation, and every line of code.

Key research findings to keep in mind:
- Context files can **reduce** agent success rates while increasing token costs by 20%+ when they're bloated or generic
- LLMs can follow roughly 150–200 instructions reliably; Claude Code's system prompt already uses ~50
- Claude Code wraps CLAUDE.md content with a note that it "may or may not be relevant" — bloated files get actively skimmed
- The **Maximum Effective Context Window** for instructions is far smaller than advertised limits

## Audit Targets

### Size is a symptom, not the verdict — judge the class mix

There is no line count that makes a file bad. A 7,000-token file of dated failure
contracts, each tied to a real incident, is correct and must not be gutted; a
2,000-token file of tech-stack tables and architecture tours is bloat. **Measure
first, then classify (Step 2), and let the mix decide.** Report size as context,
never as the finding.

- **Token estimate:** ~1 token per 4 characters. Report it for every loaded file.
- **The one mechanical threshold:** Claude Code warns when a single memory file
  exceeds ~5% of the model's context window in characters, floor ~40,000 chars.
  State which files trip it before and after your proposed cuts.
- **Prose paragraphs:** a block of 3+ lines of flowing prose is a candidate for
  compression — *unless* it is the "why" behind a rule. Narrative that explains
  why a bug happened is what stops someone re-introducing it; a rule stripped of
  its reason gets overruled by the next person who thinks they know better.
- **Red flag that beats any line count:** the same guidance appearing in two
  loaded files, or a section that describes rather than directs.

### The core question for every section
**"Does this change what the agent does, or does it describe what the project is?"**

- Behavioral → keep (changes actions, prevents bugs, encodes past failures)
- Descriptive → cut or move to a referenced doc

---

## Audit Process

### Step 0: Audit the whole cascade, never one file

A single-file audit misses the two highest-value findings. Before measuring, list
everything that loads and note *when* each loads:

| File | Loads |
|------|-------|
| `~/.claude/CLAUDE.md` | **every session in every project** |
| `CLAUDE.local.md` | every session in this project (gitignored) |
| `CLAUDE.md` / `.claude/CLAUDE.md` | every session in this project |
| `<subdir>/CLAUDE.md`, `.claude/rules/*.md` with `paths:` | only when working on matching files |
| `.claude/skills/*/SKILL.md`, `~/.claude/skills/*/SKILL.md` | on invocation (description stays resident) |

Then check three things the per-file pass cannot see:

1. **Scope fit.** Is content in an always-everywhere file actually specific to one
   repo? A project guide sitting in `~/.claude/CLAUDE.md` taxes every unrelated
   project. Move it to that repo — and if the repo has no CLAUDE.md, that is the
   finding.
2. **Duplication against lazy skills.** Guidance that already lives in a skill may
   not need to be resident. Weigh it against the safety-net rule below.
3. **Contradictions between loaded files.** These are the most damaging defect an
   audit can find, and they are invisible file-by-file: a skill saying
   `zsh -l -c 'bin/ci'` while CLAUDE.md says that shell is broken, or a skill
   saying "wait for CI" while the project disabled CI. Quote both sides. The more
   specific and more recently maintained file normally wins; confirm with the user
   rather than resolving it yourself.

### Step 1: Measure
Count total lines and estimate token usage. Report both upfront.

### Step 2: Classify every section

Go through each section and classify it as one of:

| Type | Description | Action |
|------|-------------|--------|
| 🟢 **Critical behavioral** | Rules that prevent bugs/security issues, non-obvious gotchas, past failure lessons | Keep, protect |
| 🟡 **Useful but verbose** | Correct content but too wordy, prose where a table would do | Compress |
| 🔴 **Descriptive bloat** | Project description, tech stack list, philosophy, things inferrable from code | Cut or move to referenced doc |
| 🔵 **Human docs** | Readable explanations, "why OKLCH is good", onboarding text | Cut (belongs in README or docs/) |
| 🟠 **Inferrable** | Patterns the agent can discover from the codebase itself | Cut |

### Step 3: Produce the audit report

Use this structure:

```
## CLAUDE.md Audit Report

**Current:** X lines / ~Y tokens
**Target:** Z lines / ~W tokens  
**Savings opportunity:** N lines

### 🔴 Cut these sections (saves ~N lines)
[section name] — reason
[section name] — reason

### 🟡 Compress these sections (saves ~N lines)
[section name] — current vs. compressed version side-by-side

### 🟢 Keep these sections
[section name] — why it earns its place

### Recommended rewrite
[full rewritten file if feasible, or most impactful changes if very long]
```

### Step 4: Offer the rewrite

Always offer to produce the full revised file. For files under 200 lines, produce it. For longer files, produce the most impactful sections.

---

## What earns its place in CLAUDE.md

These categories justify tokens:

**Non-obvious constraints that cause bugs if violated**
- Money as integer cents (not floats) — the agent will use floats otherwise
- Always scope queries to `Current.user` — security issue if missed
- `status: :see_other` required for Turbo Morph actions — non-obvious Rails behavior

**Project-specific tool setup with non-obvious gotchas**
- MCP server: must call `switch_project` first or it silently uses wrong project
- Tailwind file location being non-standard (`tailwind/` not `stylesheets/`)
- Always wrap `execute_ruby` output in `puts` or get empty response

**Workflow gates that prevent wasted effort**
- "Plan before coding — present plan, wait for approval"
- "Ask which branch at session start"
- "Run `bin/ci` before committing"

**Anti-patterns derived from real failures**
- These are the most valuable lines in the file — each one represents a bug that happened
- Never remove these without understanding the original failure

**Cross-tool references via progressive disclosure**
- `docs/ARCHITECTURE.md` — "read when adding features"
- This pattern lets CLAUDE.md stay lean while preserving deep knowledge
- But see "Index rows" below — a pointer table only stays cheap if the rows stay short

**Safety rules stay resident even when they're duplicated**
- Never migrate a "never do X" prohibition, a destructive-command gate, or a
  workflow gate into a lazy skill. A rule that isn't loaded when it matters is
  worse than no rule, because the file implies coverage it doesn't have.
- Duplication between CLAUDE.md and a lazy skill is therefore sometimes *correct*.
  Flag it as a deliberate safety net, not as bloat — and check the two copies say
  the same thing (Step 0.3).

---

## What gets cut

**Project description / philosophy**
```
# BEFORE (cut this)
Resto is a personal finance app based on the Kakeibo method. "Resto" means 
"remainder" in Spanish — what's left after expenses. Philosophy: Reflection 
over prediction. No complex categories, no forecasting anxiety.

# AFTER (one line max, or nothing)
Resto — Rails 8.1 personal finance app (Kakeibo method). Bilingual (es/en).
```

**Standard tech stack listings**
The agent reads the Gemfile. Don't list Rails, Hotwire, Tailwind, Chart.js, SQLite — the agent will find them. Only list **non-obvious** stack choices:
```
# BEFORE (too much)
| Framework | Rails 8.1 |
| Frontend | Hotwire (Turbo + Stimulus) |
| Styling | Tailwind CSS 4 |
| Database | SQLite + Solid Queue/Cache/Cable |
| Auth | Passwordless (email verification codes) |
| Charts | Chart.js |
| Deploy | Kamal 2 |

# AFTER (keep only non-obvious)
- Auth: Passwordless email codes (no passwords, no Devise)
- DB: SQLite + Solid Queue/Cache/Cable (not Redis/Sidekiq)
- Money: integer cents only (see Critical Rules)
```

**Conceptual explanations inside instructions**
```
# BEFORE
OKLCH format: oklch(lightness chroma hue) — perceptually uniform, better 
for palettes. Use generated utilities: bg-resto-400, text-success

# AFTER
Colors: OKLCH format in @theme → use semantic utilities (bg-resto-400, text-success)
```

**Patterns the agent gets right from context**
If you've never seen the agent make a mistake with Rails strong params syntax, don't document correct strong params syntax. Trust the model's training; reserve CLAUDE.md for what it gets *wrong* in *your* codebase.

**Index rows that grew into summaries** (the most common late-stage bloat)

An index exists to *route*: it must carry enough to pick the right target and not
one word more. The moment a row is long enough to **answer** the question, it has
stopped being a pointer and become a second, drifting copy of its target — paid
for on every request, while the real document goes unread.

Budget: `id or path` + `label` + a hook of **≤15 words**. One line per row.

```
# BEFORE (one row, 780 chars — and there are 22 of them)
| **Execution canonical — the process (remote-only, sweep → build worktree→branch→PR
→ iterate → ship), THE reference for build/execution work; the ONE `Executable` stack
shared by FeatureSpec + Issue (differ by FK), multi-signal turns + task checklist,
server-side reparse, the remote two-plane (claim→run→stream adaptive poll,
lease/heartbeat), rich build recovery** | **1310** |

# AFTER (one row, 78 chars)
| Execution — the shared `Executable` build stack (worktree → branch → PR) | 1310 |
```

State the trade-off when you propose this: the long rows sometimes let the agent
answer without opening the target. After compression it must actually go read it —
which is the intended flow if the file says "search memory/docs first", and a real
behavior change if it doesn't.

**Long tool reference tables** (move to referenced doc)
Extensive tool invocation examples belong in `docs/DEVELOPMENT.md`, not the root instruction file. Keep only the critical gotcha and a pointer:
```
# BEFORE (20 lines of tool examples)
| Project overview | execute_tool tool_name: "project_info" |
| Analyze models | execute_tool tool_name: "analyze_models"... |
...

# AFTER (3 lines)
### Rails MCP Server
Always call switch_project first: railsMcpServer:switch_project project_name: "X"
Full tool reference → docs/DEVELOPMENT.md
```

---

## Mac's Rails project patterns (apply contextual knowledge)

Mac's projects share a common structure. When auditing his CLAUDE.md files, apply these project-specific heuristics:

**Standard stack (inferrable, don't document)**
- Rails 8.1, Hotwire, Tailwind CSS 4, SQLite, Solid trio, Kamal 2 — agent discovers from the Gemfile
- Standard maquina_components UI patterns — owned by the `maquina-ui-standards`
  agent; Ruby/Rails structure by `rails-simplifier`, Stimulus by `better-stimulus`.
  If a rule belongs to one of those agents, cut it from CLAUDE.md and point at the agent.

**Toolchain (non-obvious, keep — and keep consistent across files)**
- Commands run through **mise** (`mise exec -- bin/ci`); a `zsh -l -c` wrapper falls
  back to system Ruby and breaks bundler. Any file still saying `zsh -l -c` is a
  contradiction to fix, not a style difference.
- Where CI is local-only (`bin/ci` + `gh signoff`, GitHub Actions off), "wait for CI"
  in any loaded file is actively wrong — it makes the agent block forever.
- No AI attribution in commits — safety-class rule, always resident, never migrated.

**Non-obvious, worth keeping**
- Any passwordless auth setup (not Devise — agent will assume Devise)
- Any non-standard file locations (e.g., `tailwind/` vs `stylesheets/`)
- Money as integer cents — always worth keeping, always causes float bugs
- `Current.user` scoping — always keep, security-critical
- MCP `switch_project` requirement — keep, causes silent failures if missed
- I18n requirement when app is bilingual — keep

**Fizzy / Recuerd0 tool integrations**
- Keep tool invocation shortcuts, but compress the examples table into a pointer to docs
- The "workspace is pre-configured" note is worth 1 line — keep

**Session workflow**
- "Ask which branch at session start" + the development cycle are high-value behavioral instructions — keep, but compress prose into numbered list

**Progressive disclosure index**
- The `docs/` and memory-id tables are the right pattern — keep them, but enforce the
  ≤15-word row budget. These tables are exactly where bloat accumulates unnoticed,
  because every individual row addition looks reasonable.

---

## Compression patterns

Convert prose → table: multi-sentence descriptions become one row
Convert table → one-liner: 2-column tables with obvious values become `key: value` inline
Remove explanatory columns: "Why" columns in tables are for humans; cut if it's obvious
Merge related rules: Two related anti-patterns can share a row

**Aggressive one-liner compression:**
```
# BEFORE (4 lines)
### Turbo Preference
1. Default: Full page + Turbo Morph
2. Frames: Only for inline edit, modals  
3. Streams: Multi-element updates

# AFTER (1 line)
Turbo: default to Morph full-page; Frames only for inline edits/modals; Streams for multi-element updates.
```

---

## Output format

When producing the revised file:
1. Show the audit report first (measurements, what's cut and why)
2. Show the full revised file in a code block
3. End with: `Before: X lines / After: Y lines / Savings: Z%`

If the user just wants the revised file without explanation, provide it directly.
