---
description: Read-only audit of this project's Dockerfiles, compose files, Taskfile and Traefik labels
argument-hint: "[path or service name]"
allowed-tools: Read, Grep, Glob, Bash(docker compose config:*), Bash(git ls-files:*), Bash(wc:*), Skill
---

# Review the container setup

Audit the Docker setup in `$ARGUMENTS` (default: the whole repository).

**This command does not change anything.** No edits, no builds, no `up`, no
`down`. If a fix is worth making, describe it and let the user ask.

## Steps

1. **Find the files.**
   ```bash
   git ls-files | grep -Ei 'dockerfile|compose.*\.ya?ml|\.dockerignore|Taskfile.*\.ya?ml'
   ```
   Nothing found → say so and stop.

2. **Validate what can be validated cheaply.** `docker compose config -q` reports
   schema errors and unresolved variables without starting anything. It is the
   only command you may run against Docker.

3. **Load the relevant skill before judging** — `devops:dockerfile`,
   `devops:compose`, `devops:traefik`, `devops:taskfile`. Do not review from
   memory; the rules and the per-service blocks live in those files.

4. **Check, in this order of severity:**
   - *critical* — a secret in `ENV`/`ARG` or committed; the Docker socket mounted
     writable; `privileged: true`; a stateful service with no volume; a published
     database with no password.
   - *major* — no `.dockerignore`; `depends_on` without `condition:
     service_healthy` where a dependency needs one; container runs as root;
     unpinned images (`latest`, or no tag); source copied before dependency
     install; build stage shipped as the runtime image; `version:` key still
     present.
   - *minor* — shell-form `CMD`; no `HEALTHCHECK`; missing `restart:`; ports
     published that nothing on the host uses; missing `desc:` in the Taskfile.

5. **Report to the terminal. No file is written.**

## Output

```
<n> files reviewed: <list>

[critical] compose.yaml:14 — POSTGRES_PASSWORD is a literal in the file → use a
           file secret with POSTGRES_PASSWORD_FILE
[major]    Dockerfile:8 — COPY . . precedes npm ci, so every source edit
           reinstalls dependencies → copy package*.json first
...

Verdict: <one or two lines>
```

## Rules

- **At most 10 findings**, most severe first. If there are more, say how many
  were left out and in which category. A wall of style notes gets ignored
  wholesale.
- Every finding needs a file and a line. "Consider adding healthchecks" is not
  a finding; `compose.yaml:22 — db has no healthcheck, yet api depends on it` is.
- **Only report what is actually wrong.** A missing healthcheck on a service
  nothing depends on is not a defect. No findings is a valid, useful result —
  say "nothing to fix" rather than filling the list.
- Judge against what the project is. A local development stack does not need
  production hardening, and saying so is part of the review.
