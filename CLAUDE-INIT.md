Codebase orientation for AI sessions. Posture and governance live in
AGENTS.md (furnace-compiled); this file is the repo-specific map, read on
demand.

## Overview

`tofu-repo-template` is a **copier template**, not an application. Its only
"function" is to be copied (`copier copy --trust`) to produce a new Forge
**OpenTofu substrate star** repo — a thin per-substrate repo whose runtime is
declared as `kreuzwerker/docker` resources and deployed via **Nereus**
(OpenTofu), never hand-composed. Role in the fleet: the Tofu-flavored sibling
of `python-repo-template` — same governance + spec-kit spine, a `.tf` body
instead of a Python package. There is no Python/Node runtime in this repo
itself (`copier.yml` comment: "No uv/pyproject/pre-commit — there is no
Python env").

Everything a session needs to reason about lives in two files:
`copier.yml` (the question set + post-copy `_tasks`) and `template/` (the
files that get rendered into the new repo). Nothing under `template/` is
meant to run as-is inside *this* repo — the `{{ ... }}` Jinja placeholders
resolve only when copier renders them into the target repo.

## Architecture / module map

```
copier.yml                              # questions + defaults + _tasks (the whole template's logic)
template/                               # _subdirectory root — rendered into the new repo
├── star.toml.jinja                     # star manifest: identity/cluster/charter, [substrate],
│                                        #   [logic] entrypoint=main.tf, [interface] sync/async,
│                                        #   [observability] exports, [governance] policy bundle pin,
│                                        #   optional [operable] curation stub (Argus F1, commented out)
├── versions.tf                         # Tofu >=1.8.0 + kreuzwerker/docker ~> 3.0 pin (NOT templated)
├── providers.tf                        # provider "docker" { host = var.docker_host } (NOT templated)
├── variables.tf.jinja                  # var.docker_host, default from the docker_host answer
├── main.tf.jinja                       # commented skeleton only — real docker_image/docker_container
│                                        #   resources are authored later, per-feature, via spec-kit
├── .gitignore                          # NOT templated — the one file `copier update` 3-way-merges
│                                        #   (Flame C1); everything else here is create-if-absent
│                                        #   (star-owned) or anneal-clobbered, never merged
├── README.md.jinja                     # the README the STAMPED repo gets (contains live Jinja —
│                                        #   do not "fix" its {{ }} placeholders, they're intentional)
├── LICENSE.jinja                       # Apache-2.0, stamped with author_name/year
├── {{_copier_conf.answers_file}}.jinja # renders to .copier-answers.yml in the stamped repo —
│                                        #   copier's own answers record, used by `copier update`
└── .forgejo/workflows/admit.yml        # NOT templated (no Jinja) — thin caller stub to the
                                         #   foundry/foundry-stocks reusable admit workflow (not an
                                         #   inline gate); runs on every PR in the stamped repo
```

No `src/`, no tests, no CI in *this* repo (a template has nothing to unit
test; correctness is "does the render + `_tasks` sequence succeed").

## Entry points

- **The template's only entry point is `copier copy`** (or `copier update`
  on an already-stamped repo). There is no CLI, no server, no library API
  here.
- Inside the stamped repo, `star.toml`'s `[logic] entrypoint = "main.tf"` —
  Tofu is the application; there is no code entrypoint, only IaC resources.
- `_tasks` in `copier.yml` are the real "run" surface of this template — they
  execute in order, copy-only (`when: "{{ _copier_operation == 'copy' }}"`,
  skipped on `copier update`):
  1. `git init -q`
  2. `git remote add origin https://forgejo.notusmi.com/rob/{{ service_name }}.git`
  3. `git remote add github https://github.com/robfischer1/{{ service_name }}.git`
  4. `uvx detect-secrets@1.5.0 scan > .secrets.baseline`
  5. `specify init --here --force --integration claude --script ps --ignore-agent-tools`
  6. `specify extension disable agent-context`
  7. `furnace ignite . --kit code-repo-sdd` — the furnace governance kit; must
     run *after* `specify init` so its overrides win as last writer
  8. `specify preset resolve speckit.plan` — verifies step 7's override took

## Build / Test / Run

There is no build/test/lint for this repo (no manifest — no `pyproject.toml`,
no `package.json`, no `Dockerfile`, no `compose.yaml`, no `justfile`/
`Makefile`). The only executable action documented anywhere in this repo:

```bash
uvx copier@latest copy --trust /path/to/tofu-repo-template /path/to/new-star
```

Iterating on the template: dry-run into a scratch dir and inspect the
rendered tree —

```bash
copier copy --trust . /tmp/some-test-star
```

`specify` and `furnace` must be on `PATH`, and `$FURNACE_SOURCE` set, for
`_tasks` 5–8 to succeed. The Forgejo/GitHub remotes added in `_tasks` 2–3
assume the target repo already exists on Forgejo (push-to-create is disabled
instance-wide there).

Downstream, in a *stamped* repo: `tofu init`, `tofu plan`, `tofu apply`
(state lives in the shared Nereus PG backend, not locally) — see the
rendered `README.md.jinja`.

## Conventions and gotchas

- **`.jinja` suffix = templated, no suffix = copied verbatim.** `versions.tf`,
  `providers.tf`, and `.forgejo/workflows/admit.yml` have no `.jinja` suffix
  and contain no `{{ }}` — they're identical in every stamped repo. Don't add
  Jinja to them without a reason; don't strip Jinja from the `.jinja` files.
- `main.tf.jinja` renders to a **skeleton with the resources commented out**.
  This is intentional — the real `docker_image`/`docker_container` blocks are
  written per-feature via the spec-kit inner loop in the *stamped* repo, not
  by this template.
- `star.toml.jinja`'s `[governance]` block pins a cosign-signed policy bundle
  (`bundle`/`tag`/`digest`). This value is renovate-bumped in stamped repos,
  not something to hand-edit here casually — check `copier.yml`'s git log
  for why the default digest was corrected (`fix: default [governance] pin
  to the real policy-v2 bundle`) before changing it again.
- `admit.yml` is a thin caller (`uses: foundry/foundry-stocks/.forgejo/
  workflows/admit.yml@main`) — the pull-and-verify-policy-bundle logic lives
  in that reusable workflow, not inlined here. It still requires
  `[governance].digest` to be non-empty in `star.toml` or the gate fails
  closed — deliberate (fail-closed admission), not a bug to relax. The prior
  ~180-line inline gate (against `rob/constellation`, now archived) was
  replaced 2026-07-23 (`cfca7fbe`) because it only worked via a dead 301
  redirect.
- `_skip_if_exists` in `copier.yml` (Flame C1) governs `copier update`
  behavior: docs + the star's own Tofu files + `star.toml` are
  create-if-absent (star-owned once born, never re-clobbered); `.gitignore`
  is the one real 3-way-merge seam; `.forgejo/**` is skipped here and
  clobbered instead by hephaestus anneal (C2) from a fresh render — a
  conformance surface is enforced, not merged.
- `star.toml.jinja`'s `[operable]` block (Argus F1) is an opt-in,
  commented-out stub — uncomment and fill in `[[operable.verb]]` entries to
  declare the star's controllable surface. Two owners: the verb
  `name`/`schema` half is machine-introspected and re-derived on anneal; the
  `kind`/`group`/`danger`/`widget`/`label` curation half is author-owned and
  preserved across re-derivation (joined by name).
- Nyx (not Tofu) judges runtime health — `providers.tf` and `main.tf.jinja`
  both note the Tofu layer stays "deliberately dumb" (place containers only,
  no health logic).
- README.md.jinja is the stamped repo's README and legitimately contains
  Jinja — when editing docs in *this* repo, don't confuse it with this
  template's own top-level `README.md`.
- `.copier-answers.yml.jinja` (the `{{_copier_conf.answers_file}}.jinja`
  file) is what makes `copier update` possible later — don't delete or
  hand-edit its rendered output in a stamped repo.

## Related repos

- `python-repo-template` — the Python-body sibling; same copier + governance
  + spec-kit spine, referenced by name in both `README.md` and `copier.yml`'s
  header comment. Not present in this worktree; consult it directly for the
  Python-side equivalent of any convention here.
- `rob/constellation` — **archived** (2026-07-23); no longer supplies
  anything live. `star.toml`'s schema authority is now `stellar_core`
  (`build_admission_input`) + the `ouranos` rego policy, and `admit.yml`
  calls the `foundry/foundry-stocks` reusable workflow instead of the old
  inline `constellation.gate` (which only worked via a dead 301 redirect) —
  see commits `81ab7dee` / `cfca7fbe`.
- `foundry/foundry-stocks` — supplies the reusable
  `.forgejo/workflows/admit.yml` this template's own `admit.yml` calls
  (`uses: foundry/foundry-stocks/.forgejo/workflows/admit.yml@main`).
- `furnace` — supplies the `code-repo-sdd` governance kit poured by `_task`
  7 (`furnace ignite`).
- **Nereus** — the OpenTofu deployment layer that actually applies the
  `main.tf` this template lays (named in `README.md.jinja`, not a repo found
  in this worktree).
- **Nyx** — health-judging layer for the deployed containers (named in
  `providers.tf` / `main.tf.jinja` comments).
