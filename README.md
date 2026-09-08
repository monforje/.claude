# qa-suite

A QA orchestrator plus specialised verification sub-agents, packaged as a
Claude Code plugin.

| Component | Location | Name in a session |
| --- | --- | --- |
| Orchestrator | `agents/qa.md` | `qa-suite:qa` |
| Sub-agents | `agents/qa/*.md` | `qa-suite:qa-<name>` |
| Skills | `skills/qa/*/SKILL.md` | `qa-suite:<skill-name>` |

## Install

```bash
claude plugin marketplace add monforje/qa-suite
claude plugin install qa-suite@monforje
```

Components load at session start, so restart the session after installing.
Verify with `claude plugin details qa-suite`, or type `@qa-suite:` in a session
and let autocomplete list the agents. (`plugin details` reports `Agents (0)` —
it does not count agents declared as explicit paths. They do load.)

## Develop

An install is a cached copy, so edits to a clone are not picked up. To work on
the plugin, link the checkout instead:

```bash
ln -s /path/to/qa-suite ~/.claude/skills/qa-suite
```

A directory under `~/.claude/skills/` that contains `.claude-plugin/plugin.json`
auto-loads as a plugin, so every new session sees the current files with no
reinstall step. For a one-off session: `claude --plugin-dir /path/to/qa-suite`.

## Adding a component

Everything is declared in `.claude-plugin/plugin.json`, and that declaration is
what makes the nested layout work:

- **A skill** must be listed under `skills` as a directory path. Skill
  auto-discovery only looks one level deep (`skills/<name>/SKILL.md`), so a
  skill at `skills/qa/<name>/` is invisible unless it is declared.
- **An agent** is auto-discovered recursively, but then the subdirectory lands in
  its name (`qa-suite:qa:qa-unit`). Listing it under `agents` gives the clean
  `qa-suite:qa-unit`, which is what `agents/qa.md` section 2 refers to.
- **`tools:` in an agent's frontmatter is a strict allowlist.** An agent that
  needs to invoke a skill must list `Skill`, or it sees no skills at all.

Renaming the plugin renames every agent and skill, so the table in
`agents/qa.md` section 2 has to be updated along with it.

## Status

`qa-unit` and its `unit-testing` skill are implemented. The other eight
sub-agents are frontmatter-only stubs.
