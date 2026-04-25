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

### Hermes Agent

```bash
# Provider API keys (OpenRouter / Nous Portal / OpenAI / etc.)
mise exec -- hermes setup

# Messaging platform tokens (Telegram / Discord / Slack / etc.)
mise exec -- hermes gateway setup
```

After both, re-run the play to install and start the gateway service:

```bash
ansible-playbook local.yml --tags hermes
```

See `~/.dotfiles/README.md` (the "Hermes Agent" section) for what each
command writes into `~/.hermes/` and how the resulting credentials are
kept out of git.

### Signal CLI

If you're routing Signal through the Hermes gateway, signal-cli must be
linked to your Signal account before the daemon can run:

```bash
# Scan the QR code with your phone (Signal → Settings → Linked Devices)
signal-cli link -n "HermesAgent"
```

After linking, re-run the play to enable the signal-cli HTTP daemon:

```bash
ansible-playbook local.yml --tags signal-cli
```

The daemon listens on `localhost:8080` and serves every linked account on
this host, which is what `hermes gateway` expects.
