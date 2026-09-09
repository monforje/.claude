---
description: Create a container setup from scratch — Dockerfile, .dockerignore, compose.yaml and a minimal Taskfile
argument-hint: "[stack description, e.g. 'fastapi + postgres + redis']"
allowed-tools: Read, Grep, Glob, Write, Edit, Bash, Skill
---

# Set up containers for this project

Create the container setup for this repository. `$ARGUMENTS` describes the stack
if given; otherwise work it out from the repository.

## Steps

1. **Detect before writing.** Language and package manager (`go.mod`,
   `package.json`, `pyproject.toml`, `requirements.txt`), the build and start
   commands, the port the app listens on, and what it connects to (search the
   config and env handling for database, cache and broker URLs).

2. **Check what already exists.** A Dockerfile or compose file already here means
   this is not an `init` — say what you found and offer `/devops:review` instead.
   Never overwrite an existing file without asking.

3. **Confirm the plan in three lines** before writing: which services, which
   images, which ports. One round, then proceed.

4. **Load the skills** — `devops:dockerfile` for the image, `devops:compose` for
   the stack, `devops:taskfile` for the entry commands, `devops:traefik` only if
   hostname routing was asked for.

5. **Write:**
   - `Dockerfile` — multi-stage, non-root, dependencies before source
   - `.dockerignore` — before anything else touches the build context
   - `compose.yaml` — services, named volumes, healthchecks,
     `depends_on: condition: service_healthy`
   - `Taskfile.yml` — **minimal**: `up`, `down`, `logs`, `ps`, and `deploy` only
     if it does more than `up`. Around 30 lines. It is an entry point, not an
     inventory of every command.
   - `.env.example` if the stack needs configuration; never a real `.env`.

6. **Verify.** `docker compose config -q`, then `docker compose build`, then
   `task up` (or `docker compose up -d`) and `docker compose ps` until every
   service is `healthy` or `running`. Fix what fails; that iteration is the job.

7. **Clean up.** Stop what you started unless the user wanted it running. Do not
   remove volumes.

## Rules

- Small. A stack that starts beats a stack that covers every eventuality.
- Pin every image to `major.minor`.
- No secrets in the files you write. Placeholders in `.env.example`, real values
  are the user's to add.
- Do not add a service the project doesn't use. If nothing in the code speaks to
  Redis, there is no Redis.
- Report at the end: which files were created, the command to start, and the URL.
