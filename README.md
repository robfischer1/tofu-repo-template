# tofu-repo-template

> **DEPRECATED (Smaller Hammers F8, 2026-07-05).** A substrate star no longer
> needs a separate template: the decentralization refactor gave every star
> (`python-repo-template`, `decentralized: true`) its own `/infra` scaffolded
> from the shared foundry `star` module. A substrate star is that same `/infra`
> shape — the broker/registry declared through the `star` module's `db`/`stores`/
> `extras` inputs (see the Pontus child design). Do NOT stamp new substrate stars
> from here; use `python-repo-template`. Retained read-only as the prior-art
> reference until the last substrate star (Pontus) adopts its per-repo `/infra`.

---

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
2. `git remote add origin https://git.notusmi.com/<service_name>.git` — the **Ourea door** (bare repo name, no `rob/` prefix)
3. `git remote add forgejo https://forgejo.notusmi.com/rob/<service_name>.git` + `git remote set-url --push forgejo no_push_f27` — **fetch only**
4. `git remote add github https://github.com/robfischer1/<service_name>.git` — push mirror
5. `uvx detect-secrets@1.5.0 scan > .secrets.baseline`
6. `specify init --here --force --integration claude --script ps --ignore-agent-tools` — the spec-kit SDD scaffold
7. `specify extension disable agent-context` — cedes `CLAUDE.md` to furnace instead of spec-kit's agent-context extension
8. `furnace ignite . --kit code-repo-sdd` — pours the furnace governance kit (`.claude/` AI layer)
9. `specify preset resolve speckit.plan` — verifies the plan-command override won

Steps 6–9 require `specify` and `furnace` on `PATH` and `$FURNACE_SOURCE` set.

The remote wiring is the post-F27 convention: `origin` is the Ourea door, the
authoritative git gateway that gates landings via `Serves`. Forgejo stays as a
fetch-only remote with its push side disabled by the `no_push_f27` sentinel, so
a stray `git push forgejo` fails loudly rather than silently bypassing the gate.
Steps 2–4 only wire local remotes — the repo must already exist on Forgejo
(push-to-create is disabled instance-wide), so create it there first.

## Development (of the template itself)

Every PR renders the matrix in `ci-matrix.toml` through the canonical
`template-ci.yml` in `foundry/foundry-stocks`: each case renders, its expected
file set is asserted, and every rendered `.toml`/`.json` is parsed. This
template has no conditional paths, so that last part — `star.toml` parsing under
both an empty and a populated `substrate_kind` — is the gate that matters.

Since 2026-08-12 a second matrix runs beside it —
`template_update_matrix.py`, which gates the `copier update` path the render
matrix cannot see — and this repo has its own pre-commit config that runs both.
Install it and you get them for free; see
[Development (on this template repo)](#development-on-this-template-repo).

Reproduce either locally:

```bash
python3 /path/to/foundry-stocks/ci/lib/template_render_matrix.py --template .
python3 /path/to/foundry-stocks/ci/lib/template_update_matrix.py --template .
```

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
| `star.toml` | the star manifest (identity, tier, seams; optional commented-out `[operable]` curation stub) — conforms to the schema `stellar_core`'s `build_admission_input` builds and `ouranos`'s rego policy validates (`rob/constellation`, the prior schema authority, is archived) |
| `versions.tf` | Tofu + provider pins (`kreuzwerker/docker`) |
| `providers.tf` | docker daemon connection (`var.docker_host`) |
| `variables.tf` | inputs (the nas01 daemon endpoint) |
| `main.tf` | the runtime resources (authored per feature via spec-kit) — starts as a commented skeleton |
| `.forgejo/workflows/admit.yml` | thin caller (`uses: foundry/foundry-stocks/.forgejo/workflows/admit.yml@main`) — the fail-closed admission logic (policy-bundle pull + cosign verify, supply-chain attestation, schema conformance) lives in that reusable workflow, not inlined here |
| `.gitignore` | the one file `copier update` actually 3-way-merges — every other rendered path is create-if-absent or foundry-anneal-owned (see "Updating a stamped repo" below) |
| `LICENSE` | Apache-2.0, stamped with `author_name` / `year` |
| `.specify/` + `.claude/` | spec-kit scaffold + furnace governance (poured by the `_tasks`, not by copier's file rendering) |

`main.tf` is intentionally a skeleton — the actual `docker_image` /
`docker_container` resources are authored per-feature via the spec-kit inner
loop (`/execute` on the master-plan for the new star), not by this template.

## Updating a stamped repo

`copier update` re-renders `template/` against a newer version of this
template and merges into the target repo per `_skip_if_exists` (Flame C1;
see `copier.yml`):

- **create-if-absent** — `README.md`, `LICENSE`, `main.tf`, `variables.tf`,
  `providers.tf`, `versions.tf`, `star.toml`: star-owned once born, copier
  never clobbers an existing one.
- **skipped here, clobbered by anneal** — `.forgejo/**`, `cliff.toml`: the
  foundry conformance surface is enforced by hephaestus anneal (C2) from a
  fresh render, not merged by copier.
- **3-way merge** — nothing. `.gitignore` was the last real seam and was
  closed 2026-08-12; the only rendered path copier still rewrites is
  `.copier-answers.yml`, which it owns (that is how the pin advances).

So an update ADDS files a star has never had and rewrites none of its own. A
change to a file this template ships therefore reaches only NEW stamps unless it
ships with a versioned `_migrations` entry at the `after` stage — the only stage
that runs once copier has reapplied its diff. A `_task` cannot do it; tasks run
before the diff and are overwritten.

`_tasks` (the numbered list above) never re-run on `copier update` — every
one is gated `when: "{{ _copier_operation == 'copy' }}"`, so they only fire
on the first `copier copy`.

## Development (on this template repo)

Install the hooks first — this repo has its own gate as of 2026-08-12, where
before it gated only its children:

```bash
uvx pre-commit install
```

`pre-commit` checks the SOURCE repo (`copier.yml` as YAML, `ci-matrix.toml` as
TOML, no committed conflict markers). `pre-push` runs the same two matrices CI
runs — render and update — so a push that would red `template-ci` fails locally
first. `template/` is excluded from the parsers on purpose: it is jinja source,
not the rendered result. Parsing the RENDERED tree is the render matrix's job.

Both matrices live in `foundry/foundry-stocks` so all five templates share one
definition. Set `FOUNDRY_STOCKS` if your checkout is not at `../foundry-stocks`;
absent, the hooks print a notice and pass, and CI enforces.

There is still no Python/Node env for the *stamped* repo, and secret-scanning and
schema conformance are validated in its own CI
(`.forgejo/workflows/admit.yml`), not here.

To iterate on the template itself: edit files under `template/`, then dry-run
with `copier copy --trust . /tmp/some-test-star` and inspect the rendered
output. `copier.yml` is the single source of truth for the question set and
task list.
