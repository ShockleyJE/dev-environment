# dev-environment

Ansible playbook that provisions this machine. Runs against `localhost`.

```bash
./install                                         # auto-detects OS
ansible-playbook local.yml                        # full play
ansible-playbook local.yml --tags <tag>           # scoped run
ansible-playbook local.yml --check --diff         # dry-run
```

For contributor guidance (style, tests, commit conventions), see
[`AGENTS.md`](AGENTS.md). Stow-managed dotfiles live in `~/.dotfiles/`; see
that repo's [`README.md`](https://github.com/ShockleyJE/.dotfiles) for what
gets symlinked into `$HOME` on a fresh provision.

## Manual setup steps the playbook can't automate

A few interactive credential / device-linking steps must be run by hand,
once per host. The ansible play warns you when each one is needed and
otherwise continues without failure.

### Signal CLI

If you're routing Signal through the Hermes gateway, signal-cli must be
linked to your Signal account before the daemon can run.

`signal-cli link` itself only emits the device-link URI as plain text in
v0.14.3 (upstream PR #1927 would add in-terminal QR rendering but isn't
released yet). Use the `signal:link` mise task instead — it pipes the URI
through `qrencode` to produce a scannable QR:

```bash
# Scan the QR with your phone (Signal → Settings → Linked Devices)
mise run signal:link HermesAgent
```

After the link completes, re-run the play to enable the HTTP daemon:

```bash
ansible-playbook local.yml --tags signal-cli
```

The daemon listens on `localhost:8080` and serves every linked account on
this host, which is the URL `hermes gateway setup` defaults to for Signal.

### Proton Mail Bridge

```bash
# Log in interactively (username, password, 2FA)
protonmail-bridge --cli
```

At the `>>> ` prompt, run `login`, complete the flow, then `quit`.
After login, re-run the play to enable the background service:

```bash
ansible-playbook local.yml --tags proton-bridge
```

Bridge then runs as a user-level systemd unit, exposing IMAP on
`127.0.0.1:1143` and SMTP on `127.0.0.1:1025`. Config is persisted via
the `protonmail` stow package in `~/.dotfiles/`. See
[`~/.dotfiles/README.md`](https://github.com/ShockleyJE/.dotfiles) for
the full stow layout and service controls.
