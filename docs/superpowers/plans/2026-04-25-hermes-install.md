# Hermes Agent Install Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Install Hermes Agent on the user's Ubuntu host via mise's pipx registry, track non-secret config in `.dotfiles` via stow, run the messaging gateway as a user-level systemd unit gated on credential presence.

**Architecture:** Two repos cooperate. `.dotfiles` declares the pipx package in its mise config, ships a stow package and systemd unit, and documents the manual setup steps. `dev-environment`'s ansible play installs the package via mise, includes a new `tasks/hermes.yml` that verifies installation and conditionally enables the gateway service based on credential markers.

**Tech Stack:** mise (pipx backend), GNU stow, ansible, systemd user units, zsh.

**Spec:** `/home/james/prj/dev-environment/docs/superpowers/specs/2026-04-25-hermes-install-design.md`

**Note on TDD:** This is infrastructure/config work; "tests" here are verification commands (`ansible-playbook --check`, `stow -n`, `systemd-analyze verify`, `git check-ignore`, idempotency runs). Each implementation task ends with verification before the commit.

**Note on commits:** Each task ends with a commit step. Commits land in two repos — `.dotfiles` for stow/systemd/README work, `dev-environment` for ansible work. The commit message convention in both repos is concise imperative ("feat:", "fix:", "chore:").

---

## File Map

### `.dotfiles` repo (`/home/james/.dotfiles/`)

| Path | Action | Responsibility |
|---|---|---|
| `mise/.config/mise/config.toml` | Modify | Declare `pipx:hermes-agent` package |
| `hermes/.hermes/.gitignore` | Create | Allow-list for tracked config |
| `hermes/README.md` | Create | Package-level orientation note |
| `ubuntu` | Modify | Add `hermes` to `STOW_FOLDERS` |
| `systemd/units/dotfiles-hermes-gateway.service` | Create | User-level gateway service unit |
| `systemd/setup-all-services.sh` | Modify | Install the new unit |
| `README.md` | Modify | "Hermes Agent" section with manual-step docs |

### `dev-environment` repo (`/home/james/prj/dev-environment/`)

| Path | Action | Responsibility |
|---|---|---|
| `tasks/hermes.yml` | Create | Verify install + conditional service enablement |
| `local.yml` | Modify | Include the new task |

---

## Task 1: Discovery Pass — Verify PyPI name & install Hermes via mise

**Purpose:** Resolve the spec's open items (PyPI package name, marker filenames, allow-list entries) before writing config. Without this, the rest of the plan would be guesswork.

**Files:**
- Modify: `/home/james/.dotfiles/mise/.config/mise/config.toml`

- [ ] **Step 1: Add the pipx package to mise config**

Edit `/home/james/.dotfiles/mise/.config/mise/config.toml` and add this line under the `[tools]` block, alphabetically near the existing `"pipx:mcp-proxy"` entry:

```toml
"pipx:hermes-agent" = "latest"
```

- [ ] **Step 2: Attempt the install**

Run:
```bash
mise install
```

Expected: either succeeds (package exists on PyPI as `hermes-agent`) or fails with "package not found".

- [ ] **Step 3: If PyPI install fails, fall back to git-based pipx install**

If step 2 failed, replace the line in the mise config with:

```toml
"pipx:hermes-agent" = { version = "latest", install_args = ["git+https://github.com/NousResearch/hermes-agent"] }
```

Then re-run:
```bash
mise install
```

Record which form worked in your task notes; the plan continues with whichever succeeded.

- [ ] **Step 4: Verify the `hermes` CLI is on PATH**

Run:
```bash
mise exec -- hermes --version
```

Expected: prints a version. If `hermes` isn't found, the package metadata uses a different entry point — run `pipx list` to find the actual binary name and stop here to reassess.

- [ ] **Step 5: Run `hermes setup` interactively**

Run:
```bash
mise exec -- hermes setup
```

Walk through the prompts. Provide a single provider's API key (whichever you want to use first — OpenRouter or Nous Portal are the easiest). Skip anything you don't have keys for.

- [ ] **Step 6: Run `hermes gateway setup` interactively**

Run:
```bash
mise exec -- hermes gateway setup
```

Walk through the prompts and configure at least one platform (e.g., Telegram with a bot token from @BotFather). If you don't want to configure any platform yet, skip this — but the gateway service won't start until you do.

- [ ] **Step 7: Inventory `~/.hermes/`**

Run:
```bash
ls -la ~/.hermes/
find ~/.hermes/ -maxdepth 3 -type f | sort
```

Record the file list. Identify:
1. **Provider-config marker file** — the file `hermes setup` wrote that contains provider config (e.g., `config.toml`, `providers.json`, etc.). Record its exact path/filename.
2. **Gateway-config marker file** — the file `hermes gateway setup` wrote (e.g., `gateway.config.json`, `gateway/config.toml`). Record its exact path/filename.
3. **Files safe to track in git** — non-secret config like model preferences, personality presets, tool configs.
4. **Files that must NOT be tracked** — anything containing API keys, tokens, conversation history, logs, or runtime caches.

Pin these values in your scratchpad — Tasks 2 and 5 use them.

- [ ] **Step 8: Commit the mise config change**

```bash
cd /home/james/.dotfiles
git add mise/.config/mise/config.toml
git commit -m "feat(mise): declare pipx:hermes-agent package"
```

---

## Task 2: Create the `hermes/` stow package

**Purpose:** Provide a stow package that tracks Hermes' non-secret config and excludes everything else by default.

**Files:**
- Create: `/home/james/.dotfiles/hermes/.hermes/.gitignore`
- Create: `/home/james/.dotfiles/hermes/README.md`

- [ ] **Step 1: Create the package directory**

```bash
mkdir -p /home/james/.dotfiles/hermes/.hermes
```

- [ ] **Step 2: Write the allow-list `.gitignore`**

Create `/home/james/.dotfiles/hermes/.hermes/.gitignore` with:

```gitignore
# Allow-list pattern: everything is ignored by default; explicit entries below
# are tracked. To track a new config file, add a `!path/to/file` line.

*

!.gitignore
```

Then add `!` lines for the safe-to-track files identified in Task 1, Step 7. For example, if Task 1 found that `config.toml` is the provider-config and contains API keys, you would NOT add it. If Task 1 found a `personality/` directory with non-secret personality presets, add:

```gitignore
!personality/
!personality/**
```

If you're unsure whether a file is safe to track, leave it out for now — you can always allow-list it later.

- [ ] **Step 3: Verify the `.gitignore` works as expected**

```bash
cd /home/james/.dotfiles
git check-ignore -v hermes/.hermes/.gitignore
git check-ignore -v hermes/.hermes/some-runtime-file 2>/dev/null || echo "would be ignored"
```

Expected: `.gitignore` itself is **not** ignored (excluded by `!.gitignore`); arbitrary other paths inside `hermes/.hermes/` ARE ignored.

- [ ] **Step 4: Create the package README**

Create `/home/james/.dotfiles/hermes/README.md`:

```markdown
# hermes/ — Hermes Agent stow package

Stows to `~/.hermes/`. Tracks Hermes' non-secret global config; runtime state
and credentials are excluded by an allow-list `.gitignore`.

## Adding a new tracked config file

1. Drop the file into `.hermes/` (or wherever Hermes wants it under `~/.hermes/`)
2. Add an explicit `!filename` line to `.hermes/.gitignore`
3. `cd /home/james/.dotfiles && stow -R hermes` to refresh symlinks

## Manual setup required (per host)

See the "Hermes Agent" section in the top-level `.dotfiles/README.md`.
```

- [ ] **Step 5: Commit**

```bash
cd /home/james/.dotfiles
git add hermes/
git commit -m "feat(hermes): add stow package with allow-list .gitignore"
```

---

## Task 3: Wire the stow package into the Ubuntu bootstrap

**Purpose:** Make `./ubuntu` actually stow the new package.

**Files:**
- Modify: `/home/james/.dotfiles/ubuntu`

- [ ] **Step 1: Add `hermes` to `STOW_FOLDERS`**

Open `/home/james/.dotfiles/ubuntu` and find the line:

```zsh
STOW_FOLDERS="vim,zsh,htop,mise,aws,rider,civitai,comfyui,vibe-kanban,codex,vscode,tmux,tmuxinator"
```

Append `,hermes`:

```zsh
STOW_FOLDERS="vim,zsh,htop,mise,aws,rider,civitai,comfyui,vibe-kanban,codex,vscode,tmux,tmuxinator,hermes"
```

(Do NOT modify the `fedora` script — Ubuntu-only scope.)

- [ ] **Step 2: Dry-run stow to verify no conflicts**

```bash
cd /home/james/.dotfiles
stow -n -v hermes 2>&1
```

Expected output: shows what symlinks `stow` would create. No errors about conflicts. If `~/.hermes/` already exists from Task 1, you'll see per-file link plans (e.g., `LINK: .hermes/.gitignore => ../.dotfiles/hermes/.hermes/.gitignore`). If the directory was empty when stow ran, it may plan to link `~/.hermes` itself as a directory symlink — also acceptable per the spec.

- [ ] **Step 3: Apply stow**

```bash
cd /home/james/.dotfiles
stow hermes
```

Expected: silent success.

- [ ] **Step 4: Verify the symlink(s)**

```bash
ls -la ~/.hermes/.gitignore
readlink ~/.hermes/.gitignore || readlink ~/.hermes
```

Expected: the symlink resolves into `/home/james/.dotfiles/hermes/.hermes/`.

- [ ] **Step 5: Commit**

```bash
cd /home/james/.dotfiles
git add ubuntu
git commit -m "feat(ubuntu): stow hermes package on bootstrap"
```

---

## Task 4: Author the systemd unit

**Purpose:** Define the user-level systemd service that runs the Hermes messaging gateway.

**Files:**
- Create: `/home/james/.dotfiles/systemd/units/dotfiles-hermes-gateway.service`

- [ ] **Step 1: Create the unit file**

Write `/home/james/.dotfiles/systemd/units/dotfiles-hermes-gateway.service`:

```ini
[Unit]
Description=Hermes Agent Messaging Gateway
Documentation=https://github.com/NousResearch/hermes-agent
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
WorkingDirectory=%h
Environment=PATH=%h/.local/bin:%h/.local/share/mise/shims:/usr/local/bin:/usr/bin:/bin
ExecStart=%h/.local/bin/mise exec -- hermes gateway start
Restart=on-failure
RestartSec=30s

[Install]
WantedBy=dotfiles.target
```

- [ ] **Step 2: Verify the unit's syntax**

```bash
systemd-analyze --user verify /home/james/.dotfiles/systemd/units/dotfiles-hermes-gateway.service 2>&1
```

Expected: no output (or only informational messages — anything that says "WARNING" or "ERROR" must be addressed). The unit isn't installed yet, so verification is purely syntactic.

- [ ] **Step 3: Commit**

```bash
cd /home/james/.dotfiles
git add systemd/units/dotfiles-hermes-gateway.service
git commit -m "feat(systemd): add hermes gateway unit"
```

---

## Task 5: Wire the unit into `setup-all-services.sh`

**Purpose:** Make the existing dotfiles bootstrap install the new unit alongside the others.

**Files:**
- Modify: `/home/james/.dotfiles/systemd/setup-all-services.sh`

- [ ] **Step 1: Read the current script to find the conventions**

```bash
cat /home/james/.dotfiles/systemd/setup-all-services.sh
```

Identify the loop or sequence that copies each unit into `~/.config/systemd/user/`. Most installations follow the pattern: `cp` the `.service` file, then `systemctl --user daemon-reload`. Note whether the script `enable`s units; if it does, our new unit must be **excluded** from auto-enable (per the spec, ansible enables it conditionally).

- [ ] **Step 2: Add the copy step for `dotfiles-hermes-gateway.service`**

Add a copy line that mirrors the existing convention but does NOT enable the unit. If the existing pattern is a list, add `dotfiles-hermes-gateway.service` to it; if the script enables every listed unit, refactor minimally so the hermes unit is copied-but-not-enabled. The exact diff depends on the script's current shape — examples:

If the script uses an array of unit names that get both copied and enabled:

```bash
# Existing units that should be copied AND enabled
ENABLE_UNITS=(
  dotfiles-mcp-apply.service
  dotfiles-mcp-proxy-bridges.service
  dotfiles-tmux-sessions.service
  # ... existing entries ...
)

# New: units copied only (enablement is gated by ansible)
COPY_ONLY_UNITS=(
  dotfiles-hermes-gateway.service
)

for unit in "${ENABLE_UNITS[@]}" "${COPY_ONLY_UNITS[@]}"; do
  cp "$DOTFILES/systemd/units/$unit" "$HOME/.config/systemd/user/$unit"
done

systemctl --user daemon-reload

for unit in "${ENABLE_UNITS[@]}"; do
  systemctl --user enable "$unit"
done
```

If the script enables units inline (one block per unit), append a new copy-only block at the bottom:

```bash
# Hermes Gateway: copy unit; ansible enables it once credentials are configured
cp "$DOTFILES/systemd/units/dotfiles-hermes-gateway.service" \
   "$HOME/.config/systemd/user/dotfiles-hermes-gateway.service"
systemctl --user daemon-reload
```

- [ ] **Step 3: Run the script and confirm the unit is staged**

```bash
bash /home/james/.dotfiles/systemd/setup-all-services.sh
ls -la ~/.config/systemd/user/dotfiles-hermes-gateway.service
systemctl --user is-enabled dotfiles-hermes-gateway.service
```

Expected: the file exists in `~/.config/systemd/user/`, and `is-enabled` reports `disabled` (not `enabled`, not `not-found`).

- [ ] **Step 4: Commit**

```bash
cd /home/james/.dotfiles
git add systemd/setup-all-services.sh
git commit -m "feat(systemd): install hermes gateway unit (disabled by default)"
```

---

## Task 6: Author the ansible task `tasks/hermes.yml`

**Purpose:** Verify Hermes is installed; warn if credentials are missing; conditionally enable+start the gateway service.

**Files:**
- Create: `/home/james/prj/dev-environment/tasks/hermes.yml`

> **Pre-req from Task 1, Step 7:** Substitute the actual filenames you discovered for `<PROVIDER_MARKER>` and `<GATEWAY_MARKER>`. Examples (most likely):
> - `<PROVIDER_MARKER>` → `~/.hermes/config.toml` (or `config.json`)
> - `<GATEWAY_MARKER>` → `~/.hermes/gateway.config.json` (or `gateway/config.toml`)
>
> If unsure, pick the file most likely to exist only AFTER `hermes setup` / `hermes gateway setup` completed successfully — not a default-installed file.

- [ ] **Step 1: Write the task file**

Create `/home/james/prj/dev-environment/tasks/hermes.yml`:

```yaml
- name: Verify hermes CLI is on PATH (Ubuntu)
  ansible.builtin.command: mise which hermes
  register: hermes_which
  failed_when: false
  changed_when: false
  when: ansible_facts.distribution == "Ubuntu"
  tags:
    - install
    - hermes

- name: Warn if hermes is not installed (Ubuntu)
  ansible.builtin.debug:
    msg: |
      hermes is not on PATH. Ensure mise.yml task ran and that the
      pipx:hermes-agent declaration in ~/.dotfiles/mise/.config/mise/config.toml
      is correct. Skipping the rest of tasks/hermes.yml.
  when:
    - ansible_facts.distribution == "Ubuntu"
    - hermes_which.rc != 0
  tags:
    - install
    - hermes

- name: Stat provider-config marker (Ubuntu)
  ansible.builtin.stat:
    path: "{{ lookup('env', 'HOME') }}/.hermes/<PROVIDER_MARKER>"
  register: hermes_provider_marker
  when:
    - ansible_facts.distribution == "Ubuntu"
    - hermes_which.rc == 0
  tags:
    - install
    - hermes

- name: Stat gateway-config marker (Ubuntu)
  ansible.builtin.stat:
    path: "{{ lookup('env', 'HOME') }}/.hermes/<GATEWAY_MARKER>"
  register: hermes_gateway_marker
  when:
    - ansible_facts.distribution == "Ubuntu"
    - hermes_which.rc == 0
  tags:
    - install
    - hermes

- name: Warn if provider config is missing (Ubuntu)
  ansible.builtin.debug:
    msg: |
      Hermes provider config not found at ~/.hermes/<PROVIDER_MARKER>.
      Run this once interactively to populate provider API keys:

          mise exec -- hermes setup

      Then re-run: ansible-playbook local.yml --tags hermes
  when:
    - ansible_facts.distribution == "Ubuntu"
    - hermes_which.rc == 0
    - not hermes_provider_marker.stat.exists
  tags:
    - install
    - hermes

- name: Warn if gateway config is missing (Ubuntu)
  ansible.builtin.debug:
    msg: |
      Hermes gateway config not found at ~/.hermes/<GATEWAY_MARKER>.
      The dotfiles-hermes-gateway.service will remain disabled until
      messaging platform tokens are configured. Run once interactively:

          mise exec -- hermes gateway setup

      Then re-run: ansible-playbook local.yml --tags hermes
  when:
    - ansible_facts.distribution == "Ubuntu"
    - hermes_which.rc == 0
    - not hermes_gateway_marker.stat.exists
  tags:
    - install
    - hermes

- name: Enable and start hermes gateway service (Ubuntu)
  ansible.builtin.systemd:
    name: dotfiles-hermes-gateway.service
    scope: user
    enabled: true
    state: started
    daemon_reload: true
  when:
    - ansible_facts.distribution == "Ubuntu"
    - hermes_which.rc == 0
    - hermes_gateway_marker.stat.exists
  tags:
    - install
    - hermes
```

- [ ] **Step 2: Lint the file**

```bash
cd /home/james/prj/dev-environment
ansible-playbook --syntax-check tasks/hermes.yml 2>&1 || true
ansible-lint tasks/hermes.yml 2>&1 || true
```

Expected: syntax-check passes (note: `--syntax-check` on a tasks file may complain it's not a play; that's expected — what matters is no YAML parse errors). `ansible-lint` may produce style notes; address any that flag actual errors.

- [ ] **Step 3: Commit**

```bash
cd /home/james/prj/dev-environment
git add tasks/hermes.yml
git commit -m "feat(ansible): add hermes task with marker-gated service enablement"
```

---

## Task 7: Wire the ansible task into `local.yml`

**Purpose:** Include `tasks/hermes.yml` in the main play so `./install` runs it.

**Files:**
- Modify: `/home/james/prj/dev-environment/local.yml`

- [ ] **Step 1: Insert the include after `tasks/utility-repos.yml`**

Open `/home/james/prj/dev-environment/local.yml`. Find the line:

```yaml
    - include_tasks: tasks/utility-repos.yml
```

Add a new line directly after it:

```yaml
    - include_tasks: tasks/utility-repos.yml
    - include_tasks: tasks/hermes.yml
      tags:
        - install
        - hermes
```

(Per-include `tags:` mirrors the pattern used elsewhere in `local.yml` — see the `projects.yml` include for an example. The internal task tags inside `tasks/hermes.yml` provide finer-grained control; the include-level tag lets `--tags hermes` target the whole file.)

- [ ] **Step 2: Dry-run the play to confirm wiring**

```bash
cd /home/james/prj/dev-environment
ansible-playbook local.yml --check --diff --tags hermes
```

Expected: the play runs through the hermes tasks. With credentials missing, you should see the two warning debug messages. With credentials present, you should see a "would change" line for the systemd enablement (or no change if already enabled).

- [ ] **Step 3: Commit**

```bash
cd /home/james/prj/dev-environment
git add local.yml
git commit -m "feat(ansible): include hermes task in local.yml"
```

---

## Task 8: Update the `.dotfiles` README

**Purpose:** Document the manual setup steps so a future-you (or anyone provisioning a new host) knows the gateway service requires interactive credential entry.

**Files:**
- Modify: `/home/james/.dotfiles/README.md`

- [ ] **Step 1: Add the "Hermes Agent" section**

Open `/home/james/.dotfiles/README.md`. Insert a new section directly above the `<!-- BACKLOG.MD GUIDELINES START -->` marker (or, if no such marker exists, at the natural top-level position alongside the other tool sections like "Claude Code Integration"):

````markdown
## Hermes Agent

Hermes is installed via mise's pipx registry — `./install` from `dev-environment`
provisions the `hermes` CLI automatically. The messaging gateway runs as a
user-level systemd unit (`dotfiles-hermes-gateway.service`).

### Required manual setup (once per host)

The gateway will not start successfully until both of the following have been
run interactively to populate `~/.hermes/`:

```bash
# 1. Provider API keys (OpenRouter / Nous Portal / OpenAI / etc.)
mise exec -- hermes setup

# 2. Messaging platform tokens (Telegram / Discord / Slack / etc.)
mise exec -- hermes gateway setup
```

After completing both, re-run the dev-environment play to enable the service:

```bash
cd ~/prj/dev-environment
ansible-playbook local.yml --tags hermes
```

Both setup commands write into `~/.hermes/`. The `hermes/` stow package's
allow-list `.gitignore` keeps the resulting credential files out of git.

### Manual gateway controls

```bash
systemctl --user start  dotfiles-hermes-gateway.service
systemctl --user stop   dotfiles-hermes-gateway.service
systemctl --user status dotfiles-hermes-gateway.service
journalctl --user -u    dotfiles-hermes-gateway.service -f
```
````

- [ ] **Step 2: Commit**

```bash
cd /home/james/.dotfiles
git add README.md
git commit -m "docs(hermes): document manual setup procedure"
```

---

## Task 9: End-to-end validation

**Purpose:** Confirm the full bootstrap works on the configured host, with and without credentials.

- [ ] **Step 1: Run the dev-environment play with hermes tag, dry-run**

```bash
cd /home/james/prj/dev-environment
ansible-playbook local.yml --check --diff --tags hermes
```

Expected: zero failures. If credentials were already entered in Task 1, no warnings; otherwise two warnings.

- [ ] **Step 2: Run the play for real**

```bash
cd /home/james/prj/dev-environment
ansible-playbook local.yml --tags hermes
```

Expected: zero failures. If both credential markers exist, the systemd task should report `changed` on the first run and ok on subsequent runs.

- [ ] **Step 3: Confirm gateway service state**

If both credentials were configured:

```bash
systemctl --user is-active dotfiles-hermes-gateway.service
systemctl --user is-enabled dotfiles-hermes-gateway.service
journalctl --user -u dotfiles-hermes-gateway.service --since "1 minute ago"
```

Expected: `active`, `enabled`, journal shows the gateway booting up. If the gateway logs an error about missing credentials for a specific platform, that's fine — it just means the platform needs a working token; the unit itself is correctly wired.

If credentials were NOT configured:

```bash
systemctl --user is-enabled dotfiles-hermes-gateway.service
```

Expected: `disabled`.

- [ ] **Step 4: Idempotency check**

```bash
cd /home/james/prj/dev-environment
ansible-playbook local.yml --tags hermes
```

Expected: `changed=0` in the play recap. If anything reports `changed`, identify why and either fix the task or accept it as expected churn (e.g., systemd `daemon_reload: true` may always report changed in some ansible versions — that's OK).

- [ ] **Step 5: Verify .dotfiles git status is clean of secrets**

```bash
cd /home/james/.dotfiles
git status hermes/
```

Expected: empty (no untracked or modified files). If anything appears in `hermes/.hermes/` other than the tracked `.gitignore` and your explicit allow-listed entries, double-check the `.gitignore` is denying it correctly.

- [ ] **Step 6: Final smoke test of the CLI**

```bash
mise exec -- hermes --help
mise exec -- hermes doctor
```

Expected: `--help` works; `doctor` reports no critical errors (warnings about un-configured platforms are fine).

- [ ] **Step 7: Final commit (if any cleanup happened)**

If validation surfaced fixes, commit them with a message like:

```bash
git commit -m "fix(hermes): <what you fixed>"
```

If no fixes were needed, skip this step.

---

## Self-Review Notes

- **Spec coverage:** Every spec component is mapped to a task. Discovery pass (Task 1), stow package (Task 2), bootstrap wiring (Task 3), systemd unit (Task 4 + 5), ansible task (Task 6 + 7), README (Task 8). Validation (Task 9) covers the spec's "Validation" section verbatim.
- **Open items resolved:** Task 1's discovery pass pins all four spec-level unknowns (PyPI name, two marker filenames, allow-list entries) before Tasks 2 and 6 lock them into config.
- **No placeholders that block execution:** `<PROVIDER_MARKER>` and `<GATEWAY_MARKER>` are explicit substitution slots filled by Task 1's output — not "TODO" content. The plan refuses to author the ansible task or .gitignore until those values are real.
- **Cross-repo commits are explicit:** Each task labels which repo (`/home/james/.dotfiles` vs `/home/james/prj/dev-environment`) the commit lands in.
- **Idempotency:** Task 9 Step 4 verifies it; the ansible task itself is `creates:`-/stat-guarded throughout.
