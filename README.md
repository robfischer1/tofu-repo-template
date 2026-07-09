# tofu-repo-template

A [copier](https://copier.readthedocs.io) template that stamps a Forge **OpenTofu
substrate star** — a thin per-substrate repo whose runtime is declared as
`kreuzwerker/docker` resources and deployed through **Nereus** (OpenTofu), never
hand-composed. The Tofu sibling of `python-repo-template`: same governance +
spec-kit spine, a `.tf` body instead of Python.

This is a **template repo**, not a finished star. It has no app code to run —
running it means running `copier copy` to stamp a *new* repo from it.

## Use

```bash
uvx copier@latest copy --trust /path/to/tofu-repo-template /path/to/new-star
```

Copier prompts for the answers below (see `copier.yml` for full defaults),
renders `template/` into the new repo, then runs the `_tasks` (copy-only,
skipped on `copier update`):

1. `git init -q`
2. `git remote add origin https://forgejo.notusmi.com/rob/<service_name>.git`
3. `git remote add github https://github.com/robfischer1/<service_name>.git`
4. `uvx detect-secrets@1.5.0 scan > .secrets.baseline`
5. `specify init --here --force --integration claude --script ps --ignore-agent-tools` — the spec-kit SDD scaffold
6. `specify extension disable agent-context` — cedes `CLAUDE.md` to furnace instead of spec-kit's agent-context extension
7. `furnace ignite . --kit code-repo-sdd` — pours the furnace governance kit (`.claude/` AI layer)
8. `specify preset resolve speckit.plan` — verifies the plan-command override won

Steps 5–8 require `specify` and `furnace` on `PATH` and `$FURNACE_SOURCE` set.
The Forgejo remotes in steps 2–3 assume the repo already exists on Forgejo
(push-to-create is disabled instance-wide) — create it there first.

## Copier questions

| Variable | Purpose | Default |
| :-- | :-- | :-- |
| `project_name` | Human-readable name (e.g. "Chronos") | — |
| `service_name` | Repo / image name, kebab-case | slugified `project_name` |
| `description` | One-line repo description | "A Forge OpenTofu substrate repo, deployed via Nereus." |
| `charter` | Single-responsibility charter (<=100 chars, the SRP gate) | `description` |
| `cluster` | Constellation cluster this star belongs to | `pantheon` |
| `substrate_kind` | Substrate engine kind (e.g. `postgres`, `redpanda`), empty if none | `""` |
| `star_namespace` | Sovereign namespace this substrate owns | `service_name` with `-` → `_` |
| `interface_protocol` | Primary sync wire (`postgres`\|`redpanda`\|`http`\|`tcp`) — satisfies Reachability | `substrate_kind` or `tcp` |
| `interstellar` | Produces/consumes Kafka events? Adds `async = ["kafka"]` | `false` |
| `docker_host` | Docker daemon endpoint Tofu provisions against | `ssh://rob@nas01` |
| `author_name` | Copyright holder | `Rob Fischer` |
| `year` | Copyright year | `2026` |

## What it lays

| File | Role |
| :-- | :-- |
| `star.toml` | the star manifest (identity, tier, seams) — conforms to the `constellation` StarManifest |
| `versions.tf` | Tofu + provider pins (`kreuzwerker/docker`) |
| `providers.tf` | docker daemon connection (`var.docker_host`) |
| `variables.tf` | inputs (the nas01 daemon endpoint) |
| `main.tf` | the runtime resources (authored per feature via spec-kit) — starts as a commented skeleton |
| `.forgejo/workflows/admit.yml` | fail-closed admission gate: pulls + cosign-verifies the pinned policy bundle, checks supply-chain attestation, runs `constellation.gate` + `conftest` against `star.toml` |
| `LICENSE` | Apache-2.0, stamped with `author_name` / `year` |
| `.specify/` + `.claude/` | spec-kit scaffold + furnace governance (poured by the `_tasks`, not by copier's file rendering) |

`main.tf` is intentionally a skeleton — the actual `docker_image` /
`docker_container` resources are authored per-feature via the spec-kit inner
loop (`/execute` on the master-plan for the new star), not by this template.

## Development (on this template repo)

No build, test, or lint step — there's no Python/Node env (per `copier.yml`:
"No uv/pyproject/pre-commit — there is no Python env"). Secret-scanning and
schema conformance are validated in the *stamped* repo's own CI
(`.forgejo/workflows/admit.yml`), not here.

To iterate on the template itself: edit files under `template/`, then dry-run
with `copier copy --trust . /tmp/some-test-star` and inspect the rendered
output. `copier.yml` is the single source of truth for the question set and
task list.
