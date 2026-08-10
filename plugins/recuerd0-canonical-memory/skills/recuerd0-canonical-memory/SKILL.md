---
name: recuerd0-canonical-memory
description: >-
  Consolidate many loose Recuerd0 memories about one process/area into ONE canonical
  single-source-of-truth memory, validated against live source code, using
  divide-and-conquer subagents. Use when the user asks to "create a canonical memory
  for X", "make one memory from all the X memories", "consolidate the X memories",
  wants a single source of truth for how a process works, or points at a workspace and
  asks which areas to canonicalize. No memories are deleted — this is consolidation.
---

# Recuerd0 Canonical Memory

Collapse a sprawl of per-PR / per-decision memories about one area into a single
**canonical** memory that becomes THE reference for that area — the way ws 54 (Fragua
Engineering) did it for UI (#1250), Foundation (#439), Product Brief (#1303), Research
(#1304), Technical Guide (#1306), and the rest of the wave. The canonical is
authored fresh from the source code, **folds in** what the loose memories got right,
**corrects** what drifted, and **links back** to every source (nothing is deleted).

## Inputs & modes

- **Workspace** — id or name (Fragua default: `54` / "Fragua Engineering").
- **Topic / area** — e.g. "Technical Guide process", "the notification system".

**Mode A — topic given** (the common case): consolidate that one area now.
**Mode B — workspace only**: survey the workspace, cluster the loose memories by area,
and propose 3–6 candidate canonical topics (with rough member counts) for the user to
pick from. Then run Mode A on their choice.

Confirm the **workspace** and the **topic** before spawning subagents, and surface any
constraints the user stated (see "Honor stated constraints" below) — they change what
the canonical says.

## Hard constraints — true of every canonical

1. **NO DELETIONS.** Consolidation only. Every source memory is RETAINED and linked
   from the canonical via `[[id]]` / `memory link add`. Never `memory delete`.
2. **VALIDATE against live source code.** The loose memories are often stale. Read the
   authoritative source, flag drift, and let the canonical carry the *corrected* truth.
   When a memory disagrees with the code, the code wins (and note the correction).
3. **Honor stated constraints — especially deprecations.** If the user says a path is
   deprecated ("the local route is deprecated, remote-only"), **omit it** from the
   canonical except as a one-line "deleted / deprecated — don't use." In Fragua the
   standing rule is **REMOTE-ONLY**: the in-process execution plane
   (`ClaudeAgentRunner`, `ProcessRunner`, `*TurnRunner`, the `runner:` test seam) is
   gone — mention it only as removed.
4. **One area, disambiguated.** Exclude content artifacts that merely share a name
   (e.g. Fragua's own "Market Research Report" doc vs the Research *process*).

## The process

### 1 — Gather candidates (multi-query search)

Search the workspace several ways — jargon, intent, synonyms, class/file names — and
union the hits. Use the `recuerd0` CLI (respect the house rule to prefer the
`recuerd0:remember` agent where it exists):

```
recuerd0 search "<phrasing>" --workspace <id>
recuerd0 memory list --workspace <id> --page <n>     # sweep for anything search missed
```

> **Memory-id gotcha:** `memory list` returns the **version-HEAD** id; `search` returns
> the **BASE** id — same memory, two ids. Normalize every id through
> `recuerd0 memory show <id>` (read `data.id`) before you catalog, link, or version, or
> you'll operate on the wrong row. (Fragua's UI canonical is base `1250` / head `1298`.)

Also pull the **sibling canonicals** already in the table (see step 6) — you'll link
them and may defer shared machinery to them (see "Framing trick").

### 2 — Read the authoritative source for the spine

Before trusting any memory, read the code that defines the area: the model + its state
enum, key concern/macros, jobs, any parser, the controllers, the broadcastable concern,
and the agent **prompt file** if it's an agent process. This spine is the skeleton the
canonical is written around.

### 3 — Set up a scratchpad catalog

Create `scratchpad/<area>-consolidation/` with:
- **`CATALOG.md`** — the validated spine (from step 2) + the hard rules + the
  area-specific **DELTAS vs the sibling instance** + the drop-file schema + the
  candidate ids split into **N batches** + a `drops/` dir.
- **`drops/`** — one file per memory, written by the subagents.

### 4 — Divide and conquer: dispatch parallel subagents

Spawn **~3 parallel `general-purpose` subagents** (scale to candidate count), each
owning one batch of ids. Each subagent, per id: `recuerd0 memory show <id>`, validate
its claims against the live code, and write `drops/<id>.md`:

```markdown
# <id>  (base <base_id> / head <head_id>)
verdict: KEEP-SOURCE | OUT-OF-SCOPE | SUPERSEDED
relevance: <one line — why it does/doesn't belong in this canonical>
accuracy: still-accurate | DRIFTED — <what changed vs code>
phases_touched: [<phase/section tags this memory informs>]
deprecated_to_omit: <local-route / in-process bits to drop, or "none">
area_specific_facts: <facts unique to this area, not shared machinery>
---
<paste-ready, phase-tagged facts to fold into the canonical>
```

The subagents are **authors, not just auditors** — they return paste-ready prose, and
you integrate. (See the parent project's "subagents as authors" preference.)

### 5 — Pre-compose from the spine, then fold in

While the subagents run, draft the canonical from the step-2 spine, mirroring the
**structure of an existing sibling canonical** (same phase headings / rhythm — the
reference instance is your template). When drops land, fold in their corrected facts,
resolve conflicts in favor of the code, and keep it tight.

**Framing trick — don't re-derive shared machinery.** If the area is a sibling of an
already-canonical one (Research ⇄ Product Brief share `Interviewable`; Issue ⇄
FeatureSpec share `Executable`), state only the **DELTAS** and link the sibling
canonical for everything shared. This is what keeps canonicals non-duplicative.

### 6 — Publish, link, point

- **Create/version:** if a canonical draft for this area already exists,
  `recuerd0 memory version create <id> …`; else `recuerd0 memory create --workspace <id>
  --category decision …`. Title it so it's obviously THE reference (e.g. "<Area>
  canonical — the process (remote-only), THE reference for <Area> work").
- **Link every validated source + the sibling canonicals:**
  `recuerd0 memory link add <new_id> --to <source_id>` (batch — typically ~25–30 links).
- **CLAUDE.md pointer row:** add a row to the workspace's always-know table in
  `.claude/CLAUDE.md` so the canonical is discoverable next session.

### 7 — Ship it (Fragua DoD)

Branch `fragua/<area>-canonical-pointer` → commit the CLAUDE.md change **with NO AI
attribution** (keep the `Claude-Session:` trailer) → push → `mise exec -- bin/ci` until
green (ends in `gh signoff`) → `gh pr create` → `gh pr merge --merge --delete-branch`.
The Recuerd0 memory itself is created via the CLI, not git.

## Definition of done

- One canonical memory exists (or a new version), titled as THE reference, remote-only,
  code-validated, structured like its sibling canonicals.
- Every source memory is **retained** and **linked**; sibling canonicals linked too.
- CLAUDE.md pointer row added; PR merged.
- A short note on what **drifted** vs the old memories (so the user knows what changed).

## Notes

- Match effort to sprawl: a 6-memory area doesn't need 3 subagents; a 30-memory one does.
- Don't let the canonical balloon — link outward instead of restating shared machinery.
- This skill lives in-repo; copy it to `~/.claude/skills/` to use it from any project.
- Related: the `recuerd0-memory-graph` skill visualizes a workspace; run it after a
  consolidation to see the new canonical as a hub.
