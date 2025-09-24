# Repository Guidelines

## Project Structure & Module Organization
- `local.yml`: Main Ansible playbook orchestrating all tasks.
- `tasks/`: Ansible task files included by `local.yml` (e.g., `core.yml`, `zsh-setup.yml`).
- `mise-tasks/`: File-based mise tasks to run common workflows (e.g., `fedora`, `test/fedora`).
- `docker/`: Container build contexts to test provisioning in isolation.
- `install`, `ubuntu`, `fedora`, `macos`: Bootstrap scripts to install Ansible and run the playbook.
- `key/`: Local vault/key material. Do not commit secrets beyond what already exists.
- `backlog/`, `motif/`, `.vscode/`: Supporting assets and configs.

## Build, Test, and Development Commands
- Provision locally (auto-detect OS): `./install`
- Ubuntu provisioning: `./ubuntu --tags core` (prompts for sudo/vault)
- Fedora provisioning: `./fedora` (prompts for sudo/vault)
- Run specific sections: `ansible-playbook local.yml --tags core,zsh`
- Using mise tasks: `mise fedora` (host) • `mise test:fedora` (Docker test)
- Dry run / audit: `ansible-playbook local.yml --check --diff`

## Coding Style & Naming Conventions
- Ansible YAML: 2‑space indentation, kebab-case task names, idempotent tasks, use `tags`.
- Shell scripts: `bash` or `zsh` with `set -euo pipefail` (where applicable); keep scripts executable.
- File naming: Use descriptive lowercase names (`tasks/git-setup.yml`, `mise-tasks/test/fedora`).

## Testing Guidelines
- Prefer Docker-backed tests before running on your machine: `mise test:fedora`.
- Validate idempotency: run `ansible-playbook local.yml` twice without changes.
- Scope with tags during development (e.g., `--tags core`), then test full play.

## Commit & Pull Request Guidelines
- Commits: concise, imperative. Examples: `feat(core): add docker install`, `fix: vault key path`.
- Keep diffs focused; avoid unrelated cleanup. Ensure files are formatted and executable bits set.
- PR/MR: include purpose, scope, notable tags affected, and testing notes (dry‑run, docker test).

## Security & Configuration Tips
- Do not commit secrets. Store sensitive values outside VCS (e.g., `mise.local.toml` env vars or Ansible Vault).
- If provisioning requires VPN or SSO, log in first; vault prompts need local key material in `key/`.
- Review commands with `--check --diff` before applying destructive changes.

