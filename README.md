# monforje plugins

A personal Claude Code plugin marketplace. Each plugin under `plugins/` is
installed and versioned on its own.

| Plugin | What it is | Status |
| --- | --- | --- |
| [`qa-suite`](plugins/qa-suite) | QA orchestrator and nine specialised verification sub-agents | `qa-unit` and `qa-integration` implemented, seven stubs |
| [`devops`](plugins/devops) | Docker, Compose, Traefik and Taskfile — three agents, five skills, three commands | implemented |

## Install

```bash
claude plugin marketplace add monforje/claude-plugins
claude plugin install qa-suite@monforje
```

Components load at session start, so restart the session after installing.
Verify with `claude plugin details <name>`, or type `@qa-suite:` in a session and
let autocomplete list the agents. (`plugin details` reports `Agents (0)` — it
does not count agents declared as explicit paths. They do load.)

## Develop

An install is a cached copy, so edits to a clone are not picked up. Link the
plugin you are working on instead:

```bash
ln -s /path/to/claude-plugins/plugins/qa-suite ~/.claude/skills/qa-suite
```

The symlink points at the **plugin** directory — the one containing
`.claude-plugin/plugin.json` — not at the repository root. The root holds
`marketplace.json` and is not itself a plugin, so linking it loads nothing.
Link only the plugins you want live in every session; a plugin still under
construction is better left unlinked than filling the skill list with
placeholders.

For a one-off session: `claude --plugin-dir /path/to/claude-plugins/plugins/qa-suite`.

## Layout

```
.claude-plugin/marketplace.json   one entry per plugin, source: ./plugins/<name>
plugins/<name>/
├── .claude-plugin/plugin.json    agents and skills declared as explicit paths
├── agents/
├── skills/
└── README.md
```

Adding a plugin means a directory under `plugins/` with its own
`.claude-plugin/plugin.json`, plus an entry in `marketplace.json` pointing at it.
Nothing else is wired up globally.
