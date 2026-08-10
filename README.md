# skills

Personal [Agent Plugins](https://agent-plugins.org/specification) packages — portable, client-neutral
skill bundles. This is a multi-plugin repository: each directory under `plugins/` is a self-contained
plugin rooted at its own `plugin.json`.

There is no marketplace file here. The repo targets the Agent Plugins 1.0.0 package format only;
distribution and installation are the client's business.

## Plugins

| Plugin | Skills | What it does |
| --- | --- | --- |
| [`claude-md-auditor`](plugins/claude-md-auditor) | `claude-md-auditor` | Audits and trims `CLAUDE.md` / `AGENTS.md` for behavioral signal per token. |
| [`recuerd0-knowledge`](plugins/recuerd0-knowledge) | `recuerd0-canonical-memory`, `recuerd0-memory-graph` | Consolidates loose [Recuerd0](https://recuerd0.ai) memories into canonical ones, and renders a workspace as an interactive graph. |

`recuerd0-knowledge` assumes the Recuerd0 CLI or MCP server is already available — it ships no transport
of its own. The complementary `recuerd0` plugin (the `remember` skill) lives in
[`maquina-app/rails-claude-code`](https://github.com/maquina-app/rails-claude-code).

## Layout

```
plugins/
├── claude-md-auditor/
│   ├── plugin.json
│   └── skills/
│       └── claude-md-auditor/
│           └── SKILL.md
└── recuerd0-knowledge/
    ├── plugin.json
    └── skills/
        ├── recuerd0-canonical-memory/
        │   └── SKILL.md
        └── recuerd0-memory-graph/
            ├── SKILL.md
            └── assets/
                └── template.html
```

A client loads `plugin.json`, discovers the immediate children of `skills/`, and validates each
`SKILL.md` against the [Agent Skills specification](https://agentskills.io/specification). Every path a
plugin references resolves within its own plugin root.

## Adding a plugin

1. `mkdir -p plugins/<name>/skills/<skill-name>`
2. Write `plugins/<name>/plugin.json` with `$schema` and `name` (required). The name must be 1–64
   chars of `a-z0-9.-`, starting and ending alphanumeric, with no `--` or `..`.
3. Write `skills/<skill-name>/SKILL.md` with `name` and `description` frontmatter. The `name` must
   match its directory.
4. Add a row to the table above.

## License

MIT — see [LICENSE](LICENSE).
