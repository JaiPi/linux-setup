# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An Ansible playbook that gets the author up and running on any new Linux machine. Debian, Fedora and WSL (Debian on Windows) are all targets; keep every role working on all three. The RedHat/Fedora paths are the least exercised, so check them when changing a role. It installs system packages, manages dotfiles via GNU Stow, sets up Terraform tooling, configures Zsh/Zinit/Starship, and (on WSL) symlinks in Windows-side files.

## Commands

Run the playbook:
```sh
ansible-playbook playbook.yml --ask-become-pass
```

Dry run (check mode):
```sh
ansible-playbook playbook.yml --ask-become-pass --check
```

Verbose (for debugging task failures):
```sh
ansible-playbook playbook.yml --ask-become-pass -vvvv
```

Run a single role:
```sh
ansible-playbook playbook.yml --ask-become-pass --tags stow
```

Lint/validate (also run in CI via [.github/workflows/lint.yml](.github/workflows/lint.yml)):
```sh
ansible-playbook --syntax-check playbook.yml
yamllint .
ansible-lint
```

The playbook prompts interactively for `is_wsl`, `install_zsh`, `windows_user`, `repo`, and `key_file_name` at runtime (see `vars_prompt` in [playbook.yml](playbook.yml)) — each has a default matching the author's own setup, so hitting Enter through all prompts reproduces the original fixed behavior. `is_wsl`/`install_zsh` gate the `wsl`/`zsh` roles; `windows_user`, `repo`, and `key_file_name` feed the `wsl` and `stow` roles respectively.

## Architecture

Standard Ansible layout: `playbook.yml` is the entrypoint, `group_vars/*.yml` hold role configuration (one file per role, all loaded via `vars_files` regardless of which roles actually run), and `roles/*/tasks/main.yml` contain the actual task logic. Role execution order in `playbook.yml` matters:

1. **ping** — sanity check that Ansible can reach the host.
2. **directories** — creates dirs listed in `directories_to_create` ([group_vars/directories.yml](group_vars/directories.yml)), e.g. `~/.local/bin`, `~/repos/jaipi`.
3. **packages** — installs `system_packages` ([group_vars/packages.yml](group_vars/packages.yml)) via `apt`/`dnf` (OS family detected via `ansible_facts['os_family']`), then conditionally installs Terraform tooling (terraform-docs, tfswitch) via `roles/packages/tasks/terraform.yml` when `install_terraform_tools: true`. Both terraform tools check for an existing binary at `~/.local/bin/` before downloading, making the task idempotent/skippable on reruns.
4. **wsl** *(conditional on `is_wsl`)* — symlinks Windows-side dirs and programs (`.aws`, `.kube`, `aws.exe`, `kubectl.exe`) from `/mnt/c/Users/{{ windows_user }}` and `/mnt/c/Program Files/` into the Linux home directory. `.ssh` and git are native Linux, not linked from Windows. `windows_user` comes from the `vars_prompt` in `playbook.yml` (default `jaime`); the rest is configured in [group_vars/wsl.yml](group_vars/wsl.yml).
5. **stow** — clones a *separate*, **public** dotfiles repo (secrets stay in the untracked `~/.zshenv.local`) (URL supplied via the `repo` `vars_prompt`, default `git@github.com:JaiPi/stow.git`) using an SSH key expected at `~/.ssh/{{ key_file_name }}` (`key_file_name` also comes from `vars_prompt`, default `JaiPi`), into `~/repos/jaipi/stow` (`dest_name`, relative to `$HOME`), then runs `stow -d ~/repos/jaipi/stow -t ~ <dir>` for each entry in `stow_dirs` ([group_vars/stow.yml](group_vars/stow.yml): `aws`, `bash`, `claude`, `git`, `ssh`, `tfswitch`, `zsh`). If the SSH key is missing, the role warns and skips the clone rather than failing — this repo has no dotfiles content itself, just the orchestration to pull and symlink them.
6. **zsh** *(conditional on `install_zsh`)* — installs zsh, clones Zinit (plugin manager) to `~/.local/share/zinit/zinit.git`, installs Starship prompt + a Nerd Font (`roles/zsh/tasks/starship.yml`), and chsh's the user to `/bin/zsh`. Actual plugin loading happens in the stowed `.zshrc` (in the separate `stow` repo) via Zinit itself, which self-installs plugins on first shell start — this repo doesn't manage a plugin list.

## Conventions to follow when editing roles

- Guard OS-specific tasks with `ansible_facts['os_family'] in ["Debian", "RedHat"]`, matching existing tasks in `packages` and `zsh` roles. RedHat-specific setup (e.g. `dnf` cache update) goes in `roles/packages/tasks/redhat.yml`, mirroring `debian.yml`.
- Use `stat` + `when: not X.stat.exists` (or `creates:` on `shell`/`command` tasks) to keep download/install tasks idempotent — every existing install task in `packages/terraform.yml` and `zsh/starship.yml` follows this pattern. Shell tasks that pipe `curl | sh` set `set -o pipefail` (via `args: executable: /bin/bash`) so a failed download actually fails the task.
- Use fully-qualified collection names for modules (`ansible.builtin.file`, not `file`) — enforced by `ansible-lint`.
- Each role is tagged with its own name in `playbook.yml` (`ping`, `directories`, `packages`, `wsl`, `stow`, `zsh`), plus `terraform` on the Terraform sub-include in `roles/packages/tasks/main.yml`, so individual roles can be targeted with `--tags`.
- Non-secret, machine-specific values (Windows username, dotfiles repo URL, SSH key filename) are collected via `vars_prompt` in `playbook.yml` with defaults matching the author's own setup, not hardcoded in `group_vars`. If a var is sourced this way, don't duplicate it in a `group_vars/*.yml` file — `vars_prompt` has lower precedence than `vars_files`, so a matching `group_vars` key would silently win and discard the prompt answer.
- Remaining variables live in `group_vars/<role>.yml`, one file per role, all referenced from `playbook.yml`'s top-level `vars_files` (not per-role `vars/main.yml`).
