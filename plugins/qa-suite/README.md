# qa-suite

A QA orchestrator plus specialised verification sub-agents.

| Component | Location | Name in a session |
| --- | --- | --- |
| Orchestrator | `agents/qa.md` | `qa-suite:qa` |
| Sub-agents | `agents/qa/*.md` | `qa-suite:qa-<name>` |
| Skills | `skills/qa/*/SKILL.md` | `qa-suite:<skill-name>` |

Install and development workflow live in the [repository README](../../README.md).

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

`qa-unit` (`unit-testing` skill) and `qa-integration` (`integration-testing`
skill) are implemented. The other seven sub-agents are frontmatter-only stubs.

`qa-integration` can use the `devops` plugin's `devops:compose` skill when that
plugin is installed, but does not require it — everything it needs is in its own
skill.
