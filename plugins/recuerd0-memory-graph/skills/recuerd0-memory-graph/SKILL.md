---
name: recuerd0-memory-graph
description: >-
  Generate an interactive force-directed "memory graph" artifact from a Recuerd0
  workspace — nodes are memories, edges are the links between them, grouped into
  colored clusters. Use when the user wants to visualize, map, or see how the
  memories in a Recuerd0 workspace connect, asks for a "memory graph / memory map
  / knowledge graph" of a workspace, or says "make a graph of workspace <N>".
---

# Recuerd0 Memory Graph

Turn a Recuerd0 workspace into the same interactive graph artifact you see for
Fragua's workspace 54: a canvas force-layout where every memory is a node, every
memory-link is an edge, memories are grouped into colored clusters, and clicking a
node opens a detail panel with its summary and connections.

The artifact ships as a **self-contained template** at `assets/template.html`
(Recuerd0 green OKLCH tokens, embedded Jura display font, Geist Mono for IDs, the
layered-cards logo, light/dark themes, CSP-safe). You only swap its **data** and
**header** — never touch the CSS or engine.

## Inputs

- **Workspace** — id (e.g. `54`) or name. If the user didn't give one, ask which
  workspace, or list workspaces and let them pick.
- Optional: a **title** override, and any manual **cluster grouping** the user wants.

## Workflow

### 1 — Pull the workspace data

Read the workspace's memories and the links between them. Use whatever Recuerd0
access this project has, in this order of preference:

1. **`recuerd0:remember` agent / `recuerd0` CLI** if present (Fragua's house rule is
   to go through the agent — respect it here too). Ask it for: every memory's
   **id, title, one-line description, type, and tags**, plus the **memory links**
   (source id → target id, and the link's label/type if it has one).
2. **Recuerd0 MCP tools** otherwise — load them with
   `ToolSearch("select:mcp__claude_ai_Recuerd0_ai__list_memories,mcp__claude_ai_Recuerd0_ai__read_memories,mcp__claude_ai_Recuerd0_ai__list_memory_links,mcp__claude_ai_Recuerd0_ai__list_workspaces")`
   then call `list_memories` + `list_memory_links` for the workspace.

> **Memory-id gotcha (Fragua):** `memory list` returns the version-HEAD id, `search`
> returns the BASE id — same memory, two ids. Normalize every id through
> `memory show` (use `data.id`) so nodes and links reference the *same* id, or edges
> will dangle. The template silently drops links whose endpoints aren't in `N`, so a
> mismatch shows up as missing edges.

**CLI cheatsheet** (verified against the `recuerd0` Go CLI):
- `recuerd0 workspace list` → find the id/name (output is JSON; workspace 56 = "Halua").
- `recuerd0 memory list --workspace <id> --page <n>` → **paginated** (~25/page); loop
  while `pagination.has_next` is `True`. Each row has `id, title, category, tags,
  links_count, has_versions` — **but no separate description field**, so the `title` is
  your one-liner. Split it on the first `:` → left = node title, right = summary.
- `recuerd0 memory link list <id> --workspace <id>` → per-memory links (there's no
  bulk link dump). Only query memories whose `links_count > 0`; if every row is `0`,
  the workspace has no links — use the shared-tag fallback in step 3.
- `category` is `discovery|decision|preference|general` — often skewed (Halua was 37/45
  `discovery`), so prefer **tags** for clustering.

Keep summaries to **one sentence** — they render in the detail panel and as hover
context. Trim anything longer.

### 2 — Derive clusters and tiers

**Clusters** (the colored groups). Pick the grouping dimension in this order:
1. If the user named groups, use them.
2. Else if memories carry meaningful **tags**, cluster by the dominant tag.
3. Else cluster by memory **type** (`user` / `feedback` / `project` / `reference`).
4. Else fall back to **link communities** (connected components / who-links-with-whom).

Aim for **4–7 clusters** — merge tiny ones into an "Other" group. Give each a short
human label. Colors come from the template's built-in categorical wheel; you just
assign each cluster one of the palette keys (see schema below).

**Tier** — `canon` (large, labelled, ringed hub) vs `leaf` (small dot). Choose by:
- an explicit "canonical/pinned" flag or a title containing `canonical`, else
- **degree**: memories with links at or above the median → `canon`, the rest `leaf`.

Don't make everything `canon` — the contrast between a few hubs and many leaves is
what makes the graph readable.

### 3 — Get edges (explicit links, or derive them)

Each edge gets one of four types, which control how it's drawn:

| type     | drawn as              | use for |
|----------|-----------------------|---------|
| `flow`   | solid arrow (directed)| sequence / "A feeds B" / depends-on |
| `twin`   | dashed line           | sibling / peer / twin / mirror relationships |
| `ref`    | thin line             | generic "references / see also" (safe default) |
| `rollup` | hair-thin line        | a `leaf` detail belonging to a `canon` parent |

**If the workspace has explicit memory links:** map each link's label/type to the
closest bucket; default to `ref`, promote to `rollup` when exactly one endpoint is a
`leaf`.

**If it has NO links** (common — e.g. ws 56/Halua had 45 memories, zero links; every
`links_count` is `0`), don't ship an edgeless graph. **Derive edges from shared-tag
co-occurrence:**
- Connect two memories that share **≥2 topical tags**, OR **≥1 rare tag** (global tag
  frequency ≤ 3). The rare-tag rule catches niche pairings; the ≥2 rule keeps generic
  tags from over-connecting.
- **Exclude connective/generic tags** from the match (e.g. `gotcha`, `setup`) so one
  ubiquitous tag doesn't create a hairball.
- Type these `ref`, then relabel `leaf`↔`canon` edges as `rollup` for visual hierarchy.
  There's no honest `flow`/`twin` here — associative tag affinity is `ref`.
- A handful of isolated nodes (unique tags, no shared) is fine; they rest in their
  cluster's gravity well.

When links are absent, tier (step 2) must come from the **derived-edge degree**, not
`links_count`.

### 4 — Fill the template

Copy `assets/template.html` to a working file (e.g. the scratchpad), then replace the
marked regions. **Only** edit inside the markers. There are four:
`@INJECT:TITLE` (the `<title>` — set it or the browser tab keeps Fragua's name),
`@INJECT:HEADER`, `@INJECT:SUB`, and `@INJECT:DATA`.

**`<!-- @INJECT:TITLE... -->`** — `<title>…</title>` for the browser tab / gallery.

**`<!-- @INJECT:HEADER... -->` / `<!-- @INJECT:SUB... -->`** — workspace name + one-line intro:
```html
<div class="kicker">Recuerd0 · workspace <ID> · <NAME></div>
<h1>The Memory <span class="em">Graph</span></h1>
```
```html
<p class="sub">How <workspace>'s memories connect — each node a memory, each thread a link between them. Click a node to read it.</p>
```

**`/* @INJECT:DATA... */`** — the three JS structures. Exact shapes:

```js
// key => { label: shown in legend, color: palette var name }
// palette keys available: --c-foundation --c-pipeline --c-infra --c-frontend --c-ops
const CLUSTERS = {
  research: { label: "Research", color: "--c-pipeline" },
  people:   { label: "People & prefs", color: "--c-infra" },
  // …4–7 total
};

// [id, title, clusterKey, tier, summary]
//   id       string, must be unique and match link endpoints
//   tier     "canon" | "leaf"
//   summary  ONE sentence (avoid unescaped " — the strings are double-quoted JS)
const N = [
  ["102","Onboarding research","research","canon","How new users first reach value."],
  ["87","Prefers async standups","people","leaf","Team runs written standups, not calls."],
  // …
];

// [sourceId, targetId, type]   type: "flow" | "twin" | "ref" | "rollup"
const L = [
  ["102","87","ref"],
  // …
];
```

Rules that keep it from breaking:
- Every `L` endpoint id must exist in `N` (else the edge is silently dropped).
- Every `N` `clusterKey` must be a key in `CLUSTERS`.
- The template supports **exactly five palette colors** (`--c-foundation`, `--c-pipeline`,
  `--c-infra`, `--c-frontend`, `--c-ops`). Reuse a color if you have fewer clusters;
  don't exceed five distinct colors (a 6th cluster should reuse one). If you truly need
  more, add matching `--c-<name>` / `--c-<name>-hi` / `--c-<name>-lo` OKLCH triples in
  **all four** `:root` blocks and a `.cl-<name>` class — but prefer staying at five.
- Escape or avoid `"`, `\`, and lone `—`-vs-`-` issues inside summaries.

### 5 — Publish

Publish the working file with the **Artifact** tool (favicon `🗂️` or `🔥`, a one-line
`description`). It's fully self-contained, so it renders under the artifact CSP with no
external fetches. Redeploy the **same file path** to keep the same URL while iterating.

### 6 — Sanity check before handing off

- Node count ≈ memory count; no cluster is empty; ≤ 5 colors in use.
- No dangling edges (every link endpoint resolved) — if edges look missing, it's almost
  always the id-normalization gotcha from step 1.
- Spot-check 2–3 nodes: click opens the panel, summary reads as one clean sentence,
  connections list is populated.

## Notes

- The template's engine (force sim, pan/zoom/drag, legend toggles, theme switch,
  detail panel, `dirty`-flag repaint) is done — don't reimplement it.
- Keep the flat node look; a lit-sphere variant was tried and rejected.
- This skill lives in-repo; copy it to `~/.claude/skills/` to use it from any project.
