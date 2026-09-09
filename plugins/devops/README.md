# devops

Docker, Compose, Traefik and Taskfile — writing, running and debugging a stack
on one machine.

| Component | Location | Name in a session |
| --- | --- | --- |
| Agents | `agents/*.md` | `devops:docker`, `devops:traefik`, `devops:taskfile` |
| Skills | `skills/devops/*/SKILL.md` | `devops:dockerfile`, `devops:compose`, `devops:traefik`, `devops:taskfile`, `devops:container-debugging` |
| Commands | `commands/*.md` | `/devops:review`, `/devops:init`, `/devops:why` |

Install and development workflow live in the [repository README](../../README.md).

## Scope

**In:** everything that runs on one machine through Docker Engine — images,
Compose stacks, Traefik as the Docker-label reverse proxy, Taskfile as the runner
around them. Building and pushing images (`buildx`, multi-arch, registries)
counts as in.

**Out:** Kubernetes and Helm, Terraform and Ansible, ECS / Cloud Run / App
Runner, and CI pipelines themselves. An agent that hits one of these says so and
stops rather than improvising.

The boundary is deliberate. "devops" is a wide word, and without a stated edge
the agents' descriptions start overlapping and the wrong one gets picked.

## Layout

Three peer agents, no orchestrator: the domains are distinguishable from a single
word in the request, so a routing layer would only add a hop and lose detail.
Each agent hands off explicitly — `docker` sends routing and TLS to `traefik`,
`traefik` sends a broken stack back to `docker`, `taskfile` writes neither.

```
agents/{docker,traefik,taskfile}.md
commands/{review,init,why}.md
skills/devops/
├── dockerfile/          languages, cache-and-secrets, hardening
├── compose/             datastores, brokers, apps, environments
├── traefik/             labels, tls-local, middlewares
├── taskfile/            syntax, docker-patterns
└── container-debugging/ build-failures, runtime-failures
```

Every file is under 150 lines. A `SKILL.md` is the entry point — when to use it,
the workflow, the rules — and anything longer than that lives in a
`references/*.md` it links to, so only the relevant one gets read.

## Conventions

Declared in `.claude-plugin/plugin.json`, and the declaration is what makes the
nested layout work:

- **Skills** must be listed under `skills` as directory paths. Auto-discovery
  only looks at `skills/<name>/SKILL.md`, so `skills/devops/<name>/` is invisible
  unless declared.
- **Agents** are auto-discovered recursively, but a subdirectory ends up in the
  name. Listing them under `agents` keeps them as `devops:<name>`.
- **Commands** are discovered from `commands/` and are flat, so `commands/review.md`
  is `/devops:review`.
- **`tools:` in an agent's frontmatter is a strict allowlist.** An agent that
  invokes a skill must list `Skill`, or it sees no skills at all.

Renaming the plugin renames every agent, skill and command, so the cross
references in the agents and in `plugins/qa-suite/` have to be updated with it.

## Sources

Distilled, not copied. Where a file states a rule that is easy to doubt, it links
back:

- [Docker build best practices](https://docs.docker.com/build/building/best-practices/)
- [docker/awesome-compose](https://github.com/docker/awesome-compose) — the
  `nginx-nodejs-redis`, `traefik-golang`, `react-nginx`, `fastapi` and
  `nginx-golang-postgres` examples are taken apart in
  `skills/devops/compose/references/apps.md`, including what each gets wrong
- [Traefik documentation](https://doc.traefik.io/traefik/) and
  [JensKnipper/traefik-examples](https://github.com/JensKnipper/traefik-examples)
- [taskfile.dev](https://taskfile.dev/docs/guide)
