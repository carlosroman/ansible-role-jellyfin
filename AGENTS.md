# AGENTS.md — Ansible Role

## What this is

An Ansible role for installing the Jellyfin media server. Standard role layout: `tasks/`, `handlers/`, `defaults/`, `vars/`, `templates/`, `files/`, `meta/`.

The role's Galaxy namespace and name are defined in `meta/main.yml`. The fully-qualified role name (e.g., `<namespace>.<role_name>`) is what you reference in playbooks and `molecule/default/converge.yml`.

## Dev commands

All driven by `make`:

| Command | What it does |
|---|---|
| `make lint` | Runs `yamllint .` |
| `make test` | Runs `molecule test` (single distro, defaults to `ubuntu2404`) |
| `make test/ubuntu-2404` | Test against a specific distro |
| `make test/all` | Test against all 7 supported distros sequentially |

Available distro targets: `ubuntu-2604`, `ubuntu-2404`, `ubuntu-2204`, `debian-13`, `debian-12`, `fedora-44`, `rocky-10`.

Override distro without a make target: `MOLECULE_DISTRO=debian12 molecule test`.

> Note: Debian 11, EL9 and Amazon Linux are not supported — Jellyfin publishes no
> official RPMs (since 10.9), RPM Fusion only builds for Fedora and EL10+, and
> Debian 11 is EOL with its archives removed from deb.debian.org.

## Molecule setup

- **Driver**: Docker (uses `geerlingguy/docker-${MOLECULE_DISTRO}-ansible:latest` images)
- **Config**: `molecule/default/molecule.yml`
- **Converge playbook**: `molecule/default/converge.yml` — references the role by its Galaxy name
- Requires Docker running locally. The devcontainer includes docker-in-docker.

## CI workflow (`.github/workflows/ci.yml`)

Two jobs run on push/PR:
1. **Lint** — `make lint`
2. **Test** — `make test` across matrix: `debian12`, `debian13`, `ubuntu2204`, `ubuntu2404`, `ubuntu2604`, `fedora44`, `rockylinux10`

## Dependencies

Pinned in `pyproject.toml` / `uv.lock`. Install with `uv sync` (`make setup`; the devcontainer does this automatically on create).

## YAML linting

Config in `.yamllint`: extends default, line-length max 120 (warning level). Ignores `venv/` and `.venv/`.

## Conventions

- All YAML files start with `#SPDX-License-Identifier: MIT-0` header
- Standard Ansible role structure; add tasks to `tasks/main.yml`, defaults to `defaults/main.yml`
- `tasks/main.yml` uses `include_tasks: setup-{{ ansible_facts.os_family }}.yml` for OS-family-specific setup
- OS-family setup files: `tasks/setup-Debian.yml` (official Jellyfin APT repo via `deb822_repository`) and `tasks/setup-RedHat.yml` (RPM Fusion free repo via release RPM; asserts supported distro)
- Package install and service management live in `tasks/main.yml` per the conventions docs
- `molecule/default/converge.yml` includes a pre-task for updating apt cache on Debian and post-tasks verifying the `jellyfin` binary and service state

## License

This role is licensed under MIT. See `LICENSE` file for details.

## Repository Management

- **Debian/Ubuntu**: See `CONVENTIONS-DEBIAN.md` for APT repository patterns (uses `deb822_repository`)
- **RedHat/CentOS/Rocky/Amazon Linux**: See `CONVENTIONS-REDHAT.md` for YUM/DNF repository patterns (uses `yum_repository`)

## Commit Messages

This role uses Conventional Commits format. See `.agents/skills/conventional-commits/SKILL.md` for the skill definition.

Trigger with: "commit these changes" or similar requests.
