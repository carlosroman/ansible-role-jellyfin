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
| `make test/hardware` | Test the hardware acceleration opt-in (`converge-hardware.yml`) |
| `make test/hardware/ubuntu-2604` | That opt-in scenario against Ubuntu 26.04 |
| `make test/all` | Test against all 7 supported distros sequentially |

Available distro targets: `ubuntu-2604`, `ubuntu-2404`, `ubuntu-2204`, `debian-13`, `debian-12`, `fedora-44`, `rocky-10`.

Override distro without a make target: `MOLECULE_DISTRO=debian12 molecule test`.

> Note: Debian 11, EL9 and Amazon Linux are not supported — Jellyfin publishes no
> official RPMs (since 10.9), RPM Fusion only builds for Fedora and EL10+, and
> Debian 11 is EOL with its archives removed from deb.debian.org.

> Note: a role installed under `~/.ansible/roles/carlosroman.jellyfin` (for
> example by `ansible-galaxy role install`) shadows the working tree, because the
> playbooks reference the role by its Galaxy name. Remove that copy, or export
> `ANSIBLE_ROLES_PATH` pointing at a directory containing a
> `carlosroman.jellyfin` symlink to this repository, so local runs converge the
> code you are editing.

## Molecule setup

- **Driver**: Docker (uses `geerlingguy/docker-${MOLECULE_DISTRO}-ansible:latest` images)
- **Config**: `molecule/default/molecule.yml`
- **Converge playbook**: `molecule/default/converge.yml` — references the role by its Galaxy name
- **Hardware opt-in playbook**: `molecule/default/converge-hardware.yml` — the same
  role with the acceleration opt-in enabled (VA-API, HEVC, OpenCL tone mapping);
  asserts the rendered `encoding.xml` and its ownership, the driver packages on
  the Debian family, and that Jellyfin still runs although the container has no
  `/dev/dri`. Selected through `MOLECULE_PLAYBOOK`, so `molecule.yml` and the CI
  matrix are unchanged.
- Requires Docker running locally. The devcontainer includes docker-in-docker.

## CI workflow (`.github/workflows/ci.yml`)

Two jobs run on push/PR:
1. **Lint** — `make lint`
2. **Test** — `make test` across matrix: `debian12`, `debian13`, `ubuntu2204`, `ubuntu2404`, `ubuntu2604`, `fedora44`, `rockylinux10`

The hardware opt-in scenario is not part of that matrix: its driver packages come
from `non-free` (Debian) and `universe` (Ubuntu), which the plain Debian container
images do not enable. Run it with `make test/hardware` on an Ubuntu distro.

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
- Hardware acceleration is opt-in and every task in that block is guarded by `jellyfin_hardware_acceleration`. Keep it that way: an unset opt-in must install no packages and write no files, and the values that work are per-GPU policy for the inventory, not a role default
- `templates/encoding.xml.j2` renders only the encoding elements the role owns and `vars/main.yml` mirrors them in `jellyfin_encoding_managed_elements`; `tasks/main.yml` compares that list with the file on disk instead of trusting the template diff, because Jellyfin re-saves the complete option document a few seconds after it starts. Keep the two in sync (the hardware scenario asserts the values, so drift between them fails the test)
- `handlers/main.yml` holds `Restart Jellyfin`, notified by the encoding template: GPU-related package and configuration changes only reach a running Jellyfin after a restart
- `molecule/default/converge.yml` includes a pre-task for updating apt cache on Debian and post-tasks verifying the `jellyfin` binary and service state

## License

This role is licensed under MIT. See `LICENSE` file for details.

## Repository Management

- **Debian/Ubuntu**: See `CONVENTIONS-DEBIAN.md` for APT repository patterns (uses `deb822_repository`)
- **RedHat/CentOS/Rocky/Amazon Linux**: See `CONVENTIONS-REDHAT.md` for YUM/DNF repository patterns (uses `yum_repository`)

## Commit Messages

This role uses Conventional Commits format. See `.agents/skills/conventional-commits/SKILL.md` for the skill definition.

Trigger with: "commit these changes" or similar requests.
