# AGENTS.md

> Project memory and operating rules for AI agents (GitHub Copilot, Claude Code, Cursor, Gemini, etc.)
> This file is the authoritative context layer. Read it fully before making any change.

---

## System Architecture

This repository is a collection of **Dev Container Features** managed under the [Dev Container spec](https://containers.dev/implementors/features/).

Each Feature is a self-contained unit:
- A shell script (`install.sh`) that installs a language/tool/CLI inside a container
- A metadata file (`devcontainer-feature.json`) that declares options, environment variables, and VS Code customizations
- Optional helper scripts in a `scripts/` subdirectory

Features are published to `ghcr.io/devcontainers/features/<feature-id>` via GitHub Actions.

---

## Folder Structure

```
.
├── src/                        # One subdirectory per Feature
│   └── <feature-id>/
│       ├── devcontainer-feature.json   # Feature metadata & options
│       ├── install.sh                  # Main install script (runs as root inside container)
│       ├── README.md                   # Feature documentation
│       └── scripts/                    # Optional helper/vendor scripts
│           └── vendor/                 # Vendored third-party scripts (e.g. dotnet-install.sh)
├── test/                       # One subdirectory per Feature (mirrors src/)
│   └── <feature-id>/
│       ├── test.sh             # Main test script (run by devcontainer CLI)
│       ├── scenarios.json      # Named install scenario matrix
│       └── *.sh                # Per-scenario test scripts
├── .github/
│   └── workflows/              # CI/CD automation
│       ├── docker-publish.yml              # Publishes features to GHCR on release
│       ├── validate-metadata-files.yml     # Validates devcontainer-feature.json files
│       ├── update-documentation.yml        # Auto-generates README docs
│       ├── update-dotnet-install-script.yml # Weekly vendor script updater
│       └── update-aws-cli-completer-scripts.yml
└── AGENTS.md                   # This file
```

---

## Coding Rules

### Shell Scripts (`install.sh`, helper scripts)
- Always use `#!/bin/sh` or `#!/bin/bash` shebang on line 1
- Scripts run as `root` inside the container during Feature installation
- Use `set -e` to fail fast on errors
- Do not hardcode versions — read from `devcontainer-feature.json` options passed as environment variables
- Prefer POSIX-compatible sh where possible; use bash only when necessary
- Never leave temporary files after installation; clean up in the same script

### `devcontainer-feature.json`
- `"version"` follows [semver](https://semver.org/) — bump **patch** for automated vendor script updates, **minor** for new options, **major** for breaking changes
- All options must have a `"default"` value and a `"description"`
- `"containerEnv"` sets environment variables available inside the container
- `"installsAfter"` lists Features this Feature depends on

### Version Bumping
- Automated workflows bump the **patch** version (`.Z` in `X.Y.Z`) via `jq`
- Manual changes to behavior or options require a human to bump minor/major

---

## Testing Rules

- Tests live in `test/<feature-id>/`
- `scenarios.json` defines named build scenarios (base image + feature options)
- `test.sh` (and per-scenario `*.sh` files) are executed by the [devcontainer CLI](https://github.com/devcontainers/cli)
- Run tests locally: `devcontainer features test --workspace-folder . --features <feature-id>`
- CI runs tests via `.github/workflows/` on PRs
- Never remove or weaken existing test assertions
- When adding a new Feature option, add a corresponding test scenario

---

## CI / Workflow Definitions

| Workflow | Trigger | Purpose |
|---|---|---|
| `docker-publish.yml` | Release tag push | Publish Feature OCI images to GHCR |
| `validate-metadata-files.yml` | PR / push | Validate all `devcontainer-feature.json` files |
| `update-documentation.yml` | Push to main | Auto-update Feature READMEs |
| `update-dotnet-install-script.yml` | Weekly (Sunday 00:00 UTC) + manual | Fetch latest `dotnet-install.sh` from Microsoft, bump dotnet+oryx patch versions, open PR |
| `update-aws-cli-completer-scripts.yml` | Schedule | Similar vendor update for AWS CLI |

### Automated PR Branches
Automated workflows create branches named `automated-script-update-<GITHUB_RUN_ID>`.  
These branches use `git push --force-with-lease` to safely overwrite stale remote branches on re-runs.

---

## Security Constraints

- `secrets.PAT` (Personal Access Token) is stored in the `documentation` GitHub Actions environment
- Never log or echo `GITHUB_TOKEN` or `PAT`
- Vendor scripts (e.g. `src/dotnet/scripts/vendor/dotnet-install.sh`) are fetched from official Microsoft sources only
- Do not add new external dependencies without review

---

## Agent Workflow Guidelines

### Before making any change
1. Read this file (`AGENTS.md`) fully
2. Identify which Feature(s) are affected (`src/<feature-id>/`)
3. Check if there is a corresponding test in `test/<feature-id>/`
4. Check relevant CI workflows in `.github/workflows/`

### When fixing a bug
- Make the minimal change in `install.sh` or helper scripts
- Bump patch version in `devcontainer-feature.json` only if the fix changes installed behavior
- Verify the fix does not break existing test scenarios

### When updating a vendor script
- Replace the file in `src/<feature-id>/scripts/vendor/`
- Also sync to any dependent Features (e.g. `src/oryx/scripts/vendor/` mirrors `src/dotnet/scripts/vendor/`)
- Bump patch version in both affected `devcontainer-feature.json` files

### When adding a new Feature option
- Add the option to `devcontainer-feature.json` with a `default` and `description`
- Handle the option in `install.sh`
- Add a test scenario in `test/<feature-id>/scenarios.json` and a corresponding `*.sh` test script
- Bump minor version

### Commit message conventions
- `Automated dotnet-install script update` — vendor script automation
- `Bump version` — version-only commit from automation
- `Fix: <short description>` — manual bug fixes
- `Feat: <short description>` — new options or features

### Never
- Force-push to `main`
- Merge without CI passing
- Add platform-hosted deployment steps (prefer local/self-hosted execution paths)
- Modify `AGENTS.md` to relax any rule without explicit human approval

---

## Key Files Reference

| File | Purpose |
|---|---|
| `src/<id>/devcontainer-feature.json` | Feature manifest, options, env vars |
| `src/<id>/install.sh` | Container install entrypoint |
| `src/<id>/scripts/fetch-latest-*.sh` | Vendor script updater (runs in CI) |
| `src/<id>/scripts/vendor/*.sh` | Vendored third-party scripts |
| `test/<id>/scenarios.json` | Test scenario matrix |
| `test/<id>/test.sh` | Main test runner |
| `.github/workflows/*.yml` | All CI/CD automation |
| `CODEOWNERS` | Review requirements |
