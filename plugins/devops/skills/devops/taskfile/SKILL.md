---
name: taskfile
description: Write and fix Taskfile.yml for go-task — wrapping docker and docker compose behind `task up` / `task down` / `task deploy`, making setup steps idempotent with `status:`, splitting a monorepo with includes and `task -d`, passing host UID/GID into containers, and cutting an overgrown Taskfile back down. Use whenever a project needs a single command to start, a Taskfile is being written or refactored, a task reruns work it should skip or runs steps in the wrong order, or `task --list` no longer describes anything useful. Task only — not Make, just, or npm scripts.
---

# Taskfile

Task is a YAML task runner. Used well, `Taskfile.yml` is the interface to a
project: five commands a newcomer can run without reading anything else.

## The shape

```yaml
version: '3'

vars:
  COMPOSE: docker compose

tasks:
  default:
    desc: "List available tasks"
    silent: true
    cmds: [task --list]

  up:
    desc: "Start the stack"
    deps: [network]
    cmds:
      - "{{.COMPOSE}} up -d --build"
      - task: ps

  down:
    desc: "Stop the stack (keeps volumes)"
    cmds: ["{{.COMPOSE}} down"]

  logs:
    desc: "Follow logs: task logs -- api"
    cmds: ["{{.COMPOSE}} logs -f --tail 100 {{.CLI_ARGS}}"]

  ps:
    desc: "What is running"
    cmds: ["{{.COMPOSE}} ps"]

  network:
    internal: true
    status: ["docker network inspect traefik"]
    cmds: ["docker network create traefik"]
```

That is a complete, useful Taskfile. Resist growing it beyond what someone
actually types.

## The three fields that carry the weight

**`status:`** — a list of shell commands. All exit 0 → the task is already done,
skip it, but **still run whatever depends on it**. This is what makes setup
idempotent: "network exists", "certificate issued", "binary present". Cheap
checks with no side effects, so `task up` is safe to run twice.

**`preconditions:`** — looks similar, behaves oppositely. Fails the task *and*
everything depending on it. Use it for "docker is running", "this variable is
set" — things that must be true, not things you can create.

**`sources:` / `generates:`** — checksum-based. Skip when inputs are unchanged.
Right for builds (`sources: ["**/*.go"]`, `generates: ["bin/app"]`), wrong for
"does this Docker resource exist", which `status:` answers directly.

## Rules

- **`deps:` always run in parallel.** Anything order-dependent belongs in `cmds:`
  as sequential `- task: <name>` entries. This is the most common Taskfile bug,
  and it produces intermittent failures that look like Docker flakiness.
- **`desc:` on everything a person types**, `internal: true` on everything else.
  `task --list` is the documentation.
- **Destructive tasks get their own name and never hide inside `up`.** Anything
  running `down -v`, `prune` or `rm -rf` is typed deliberately or not at all.
- **`{{.CLI_ARGS}}` for pass-through**: `task logs -- api -f`. Everything after
  `--` lands in that variable.
- **Reference tasks, don't inline duplicates.** `- task: ps` beats repeating the
  command.
- **`silent: true`** on tasks whose own echo is noise (`task --list`, banners).
- Keep the root Taskfile thin: the interface, plus `includes:` for the rest.

## Monorepos

```yaml
includes:
  proxy:
    taskfile: ./Taskfile.proxy.yml
    flatten: true            # tasks keep bare names: `task certs`, not `task proxy:certs`
  docs:
    taskfile: ./docs
    optional: true           # no error when the directory isn't there
```

**Included Taskfiles cannot use `dotenv:`** — Task documents this explicitly.
A subproject that loads its own `.env` therefore cannot be pulled in with
`includes:` at all. Call it instead:

```yaml
  test:
    cmds:
      - task -d services/auth test
      - task -d services/users test
```

`task -d <dir>` is `cd <dir> && task`, and it picks up that directory's `.env`
correctly. This is a real limitation, not a mistake in your file.

## Workflow

1. **Read what exists** — compose file, Makefile, README commands, CI config.
   The tasks are usually already written down somewhere as prose.
2. **Name the interface first**: `up`, `down`, `logs`, `ps`, `test`, `deploy`.
   Everything else is `internal:` or doesn't exist.
3. **Add `status:` to every setup step** that creates something.
4. **Check it**: `task --list` shows the interface; `task --dry <name>` prints
   what would run without running it; run `up` twice and confirm the second is
   fast and quiet.

## Then

- Docker-specific shapes: idempotent network, UID/GID, `docker run --rm` as a
  tool, waiting for health → `references/docker-patterns.md`
- Variables, templating, loops, platform guards → `references/syntax.md`
- The compose file the tasks call → `devops:compose`

Source: [taskfile.dev](https://taskfile.dev/docs/guide).
