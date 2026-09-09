# Task syntax worth knowing

Only the parts that come up when wrapping Docker. Full reference:
[taskfile.dev/docs/reference/schema](https://taskfile.dev/docs/reference/schema).

## Variables

Precedence, highest first: command line → task-level `vars:` → call-site `vars:`
→ global `vars:` → `env:` → dotenv.

```yaml
vars:
  IMAGE: myapp
  TAG:
    sh: git rev-parse --short HEAD        # computed once, at parse time
  UID:
    sh: id -u

tasks:
  build:
    cmds: ["docker build -t {{.IMAGE}}:{{.TAG}} ."]

  release:
    cmds:
      - task: build
        vars: { TAG: "{{.VERSION}}" }     # pass a variable into another task
```

- `sh:` runs when the Taskfile is parsed, not when the task runs — so it costs
  time on *every* invocation. Keep those cheap (`id -u`, `git rev-parse`).
- `{{.VAR}}` is Go templating. Available: `OS`, `ARCH`, `CLI_ARGS`, `TASK`,
  `ROOT_DIR`, `TASKFILE_DIR`, `USER_WORKING_DIR`.
- Functions: `{{if eq OS "darwin"}}...{{end}}`, `{{default "dev" .TAG}}`,
  `{{.NAME | lower}}`, `{{shellQuote .PATH}}`.
- **A literal `{{` in a shell command must be escaped**, which bites with Docker
  format strings: write `` {{`{{.Names}}`}} `` to get `{{.Names}}` through to
  `docker ps --format`.

## Environment

```yaml
env:
  COMPOSE_PROJECT_NAME: myapp
  DOCKER_UID:
    sh: id -u

dotenv: ['.env', '.env.{{.ENV}}']         # root Taskfile only

tasks:
  psql:
    dotenv: ['.env.db']                   # per task, also allowed
    env:
      PGPASSWORD: "{{.PASSWORD}}"
```

`dotenv:` works at the root and per task, but **not in an included Taskfile** —
see the SKILL for the `task -d` workaround.

## Required inputs

```yaml
  deploy:
    requires:
      vars: [VERSION]
      # or: vars: [{ name: ENV, enum: [dev, staging, prod] }]
    preconditions:
      - sh: "docker info >/dev/null 2>&1"
        msg: "Docker is not running"
      - sh: "test -f .env"
        msg: "No .env — run `task init` first"
    prompt: "Deploy {{.VERSION}} to production?"
```

`prompt:` asks for confirmation before running and is skipped by `--yes`. It is
the right guard on anything destructive or outward-facing.

## Loops

```yaml
  lint:
    cmds:
      - for: ['auth', 'users', 'tasks']
        cmd: task -d services/{{.ITEM}} lint

  build-all:
    cmds:
      - for: { var: SERVICES, split: ',' }
        cmd: docker build -t {{.ITEM}} ./{{.ITEM}}

  compress:
    sources: ['assets/*.png']
    cmds:
      - for: sources
        cmd: optipng {{.ITEM}}
```

## Control

```yaml
  clean:
    cmds:
      - cmd: docker rm -f temp-container
        ignore_error: true          # continue even if it fails
  test:
    cmds:
      - docker compose up -d db
      - defer: docker compose down  # runs even if a later command fails
      - go test ./...
  mac-only:
    platforms: [darwin]
  once:
    run: once                       # even if several tasks depend on it
```

`defer:` is the cleanup mechanism — without it a failing test leaves containers
running.

## Output and dry runs

```yaml
version: '3'
output: prefixed          # or 'interleaved' (default), 'group'
```

- `task --list` / `--list-all` (includes tasks without `desc:`)
- `task --dry <name>` — print commands without executing
- `task --summary <name>` — show `summary:` text
- `task --force <name>` — ignore `status:`/`sources:` and run anyway
- `task --watch <name>` — rerun when `sources:` change
- `task --parallel <a> <b>` — run several tasks concurrently
