---
icon: lucide/file-code
---

# How to write a Distrorun file

DistroRun uses a declarative *Infrastructure as Code* (IaC) approach. The entire state of your target operating system is defined in a single YAML file.

This design lets you version-control your infrastructure, ensure reproducibility, and abstract away the low-level complexities of Linux system building.

---

## Examples

A few common configurations to get you started. The right one depends on what you want to do with the artifact — see [Choosing an output format](#choosing-an-output-format) below.

### 1. Fedora Server (live ISO)

Builds a bootable live ISO of headless Fedora.

```yaml
version: "1"
name: my-fedora-server

distro:
  base: fedora
  type: server

packages:
  - nginx
  - vim
  - curl

users:
  - name: admin
    password: "securepassword123"

services:
  enable:
    - nginx
    - sshd
    - NetworkManager

build:
  sbom: true
```

### 2. Fedora Workstation as a daily-driver VM

This is the right shape for "I want to use it in virt-manager or VMware as a normal OS". `build.output: disk` produces a qcow2 disk image — import it directly into virt-manager, or convert to vmdk for VMware with `qemu-img convert -O vmdk MyOS.qcow2 MyOS.vmdk`.

```yaml
version: "1"
name: bestOS_in_the_world

distro:
  base: fedora
  type: workstation

users:
  - name: admin
    password: admin123
  - name: root
    password: toor

packages:
  - vim
  - curl
  - htop

services:
  enable:
    - NetworkManager
    - gdm
    - sshd

build:
  output: disk
  disk_size: 20G
  sbom: false
```

The engine auto-installs GNOME Shell, GDM, Firefox, and a working network stack when `type: workstation`. Don't list those in `packages` yourself.

### 3. Alpine Minimal (live ISO)

For Alpine, `type` is omitted — Alpine is always a minimal server.

```yaml
version: "1"
name: my-alpine-server

distro:
  base: alpine

packages:
  - nginx

users:
  - name: root
    password: "rootpassword"

services:
  enable:
    - nginx
    - networking

build:
  sbom: false
```

### 4. Persistent USB image

`build.output: img` produces a raw disk image with two partitions: boot (squashfs + GRUB) and a `DISTRORUN_PERS`-labeled ext4 partition. The live init mounts the second partition as the overlay upper layer, so changes survive reboots.

```yaml
version: "1"
name: my-fedora-usb

distro:
  base: fedora
  type: server

packages:
  - vim
  - htop

users:
  - name: admin
    password: changeme

services:
  enable:
    - sshd
    - NetworkManager

build:
  output: img
  persist_size: 4G
```

Write to USB with `dd if=my-fedora-usb.img of=/dev/sdX bs=4M status=progress`.

---

## Choosing an output format

| You want... | Set `build.output` to | Notes |
|---|---|---|
| Try a config in QEMU temporarily | `iso` (or omit `build.output`) | Default. No persistence; tmpfs overlay. |
| Daily-driver VM (virt-manager / VMware) | `disk` | qcow2; full installed system. Set `disk_size` (default `4G`). |
| Persistent live USB | `img` | Two-partition raw image. Set `persist_size` (default `2G`). |
| Server appliance image | `disk` with `type: server` | Headless qcow2 you can clone. |

---

## Schema Reference

### `version`

**Type:** `string` | **Required:** Yes

The version of your configuration file. Any non-empty string is accepted (e.g. `"1"`, `"1.0"`, `"1.0.12"`).

### `name`

**Type:** `string` | **Required:** Yes

A human-readable name for your build. Used as the default output filename and, if you don't define users, as the system hostname.

### `distro`

Defines the target operating system and its profile. DistroRun automatically injects the necessary base packages (kernel, bootloader, init system) based on this selection.

| Key | Type | Description |
| --- | --- | --- |
| `base` | `string` | The target distribution. Supported values: `fedora`, `alpine`. |
| `type` | `string` | The system profile. For Fedora: `server` (default, headless) or `workstation` (GNOME desktop). Ignored for Alpine. |

### `packages`

**Type:** `list of strings` | **Required:** No

Packages to install via the distribution's native package manager (`dnf` for Fedora, `apk` for Alpine).

!!! note
    Don't list base system dependencies (`systemd`, kernel, NetworkManager, openrc, openssh-server) — the engine handles those automatically based on `distro.base` and `distro.type`.

### `users`

**Type:** `list of objects` | **Required:** Yes

Defines the system users. At least one user must be defined to ensure the system is accessible after booting.

| Key | Type | Description |
| --- | --- | --- |
| `name` | `string` | The UNIX username (e.g., `root`, `admin`). The first user's name is also written to `/etc/hostname`. |
| `password` | `string` | The plain-text password. |

!!! info "Security Note"
    DistroRun reads the password string and hashes it via `chpasswd` (SHA-512) before injecting it into `/etc/shadow`. The plain text is never stored in the final OS image — it only exists briefly during the chroot phase in `/tmp`.

!!! tip "Sudo on Fedora"
    Non-root users on Fedora are added to the `wheel` group automatically. The default `/etc/sudoers` rule (`%wheel ALL=(ALL) ALL`) means they can `sudo` immediately, with their account password.

### `services`

**Type:** `object` | **Required:** No

Manages which services start at boot. DistroRun translates this into `systemctl enable` for Fedora or `rc-update add ... default` for Alpine.

| Key | Type | Description |
| --- | --- | --- |
| `enable` | `list of strings` | Services to start at boot. Use systemd unit names on Fedora (`sshd`, `gdm`, `NetworkManager`). Use OpenRC service names on Alpine (`sshd`, `networking`). |

### `build`

**Type:** `object` | **Required:** No

Controls artifact generation.

| Key | Type | Description |
| --- | --- | --- |
| `output` | `string` | One of `iso` (default), `disk` (qcow2), `img` (USB image with persistence). |
| `disk_size` | `string` | Disk size for `output: disk`. Format `<int>G` or `<int>M` (e.g. `8G`). Default `4G`. |
| `persist_size` | `string` | Persistence partition size for `output: img`. Format `<int>G` or `<int>M`. Default `2G`. |
| `sbom` | `boolean` | If `true`, generates an SPDX 2.3 JSON Software Bill of Materials. Uses Trivy when available; falls back to an apk scanner on Alpine. Default `false`. |
