# skills

Personal [Agent Plugins](https://agent-plugins.org/specification) packages — portable, client-neutral
skill bundles.

Every directory under `plugins/` is a **complete, independent plugin**: its own `plugin.json` at its own
root, its own `skills/`, installed on its own. The spec is explicit that *"a plugin is a directory rooted
at a single filesystem location"* — it defines no monorepo, no registry, and no multi-plugin index, and
leaves discovery and distribution entirely to clients. So this repo does not pretend to be one unit.
It is five plugins that happen to share a git remote, each installed by pointing an installer at its
directory.

There is no marketplace file here, and none is needed.

## Plugins

### `claude-md-auditor`

Audits, trims, and maintains `CLAUDE.md` / `AGENTS.md` — the instruction files an agent re-reads on
every single request. The premise: those files sit at the apex of a leverage cascade, so the metric is
**behavioral signal per token**, not completeness. A line that describes the project without changing
what the agent *does* is a line that dilutes every plan and every diff that follows it.

Triggers on "review my CLAUDE.md", "my context file is bloated", "what should I cut", or pasting an
instruction file and asking what to improve. Returns a concrete keep/cut/rewrite verdict per section,
not prose.

### `recuerd0-canonical-memory`

Collapses many loose [Recuerd0](https://recuerd0.ai) memories about one process or area into a single
canonical, single-source-of-truth memory — **validated against live source code** by divide-and-conquer
subagents, so the result documents what the code actually does rather than what the old memories
claimed. Consolidation only: nothing is deleted.

Triggers on "create a canonical memory for X", "consolidate the X memories", "I want one source of
truth for how X works", or pointing at a workspace and asking which areas are worth canonicalizing.

A gotcha it encodes, because it silently corrupts edits otherwise: a versioned memory has **two ids** —
`memory list` returns the version-HEAD id, `search` returns the stable base id. Normalize through
`memory show` before any `version create` or `delete`.

### `recuerd0-memory-graph`

Renders a Recuerd0 workspace as an interactive force-directed graph artifact: memories are nodes, the
links between them are edges, grouped into colored clusters. Useful for seeing which areas are densely
cross-linked and which memories are orphans — the ones nothing points at are usually the ones worth
canonicalizing or retiring.

Triggers on "make a graph of workspace 54", "show me how these memories connect", or "memory map /
knowledge graph".

Ships its own `assets/template.html`, so it resolves within its plugin root and needs no network.

Both `recuerd0-*` plugins assume a working Recuerd0 CLI or MCP server is already available — neither
carries a transport of its own. The complementary `recuerd0` plugin (the `remember` skill, for
capturing memories in the first place) lives in
[`maquina-app/rails-claude-code`](https://github.com/maquina-app/rails-claude-code).

### `rust-simplifier`

Simplifies and refines Rust code toward **plain, data-oriented, idiomatic Rust** while preserving exact
behavior — the Rust counterpart of `rails-simplifier` (also in
[`maquina-app/rails-claude-code`](https://github.com/maquina-app/rails-claude-code)): "plain Rust is plenty". Concrete types, enums,
functions, and modules; abstraction only once the code already needs it. It swaps `.unwrap()` in library
code for `?` and a typed error, borrow-checker `.clone()`s for restructured borrows, single-impl traits
for concrete types, `Box<dyn Trait>` over a closed set for an `enum`, flag soup for enum state,
`Arc<Mutex<T>>` for a single owner or channels, and `mockall` over your own code for real temp files and
in-memory SQLite.

Triggers on "simplify", "clean up", "make more idiomatic", or reviewing a Rust diff, and applies
proactively after writing Rust. Verifies with `cargo fmt`, `cargo clippy -D warnings`, and `cargo test`.

### `swift-simplifier`

Simplifies and refines Swift and SwiftUI code toward **plain, value-oriented, main-actor-first Swift**
while preserving exact behavior: "plain SwiftUI is plenty". One `@Observable` model per feature instead
of a ViewModel per view, `async/await` over one-shot Combine, `.task(id:)` over `Task { }` in `onAppear`,
main-actor isolation over `DispatchQueue` in async code, restructured ownership over actors or
`@unchecked Sendable` that only silence the compiler, an `enum LoadState` over boolean flags, and
concrete types with Swift Testing over protocols created for mocking.

Triggers the same way as its Rust sibling and applies proactively after writing Swift. Verifies with
`swift build && swift test` (or `xcodebuild test`) and `swift format lint`.

Both simplifiers defer to project rules in `CLAUDE.md` or `sdd/standards/` over their own defaults. Each
keeps `SKILL.md` to principles, a gated process and an index from code shape to target; the reasoning,
exceptions and examples for each concept live in one section of `references/patterns.md`, read only for
the patterns a change actually touches.

## Layout

```
plugins/
├── claude-md-auditor/
│   ├── plugin.json
│   └── skills/
│       └── claude-md-auditor/
│           └── SKILL.md
├── recuerd0-canonical-memory/
│   ├── plugin.json
│   └── skills/
│       └── recuerd0-canonical-memory/
│           └── SKILL.md
├── recuerd0-memory-graph/
│   ├── plugin.json
│   └── skills/
│       └── recuerd0-memory-graph/
│           ├── SKILL.md
│           └── assets/
│               └── template.html
├── rust-simplifier/
│   ├── plugin.json
│   └── skills/
│       └── simplify-rust/
│           ├── SKILL.md
│           └── references/
│               └── patterns.md
└── swift-simplifier/
    ├── plugin.json
    └── skills/
        └── simplify-swift/
            ├── SKILL.md
            └── references/
                └── patterns.md
```

A client loads `plugin.json`, walks `skills/` for directories holding a `SKILL.md`, and validates each
against the [Agent Skills specification](https://agentskills.io/specification). Every path a plugin
references resolves within its own plugin root. The layout above is flat because five skills need no
grouping — a client that walks the tree (equipr does, since 0.3.1) would equally accept
`skills/<category>/<name>/SKILL.md`.

## Installing

The intended installer is [`equipr`](https://github.com/maquina-app/equipr), which installs skills,
commands, and MCP servers into Claude Code, Codex, OpenCode, and Pi without registering as a native
plugin. Each plugin is added as its own source, by its directory:

```sh
equipr add https://github.com/mariochavez/skills#plugins/claude-md-auditor
equipr add https://github.com/mariochavez/skills#plugins/recuerd0-canonical-memory
equipr add https://github.com/mariochavez/skills#plugins/recuerd0-memory-graph
equipr add https://github.com/mariochavez/skills#plugins/rust-simplifier
equipr add https://github.com/mariochavez/skills#plugins/swift-simplifier
```

Each `add` reports what it resolved, and `equipr list` shows it again later:

```console
$ equipr list
claude-md-auditor            plugin
  claude-md-auditor          1.0.0  1 skill
recuerd0-canonical-memory    plugin
  recuerd0-canonical-memory  1.0.0  1 skill
recuerd0-memory-graph        plugin
  recuerd0-memory-graph      1.0.0  1 skill
rust-simplifier              plugin
  rust-simplifier            0.1.0  1 skill
swift-simplifier             plugin
  swift-simplifier           0.1.0  1 skill
```

Then install. Without `--yes`, equipr prompts for which agents to install into,
pre-checking every one it found — space toggles, enter confirms:

```sh
equipr install claude-md-auditor/claude-md-auditor
equipr install recuerd0-memory-graph/recuerd0-memory-graph -a claude-code
equipr install rust-simplifier/rust-simplifier -a claude-code -a pi

# non-interactive: every agent equipr finds
equipr install claude-md-auditor/claude-md-auditor --yes
```

The argument is `<source>/<plugin>`, and both are the directory name under `plugins/`, not the skill
name — `rust-simplifier/rust-simplifier` installs the `simplify-rust` skill.

Use **equipr 0.3.1 or newer**. Earlier versions left the prompt's options
unchecked, so pressing enter selected nothing and installed nothing while
reporting that the plugin had no installable components.

Each plugin is its own source because the AP spec roots a plugin at a directory and defines no
multi-plugin index — the `#plugins/<name>` fragment is equipr's addressing, and nothing in this repo
depends on it. A `/tree/<ref>/<path>` URL copied from the browser works too.

Or install by hand — copy or symlink a skill directory into the agent's skills directory:

```sh
ln -s "$PWD/plugins/claude-md-auditor/skills/claude-md-auditor" \
      ~/.claude/skills/claude-md-auditor
ln -s "$PWD/plugins/recuerd0-canonical-memory/skills/recuerd0-canonical-memory" \
      ~/.claude/skills/recuerd0-canonical-memory
ln -s "$PWD/plugins/recuerd0-memory-graph/skills/recuerd0-memory-graph" \
      ~/.claude/skills/recuerd0-memory-graph
ln -s "$PWD/plugins/rust-simplifier/skills/simplify-rust" \
      ~/.claude/skills/simplify-rust
ln -s "$PWD/plugins/swift-simplifier/skills/simplify-swift" \
      ~/.claude/skills/simplify-swift
```

Symlinking keeps this repo the single source of truth; copying pins a version.

## Adding a plugin

1. `mkdir -p plugins/<name>/skills/<skill-name>`
2. Write `plugins/<name>/plugin.json` with `$schema` and `name` — the two required fields. The name is
   1–64 chars of `a-z0-9.-`, alphanumeric at both ends, no `--` or `..`.
3. Write `skills/<skill-name>/SKILL.md` with `name` and `description` frontmatter. The `name` must
   match its directory.
4. Validate it against the Agent Skills spec with the reference validator:
   `pip install skills-ref && agentskills validate plugins/<name>/skills/<skill-name>`.
5. Add it to **Plugins** above.

A model-invoked skill's `description` is loaded on every request, so keep it to what the skill does and
when to use it. Keep `SKILL.md` to the steps and an index; put reference material in `references/`,
one section per concept, read only when a step needs it.

Keep one plugin per installable unit. Two skills belong in one plugin only if installing one without
the other makes no sense.

## License

MIT — see [LICENSE](LICENSE).
