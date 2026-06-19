# gofuego/.github

Org-shared, **reusable** GitHub Actions workflows for the [Fuego](https://github.com/gofuego/fuego)
ecosystem — both **Fuego sites** (check/deploy) and **Go modules** (build/test). A
repo calls these instead of copy-pasting a pipeline.

## Choosing a workflow

Two questions pick the workflow: **what is the repo** (a Go module or a Fuego
site), and **does the project own the repo root or live in a subdirectory**. Every
workflow takes a directory input (`working_directory` / `project_dir`, default
`.`) so the *subset* case — a docs site or module that is only part of a larger
repo — uses the same pipeline as a standalone one.

| Repo is… | CI (on PR / develop) | CD (on merge to main) |
|----------|----------------------|------------------------|
| a Go module (pack, service, tool) | `go-ci.yml` | — (or an image build, later) |
| a Fuego site | `fuego-check.yml` | `fuego-deploy.yml` |
| an ADR folder | — | `fuego-adr-deploy.yml` |

The **hosting target** is independent of all this: `studio_base` → `fuego-pages`
(the fuego-studio mount), `pages_base` → `gh-pages` (GitHub Pages). Pass either or
both — one deploy workflow covers studio-only, Pages-only, and dual-host sites.

## `go-ci.yml` — build & test a Go module

`go build ./...` + `go vet ./...` + `go test` (race by default). Use
`working_directory` for a module that is a subset of the repo.

| Input | Effect |
|-------|--------|
| `working_directory` | module dir (default `.`; a subdir for a subset module) |
| `race` | run tests with `-race` (default `true`) |
| `test_args` | packages/flags for `go test` (default `./...`) |
| `go_version` | Go toolchain (default `1.25`) |

```yaml
# .github/workflows/ci.yml in a Go repo
name: CI
on:
  pull_request:
  push:
    branches-ignore: [main, gh-pages, fuego-pages, fuego-studio-edits]
jobs:
  ci:
    uses: gofuego/.github/.github/workflows/go-ci.yml@main
```

## `fuego-deploy.yml` — build & publish

Builds a Fuego site for one or two base URLs and publishes each to its serve branch:

| Input | Effect |
|-------|--------|
| `studio_base` | build for the fuego-studio mount path → push to **`fuego-pages`** |
| `pages_base` | build for a GitHub Pages project site → push to **`gh-pages`** |
| `check_links` | fail the build on a broken internal link (default `false`; requires fuego ≥ v0.4.0) |
| `project_dir` | site project directory (default `.`; a subdir when the site is a subset of the repo) |
| `go_version` | Go toolchain (default `1.25`) |

Pass either, both, or `"/"` (root Pages). Omitting one skips that host — one
workflow covers **studio-only**, **dual-host**, and **Pages-only** sites.

```yaml
# .github/workflows/deploy.yml in a site repo
name: Deploy
on:
  push: { branches: [main] }
  workflow_dispatch:
jobs:
  deploy:
    uses: gofuego/.github/.github/workflows/fuego-deploy.yml@main
    with:
      studio_base: /gofuego/my-site   # → fuego-pages
      pages_base: /my-site            # → gh-pages (omit for studio-only)
```

## `fuego-check.yml` — compile & validate

Builds (`go build ./...`) and validates (`go run . validate`) the site **without
publishing**. Run it on pull requests — including the ones fuego-studio opens from
its edit branch — and on feature-branch pushes. Pass `project_dir` (default `.`)
when the site lives in a subdirectory of a larger repo (e.g. an engine's `docs/`).

```yaml
# .github/workflows/check.yml in a site repo
name: Check
on:
  pull_request:
  push:
    branches-ignore: [main, gh-pages, fuego-pages, fuego-studio-edits]
jobs:
  check:
    uses: gofuego/.github/.github/workflows/fuego-check.yml@main
```

## `fuego-adr-deploy.yml` — build & publish an ADR site

Builds an [ADR](https://github.com/gofuego/fuego-adr) site from a folder of
`*.adr.md` files and publishes it like `fuego-deploy`, but using the `fuego-adr`
tool (fetched at a pinned `adr_ref` — nothing is added to the repo's `go.mod`).

| Input | Effect |
|-------|--------|
| `adr_path` | folder of `*.adr.md` files (default `adrs`) |
| `studio_base` | build → push to **`fuego-pages`** (fuego-studio) |
| `pages_base` | build → push to **`gh-pages`** (GitHub Pages) |
| `adr_ref` | `fuego-adr` version (default pinned) |

```yaml
# .github/workflows/adrs.yml in a repo with an adrs/ folder
name: ADRs
on:
  push:
    branches: [main]
    paths: [adrs/**]
  workflow_dispatch:
jobs:
  adrs:
    permissions: { contents: write }
    uses: gofuego/.github/.github/workflows/fuego-adr-deploy.yml@main
    with:
      studio_base: /gofuego/my-repo   # served read-only in fuego-studio
```

ADR sites are typically served **read-only** — set `editing: false` in the repo's
`.fuego-studio.yaml`.

## Versioning

Callers reference `@main` for now (changes apply to every site on next run). If you
need change-isolation, we can cut a `v1` tag and pin callers to `@v1`.
