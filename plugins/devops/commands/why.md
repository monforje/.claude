---
description: Diagnose a Docker failure — build error, container that won't start, unhealthy service, or services that can't reach each other
argument-hint: "[what is broken, e.g. 'api exits with 137']"
allowed-tools: Read, Grep, Glob, Bash, Skill
---

# Why is this broken

Diagnose the container problem described in `$ARGUMENTS`. If nothing was
described, find what is currently failing.

## Steps

1. **Look before theorising.**
   ```bash
   docker compose ps -a
   docker compose logs --tail 100
   ```
   An exited container still has its logs; `-a` is what shows it.

2. **Load `devops:container-debugging`** and follow it. Its `references/` split
   the two halves: `build-failures.md` and `runtime-failures.md`.

3. **Name the failure precisely** before proposing anything: which service, at
   which point (build / start / healthcheck / request), which exit code, which
   error line. `docker inspect <c> --format '{{.State.ExitCode}}
   {{.State.OOMKilled}} {{.State.Error}}'` answers most of it in one command.

4. **Reproduce it small.** `docker compose run --rm --entrypoint sh <service>`
   keeps the environment, mounts and networks while handing you a shell. Almost
   every runtime problem becomes obvious there.

5. **Change one thing and re-run.** Multiple simultaneous edits make the cause
   unattributable.

6. **Report:** what fails, why, the fix, and how to confirm it.

## Rules

- **Read-only until the cause is known.** Investigate first; propose the fix; ask
  before editing files.
- **Never `docker system prune`, `down -v`, or `rm -f` someone else's container**
  as a debugging step. "Reset everything" destroys the evidence and often the
  user's data. `docker compose down && docker compose up -d` recreates
  containers without touching volumes and is almost always what was meant.
- **Say when it is not a Docker problem.** A container that starts and then
  returns 500s is an application bug. Report that finding rather than continuing
  to look at Docker.
- **No guessing.** If the logs don't say, get more output
  (`--progress=plain`, `docker events`, `docker inspect`) rather than
  speculating.
