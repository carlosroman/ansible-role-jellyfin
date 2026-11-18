Jellyfin
========

Ansible role that installs and manages [Jellyfin](https://jellyfin.org), the free
and open source media server.

Installation sources
--------------------

Jellyfin stopped publishing official RPM packages after version 10.9, so this
role uses different repositories depending on the platform:

| Platform | Source |
|---|---|
| Debian 12/13, Ubuntu 22.04+ | Official Jellyfin APT repository (`repo.jellyfin.org`), same as `install-debuntu.sh` |
| Fedora, RHEL/Rocky/AlmaLinux 10+ | Community packages from [RPM Fusion](https://rpmfusion.org) (free repository) |
| Debian 11, EL9, Amazon Linux | Not supported — no usable package sources exist; use a [container](https://jellyfin.org/docs/general/installation/container/) instead |

Requirements
------------

- ansible-core >= 2.15 (for `ansible.builtin.deb822_repository`)
- Target reachable via SSH with privilege escalation

Role Variables
--------------

| Variable | Default | Description |
|---|---|---|
| `jellyfin_package` | `jellyfin` | Package name to install |
| `jellyfin_package_state` | `present` | Package state (`present`, `latest`, ...) |
| `jellyfin_package_version` | `""` | Pin an exact package version (empty = latest) |
| `jellyfin_service_name` | `jellyfin` | Service name |
| `jellyfin_service_state` | `started` | Desired service state |
| `jellyfin_service_enabled` | `true` | Enable service at boot |
| `jellyfin_apt_repo_os` | derived | APT repo component: `debian` or `ubuntu` |
| `jellyfin_apt_release` | distro codename | APT repo suite (codename of the upstream release) |
| `jellyfin_apt_uri` | `https://repo.jellyfin.org/<os>` | Jellyfin APT repository URI |
| `jellyfin_apt_gpg_key` | official key URL | GPG key URL used to sign the APT repository |
| `jellyfin_rpmfusion_release_rpm` | derived | RPM Fusion free release package URL (Fedora or EL variant) |
| `jellyfin_rpmfusion_gpg_key` | derived | RPM Fusion GPG key URL used to sign the release package |

Dependencies
------------

None.

Example Playbook
----------------

```yaml
- name: Install Jellyfin media server
  hosts: media_servers
  become: true
  roles:
    - role: carlosroman.jellyfin
```

License
-------

MIT
