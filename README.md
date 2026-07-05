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

## Use

```bash
uvx copier@latest copy --trust /path/to/tofu-repo-template /path/to/new-star
```

The final `_task`s run `specify init` (the SDD scaffold) then
`furnace ignite --kit code-repo-sdd` (the governance kit), so the repo is born
spec-kit-ready and governed. Requires `specify` + `furnace` on PATH and
`$FURNACE_SOURCE` set.

## What it lays

| File | Role |
| :-- | :-- |
| `star.toml` | the star manifest (identity, tier, seams) — conforms to the `constellation` StarManifest |
| `versions.tf` | Tofu + provider pins (`kreuzwerker/docker`) |
| `providers.tf` | docker daemon connection (`var.docker_host`) |
| `variables.tf` | inputs (the nas01 daemon endpoint) |
| `main.tf` | the runtime resources (authored per feature via spec-kit) |
| `.specify/` + `.claude/` | spec-kit scaffold + furnace governance (poured by `_task`s) |
