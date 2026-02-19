---
icon: lucide/file-code
---

# How to write a Distrorun file

DistroRun uses a declarative *Infrastructure as Code* (IaC) approach. The entire state of your target operating system is defined in a single YAML file. 

This design allows you to version-control your infrastructure, ensure reproducibility, and abstract away the low-level complexities of Linux system building.

---

## Examples

Before diving into the reference, here are two common configurations to get you started.

### 1. Fedora Server
For Fedora, the `type` field is mandatory to determine the base overlay (e.g., `server` vs `workstation`).

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
  - name: user1
    password: "securepassword123"

services:
  enable:
    - nginx

build:
  sbom: true
```

### 2. Fedora Workstation

For a desktop environment, set `type` to `workstation`. This instructs the engine to include the graphical stack and desktop dependencies.

```yaml
version: "1"
name: my-fedora-workstation

distro:
  base: fedora
  type: workstation

packages:
  - gnome-shell
  - firefox
  - vim

users:
  - name: user1
    password: "securepassword123"

services:
  enable:
    - gdm

build:
  sbom: true
```

### 3. Alpine Minimal

For Alpine Linux, the `type` field is omitted. By design, DistroRun treats Alpine as a minimal server environment.

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

build:
  sbom: false
```

---

## Schema Reference

### `version`

**Type:** `string` | **Required:** Yes

The version of your configuration file. This follows semantic versioning and can be any valid version string (e.g., `"1"`, `"1.0"`, `"1.0.12"`).

### `name`

**Type:** `string` | **Required:** Yes

A human-readable name for your build configuration (e.g., `my-fedora-server`, `production-alpine`).

### `distro`

Defines the target operating system and its architecture profile. DistroRun automatically injects the necessary base packages (like the kernel, bootloader, and init system) based on this selection.

| Key | Type | Description |
| --- | --- | --- |
| `base` | `string` | The target distribution. Supported values: `fedora`, `alpine`. |
| `type` | `string` | The system profile. Required for `fedora` (`server` or `workstation`). Ignored for `alpine`. |

### `packages`

**Type:** `list of strings` | **Required:** No

A list of software packages to install via the distribution's native package manager (`dnf` for Fedora, `apk` for Alpine).

!!! note
    Do not include base system dependencies (like `systemd` or `linux-kernel`); the DistroRun engine handles these automatically.

### `users`

**Type:** `list of objects` | **Required:** Yes

Defines the system users. At least one user must be defined to ensure the system is accessible after booting.

| Key | Type | Description |
| --- | --- | --- |
| `name` | `string` | The UNIX username (e.g., `root`, `charif`). |
| `password` | `string` | The plain-text password. |

!!! info "Security Note"
    DistroRun reads the password string and securely hashes it (e.g., via SHA-512) before injecting it into `/etc/shadow`. The plain text is never stored in the final OS image.

### `services`

**Type:** `object` | **Required:** No

Manages the runtime initialization of services. DistroRun acts as an abstraction layer: it translates these instructions into `systemd` symlinks for Fedora, or `OpenRC` runlevels for Alpine.

| Key | Type | Description |
| --- | --- | --- |
| `enable` | `list of strings` | Services to start automatically at boot (e.g., `nginx`). |

### `build`

**Type:** `object` | **Required:** No

Controls the behavior of the DistroRun Go engine during the artifact generation phase.

| Key | Type | Description |
| --- | --- | --- |
| `sbom` | `boolean` | If `true`, the engine scans the final rootfs and generates a Software Bill of Materials (SPDX JSON). Set to `false` for faster, iterative development builds. Defaults to `false` if omitted. |
