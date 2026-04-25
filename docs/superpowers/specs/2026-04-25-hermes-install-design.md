# Hermes Agent — Install & Configuration Design

**Date:** 2026-04-25
**Scope:** Install Nous Research's Hermes Agent on the user's Ubuntu host and track its user-level configuration through the existing `.dotfiles` + `dev-environment` setup.
**Reference:** https://github.com/NousResearch/hermes-agent/blob/main/README.md

## Goal

Provision Hermes such that:
1. The `hermes` CLI is available after running `./install` from `dev-environment`.
2. The Hermes messaging gateway runs as a managed user-level systemd service.
3. Non-secret Hermes configuration is tracked in `.dotfiles` and stowed into `~/.hermes/`.
4. Provider API keys and platform tokens stay out of git.
5. The setup is reproducible across reprovisions of the host.

## Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Secret handling | Allow-list `.gitignore` in stow package; secrets entered manually per host | Avoids putting credentials in a git-tracked location; mirrors existing `claude/.claude/.gitignore` pattern |
| OS scope | Ubuntu only | Matches current daily-driver host; Fedora/macOS deferred until needed |
| Install mechanism | `pipx:hermes-agent` via mise registry | Same pattern as existing `pipx:mcp-proxy`; avoids the official installer's `~/.bashrc`/`~/.zshrc` edits which would conflict with the stowed `zsh` package |
| Gateway service | Installed, enabled+started conditionally on credential presence | Service is staged unconditionally; ansible only enables it when `hermes gateway setup` has been completed (credential marker present) |
| Missing-credentials behavior | Print a warning, continue play (Option A — notice and continue) | Matches existing playbook tone; reprovisioning is non-blocking |

## Architecture

Two repos cooperate; bootstrap flow is unchanged.

```
┌─────────────────────────────────────────────────────────────────────┐
│  dev-environment (ansible)                                          │
│  ─────────────────────────                                          │
│  local.yml                                                          │
│    ├─ tasks/mise.yml      → mise install picks up pipx:hermes-agent │
│    ├─ tasks/dotfiles.yml  → clones .dotfiles, runs ./ubuntu         │
│    └─ tasks/hermes.yml    → marker checks + conditional service     │
│                                                                     │
│  mise.toml (in .dotfiles, stowed) declares the pipx package         │
└─────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  .dotfiles (stow + systemd)                                         │
│  ──────────────────────────                                         │
│  hermes/.hermes/             (stow package, allow-list .gitignore)  │
│  systemd/units/dotfiles-hermes-gateway.service                      │
│  systemd/setup-all-services.sh   (installs the unit)                │
│  ubuntu (script)             includes "hermes" in STOW_FOLDERS      │
│  README.md                   documents the two manual setup steps   │
└─────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Manual one-time per host (Hermes is an LLM agent w/ user secrets)  │
│  ─────────────────────────────────────────────────────────────────  │
│  1. mise exec -- hermes setup           (provider API keys)         │
│  2. mise exec -- hermes gateway setup   (messaging tokens)          │
│  3. Re-run ./install (or --tags hermes) to enable the service       │
└─────────────────────────────────────────────────────────────────────┘
```

## Components

### 1. mise package declaration (`.dotfiles`)

**File:** `/home/james/.dotfiles/mise/.config/mise/config.toml`

Add to the existing `[tools]` block:

```toml
"pipx:hermes-agent" = "latest"
```

The PyPI package name (`hermes-agent`) is the assumed publish name based on the GitHub repo. Verify on first install — if the actual PyPI name differs, update accordingly. If the project doesn't publish to PyPI, fall back to a git-based pipx install:

```toml
"pipx:hermes-agent" = { version = "latest", install_args = ["git+https://github.com/NousResearch/hermes-agent"] }
```

### 2. Stow package (`.dotfiles/hermes/`)

**Layout:**

```
hermes/
├── .hermes/
│   └── .gitignore     # allow-list
└── README.md          # purpose + gotcha note (optional, short)
```

**`.gitignore`** (allow-list — deny everything by default):

```gitignore
# Default: ignore everything
*

# Explicitly tracked config
!.gitignore
!config.toml
!personality/
!personality/**
!tools/
!tools/**
```

The exact allow-list entries (`config.toml`, `personality/`, `tools/`) are best-guesses based on the README. Refine empirically on first install: run `hermes setup` once, look at what files Hermes wrote into `~/.hermes/`, decide which are non-secret config worth tracking, add `!entries` accordingly.

**Behavior under stow:**
- If `~/.hermes` doesn't yet exist on the host: stow creates `~/.hermes` itself as a symlink to `$DOTFILES/hermes/.hermes/`. Subsequent Hermes runtime writes (logs, sqlite, credentials) land inside the package directory but are kept out of git by the allow-list.
- If `~/.hermes` already exists: stow lays per-file symlinks for each tracked file.
- Either is acceptable.

### 3. Bootstrap wiring (`.dotfiles`)

**`/home/james/.dotfiles/ubuntu`** — append `hermes` to `STOW_FOLDERS`:

```zsh
STOW_FOLDERS="vim,zsh,htop,mise,aws,rider,civitai,comfyui,vibe-kanban,codex,vscode,tmux,tmuxinator,hermes"
```

(Fedora and macOS scripts are not changed — Ubuntu-only scope.)

### 4. Gateway service (hermes-managed)

Hermes manages its own systemd unit. The dev-environment ansible task runs
`hermes gateway install` after gateway credentials are present; that command
creates `~/.config/systemd/user/hermes-gateway.service` and enables it. The
ansible task then ensures the service is started.

No dotfiles-managed wrapper unit is needed; an earlier draft of this design
authored one (`dotfiles-hermes-gateway.service`), but `hermes gateway start`
delegates to `systemctl --user start hermes-gateway.service`, so the wrapper
was both redundant and broken (it tried to start a service that hadn't been
installed yet). Removed during Task 9 validation.

### 5. Ansible task (`dev-environment`)

**New file:** `/home/james/prj/dev-environment/tasks/hermes.yml`

**Wired into `local.yml`** after `tasks/utility-repos.yml` (i.e., after `tasks/dotfiles.yml` has stowed and after `tasks/mise.yml` has installed the pipx package).

**Tags:** `install`, `hermes`. Ubuntu-only via `when: ansible_facts.distribution == "Ubuntu"`.

**Steps (high-level):**

1. **Verify Hermes is installed**
   `command: mise which hermes`, `register: hermes_check`, `failed_when: false`. If absent, debug-print and skip the rest of the file with a block-level `when: hermes_check.rc == 0` guard.

2. **Stat provider-config marker**
   Path: `~/.hermes/<filename>` (TBD on first install). Likely candidate: `~/.hermes/config.toml` or `~/.hermes/providers.json` — whichever `hermes setup` writes.

3. **Stat gateway-config marker**
   Path: `~/.hermes/<filename>` (TBD on first install). Likely candidate: `~/.hermes/gateway.config.json` or similar.

4. **Branch on marker presence:**

   | Provider marker | Gateway marker | Behavior |
   |---|---|---|
   | missing | missing | Two `debug:` warnings printed (provider + gateway). Service stays disabled. |
   | present | missing | One `debug:` warning (gateway). Service stays disabled. |
   | present | present | No warnings. `systemd: name=dotfiles-hermes-gateway.service scope=user enabled=yes state=started`. |
   | missing | present | One `debug:` warning (provider). Service still enabled (corner case; shouldn't normally happen). |

5. **Idempotency:** Each step is stat-checked or `creates:`-guarded; running the play twice on a fully configured host produces no changes.

### 6. Documentation (`.dotfiles/README.md`)

Add a "Hermes Agent" section near the top (above the Backlog.md guidelines block). Content:

- Hermes is installed via mise's pipx registry; `./install` from `dev-environment` provisions it automatically.
- **Two interactive setups required, once per host, before the gateway will run:**
  1. `mise exec -- hermes setup` — enter provider API keys (OpenRouter / Nous Portal / OpenAI / etc.). Required before `hermes` works.
  2. `mise exec -- hermes gateway setup` — enter messaging platform tokens (Telegram / Discord / Slack / etc.). Required before `dotfiles-hermes-gateway.service` can start.
- After completing both, re-run `./install` (or `ansible-playbook local.yml --tags hermes`) to enable the systemd unit.
- Both setups write into `~/.hermes/`. The stow package's allow-list `.gitignore` keeps the resulting credential files out of git.
- Manual control of the gateway service:
  - `systemctl --user start dotfiles-hermes-gateway.service`
  - `systemctl --user stop dotfiles-hermes-gateway.service`
  - `journalctl --user -u dotfiles-hermes-gateway.service -f`

## Open Items (resolved on first install)

These are intentional unknowns; the design names them so they're not silently guessed:

1. **PyPI package name** — verify `hermes-agent` is the actual PyPI name. If not, switch to git-based pipx install.
2. **Provider-config marker filename** — pin after observing what `hermes setup` writes into `~/.hermes/`.
3. **Gateway-config marker filename** — pin after observing what `hermes gateway setup` writes.
4. **Allow-list entries in `.gitignore`** — refine after seeing the actual file layout `hermes` produces in `~/.hermes/`.

The implementation plan should call these out as a "discovery pass" early in the work and pin the values before authoring the final ansible task and `.gitignore`.

## Out of Scope

- Fedora / macOS provisioning of Hermes
- Migrating from OpenClaw (no existing `~/.openclaw` on host)
- Hermes per-conversation overrides (already handled in-product via slash commands)
- Cross-client MCP-style parity (Hermes is an LLM agent, not an MCP server consumed by Claude/Codex)
- Other LLM agent installs

## Validation

After implementation, success means:

1. Fresh `./install` on a clean Ubuntu host completes without manual intervention; `hermes` is on PATH.
2. Stow has linked `~/.hermes/` (either as a directory symlink or per-file symlinks).
3. `dotfiles-hermes-gateway.service` is staged in `~/.config/systemd/user/`.
4. With no credentials present, the play emits warnings naming both setup commands; play exits 0.
5. After running both interactive setup commands and re-running the play, the gateway service is `enabled` and `active (running)`.
6. Re-running `./install` a third time after step 5 produces no changes (idempotency).
