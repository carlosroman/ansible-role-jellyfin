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
| `jellyfin_hardware_acceleration` | `""` | Hardware acceleration type (`vaapi`, `qsv`, `nvenc`, `amf`, `v4l2m2m`, `videotoolbox`, `rkmpp`, `none`); empty means the role does not touch the encoding configuration |
| `jellyfin_config_dir` | `/etc/jellyfin` | Server configuration directory, from the package's `JELLYFIN_CONFIG_DIR`; `encoding.xml` is rendered here |
| `jellyfin_vaapi_device` | `/dev/dri/renderD128` | VA-API render node to transcode on |
| `jellyfin_allow_hevc_encoding` | `false` | Allow HEVC hardware encoding |
| `jellyfin_allow_av1_encoding` | `false` | Allow AV1 hardware encoding |
| `jellyfin_hardware_decoding_codecs` | `["h264", "hevc"]` | Codecs decoded on the GPU (empty list = decode on the CPU) |
| `jellyfin_tonemapping` | `""` | HDR tone mapping: `""` (off), `opencl`, or `vpp` |
| `jellyfin_install_gpu_packages` | `true` | Install the Debian-family GPU runtime packages along with the opt-in |
| `jellyfin_gpu_packages_debian` | derived | Those packages (`intel-media-va-driver-non-free`, `vainfo`, plus `intel-opencl-icd` for OpenCL tone mapping) |

Hardware acceleration
---------------------

A fresh Jellyfin has hardware encoding switched off, and the only way to turn it
on has been the web dashboard — which means the setting is invisible to
configuration management and disappears with the host. This role therefore
offers an opt-in: leave `jellyfin_hardware_acceleration` empty (the default) and
the role neither writes `encoding.xml` nor installs driver packages, exactly as
before the option existed. Set it and the role renders the hardware-relevant
subset of `encoding.xml`, installs the Debian-family GPU runtime packages
(`jellyfin_install_gpu_packages: false` opts out of that part), and restarts
Jellyfin when either changes.

The values are deliberately not defaulted for you: which codecs a GPU can encode
and whether its driver implements VPP tone mapping depend on the exact hardware,
so the role ships the mechanism and your inventory ships the policy.

```yaml
- name: Install Jellyfin media server with VA-API transcoding
  hosts: media_servers
  become: true
  vars:
    jellyfin_hardware_acceleration: vaapi
    jellyfin_vaapi_device: /dev/dri/renderD128
    jellyfin_allow_hevc_encoding: true
    jellyfin_hardware_decoding_codecs:
      - h264
      - hevc
    jellyfin_tonemapping: opencl
  roles:
    - role: carlosroman.jellyfin
```

Things worth knowing before you pick values:

- **An encoder the GPU does not have is a failure, not a fallback.** Enabling
  `jellyfin_allow_av1_encoding` on hardware without an AV1 encoder (Alder Lake-N,
  for instance, decodes AV1 but cannot encode it) makes Jellyfin emit
  `av1_vaapi` transcodes that exit with an error instead of falling back to
  software. Check your GPU, not the dashboard's list of checkboxes.
- **VPP tone mapping needs driver support.** Intel's iHD driver does not
  implement `tonemap_vaapi` as of 26.1.2, and Jellyfin answers
  `EnableVppTonemapping=true` by quietly using the OpenCL chain instead. OpenCL
  tone mapping works wherever an OpenCL ICD exists, which is why the role
  installs `intel-opencl-icd` when `jellyfin_tonemapping` is `opencl`: without an
  ICD, tone mapping is silently dead.
- **The Debian-family packages need the right repository components.**
  `intel-media-va-driver-non-free` and `intel-opencl-icd` live in `non-free`
  (Debian) or `universe` (Ubuntu). Where you cannot enable them, set
  `jellyfin_install_gpu_packages: false` and provide the driver stack yourself.
- **The RedHat family renders the configuration only.** RPM Fusion's GPU driver
  packaging varies by release and vendor (`libva-intel-media-driver` on Fedora,
  no Intel OpenCL ICD on EL), so that family gets the file and keeps its driver
  choices with the operator.
- **Inside a container, the device has to be usable by the `jellyfin` user.** A
  passed-through render node is created `root:root 0660`, so grant the daemon's
  group access (in LXC, the `gid:` of the device entry) and verify with
  `su -s /bin/sh jellyfin -c vainfo`. If the device is missing or unusable
  Jellyfin still starts and software-transcodes, which is what the role's
  hardware scenario asserts (`make test/hardware`, run with no `/dev/dri` at
  all). Monitoring with `intel_gpu_top` belongs on the host: its perf PMU is not
  reachable from an unprivileged container.
- **The dashboard stops being the source of truth for these settings.** The
  rendered file carries a comment saying so, and an edit to one of them in the UI
  is reverted by the next converge — drift becomes an Ansible diff instead of a
  silent regression found at the next rebuild. Only these elements are managed:
  `HardwareAccelerationType`, `VaapiDevice`, `EnableHardwareEncoding`,
  `AllowHevcEncoding`, `AllowAv1Encoding`, `EnableTonemapping`,
  `EnableVppTonemapping`, `HardwareDecodingCodecs`. Jellyfin's own serializer
  supplies defaults for every element the file omits, so the managed set stays
  small and every other dashboard setting keeps its value. Note that Jellyfin
  re-saves the complete option document shortly after it starts, so the file you
  see in the steady state is that expanded document carrying these values.
- **Turning the opt-in off again leaves the file in place.** The role never
  deletes `encoding.xml`, so remove the variable only when you are content for
  the dashboard to own those settings again.

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
