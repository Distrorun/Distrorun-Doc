---
icon: lucide/square-terminal
---

# CLI Reference

The DistroRun CLI is your primary tool for interacting with the Go orchestration engine. It validates configurations, triggers system builds, manages a registry of configs, and launches QEMU for testing.

## Top-level commands

```text
distrorun build <config.yaml> [-o output] [-d|--debug]
distrorun test  <iso|img|qcow2> [-r RAM_MB] [-d DISK_SIZE]
distrorun login   [--server URL]
distrorun logout
distrorun push  <config.yaml> [--sbom <file>] [--server URL]
distrorun pull  <name>             [--server URL]
distrorun version
distrorun help
```

---

## distrorun build

The core command. Reads your YAML, sets up a chroot, fetches packages, applies configuration, and produces a bootable artifact (live ISO, qcow2 disk image, or USB image with persistence — based on `build.output` in the YAML).

!!! warning "Privilege Escalation"
    `build` mounts pseudo-filesystems and chroots into the rootfs. It must be run as root.

### Usage

```bash
sudo distrorun build <config.yaml> [-o <output>] [-d|--debug]
```

### Flags

* `-o <path>` — Override the default output filename. Default is `<name>.iso`, `<name>.qcow2`, or `<name>.img` depending on `build.output`.
* `-d`, `--debug` — Show full package-manager output (`dnf` / `apk`). Without this flag, dnf/apk run in quiet mode and progress is shown via a spinner.

### Examples

```bash
# Live ISO, quiet output (default)
sudo distrorun build my-fedora.yaml

# Same, but stream every dnf/apk line for troubleshooting
sudo distrorun build my-fedora.yaml -d

# Custom output path
sudo distrorun build my-fedora.yaml -o /tmp/test.iso
```

The build pipeline runs as 8 or 9 steps (9 if `build.sbom: true`). Each step prints a `[N/total]` header.

### Output formats

What you get depends on `build.output` in your YAML:

| `build.output` | Artifact | Use it for |
|---|---|---|
| `iso` (default) | Bootable live CD/DVD | Trying a config; live demo. No persistence by default. |
| `disk` | qcow2 disk image | virt-manager / QEMU / VMware (after vmdk conversion). Full installed system. |
| `img` | Raw USB disk image | `dd` to a USB stick. Persistent via the `DISTRORUN_PERS` partition. |

---

## distrorun test

Launches a QEMU virtual machine to test a generated artifact without writing to physical media.

!!! info "Dependency"
    Requires `qemu-system-x86_64`. KVM is auto-detected — if `-enable-kvm` fails, DistroRun retries without hardware acceleration.

### Usage

```bash
distrorun test <iso-file> [flags]
```

### Flags

* `-r <MB>` — RAM in MB. Default `512`. (For workstation/GNOME images, use at least `2048`.)
* `-d <SIZE>` — Create and attach a virtual qcow2 disk of the given size (`8G`, `20G`). The disk file is reused between runs, so changes persist across `distrorun test` invocations.

### Examples

```bash
# Basic test
distrorun test my-alpine-server.iso

# Workstation image with 4 GB RAM
distrorun test my-fedora-workstation.iso -r 4096

# Test with a 10 GB persistent disk attached
distrorun test my-fedora.iso -r 2048 -d 10G
```

!!! note
    When `-d` is used, the disk is created at `<iso>-disk.qcow2`. Delete that file to start fresh on the next run.

---

## Registry commands

DistroRun ships with built-in commands to push and pull YAML configs (and their SBOMs) from a remote registry.

### distrorun login

Saves credentials for a registry server. Defaults to `http://localhost:3000`.

```bash
distrorun login                              # default server
distrorun login --server https://registry.example.com
```

### distrorun logout

Forgets stored credentials.

```bash
distrorun logout
```

### distrorun push

Uploads a config (and optionally its SBOM) to the registry.

```bash
distrorun push my-fedora.yaml
distrorun push my-fedora.yaml --sbom my-fedora-sbom.spdx.json
distrorun push my-fedora.yaml --server https://registry.example.com
```

### distrorun pull

Downloads a config from the registry by name.

```bash
distrorun pull my-fedora
distrorun pull my-fedora --server https://registry.example.com
```

---

## Booting the artifact in a VM

After `build`, you'll have one of three files. Here's how to use each.

### qcow2 (`build.output: disk`)

**virt-manager:**

```text
File → New Virtual Machine → Import existing disk image
  → Browse to MyOS.qcow2
  → OS: Fedora Linux (or Alpine Linux)
  → Memory: 4096 MB / CPUs: 2
  → Firmware: BIOS (NOT UEFI — UEFI is not supported yet)
```

**VMware:**

```bash
qemu-img convert -O vmdk MyOS.qcow2 MyOS.vmdk
```

Then in VMware Workstation/Player: New VM → Use existing disk → point at the vmdk. **Firmware: BIOS.**

### Live ISO (`build.output: iso`)

```bash
distrorun test MyOS.iso -r 2048
```

Or in virt-manager: New VM → Local install media → MyOS.iso.

### USB image (`build.output: img`)

```bash
sudo dd if=MyOS.img of=/dev/sdX bs=4M status=progress conv=fsync
sync
```

Boot any machine from the USB stick. Changes persist on the `DISTRORUN_PERS` partition.

---

## Global Flags

| Flag | Scope | Description |
|---|---|---|
| `-d`, `--debug` | `build` | Show full dnf/apk output. |
| `-h`, `--help` | top-level | Print help. |
