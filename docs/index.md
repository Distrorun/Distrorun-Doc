---
icon: lucide/download
---

# Installation

Welcome to **DistroRun** — a custom Linux OS builder that produces bootable artifacts (live ISOs, qcow2 disk images, or USB images with persistence) from a single YAML file.

Because DistroRun interacts directly with low-level system primitives (`chroot`, `mount`, `losetup`, package-manager `--installroot`), **the engine runs only on Linux hosts**.

---

## Prerequisites

### Core engine

* **[Go 1.25+][go]** — to compile the `distrorun` binary.
* **Root access** — `sudo` is required for the `build` command (chroot + bind-mounts).

### Host tools

The engine shells out to standard Linux tools. Install whichever sets your target needs.

**Always required:**

* `xorriso`, `mksquashfs` (squashfs-tools)

**For Fedora targets** (`distro.base: fedora`):

* `dnf`
* `grub2-tools` (provides `grub2-mkimage`, `grub2-install`, `grub2-mkconfig`)

**For Alpine targets** (`distro.base: alpine`):

* `syslinux` (ISOLINUX bootloader files)

**For `build.output: disk` (qcow2):**

* `qemu-img`, `sfdisk`, `losetup`, `mkfs.ext4`, `grub2-tools`

**For `build.output: img` (USB image with persistence):**

* `qemu-img`, `sfdisk`, `losetup`, `mkfs.ext2`, `mkfs.ext4`, `grub2-tools`

**For `distrorun test`:**

* `qemu-system-x86_64` (and KVM kernel modules for hardware acceleration).

### Optional tooling

* **[Visual Studio Code][vscode]** with the DistroRun LSP extension — provides YAML schema validation and autocomplete for `*.distrorun.yaml` files.
* **[Node.js 22+][node]** — required if you want to run the optional artifact registry server locally.
* **Trivy** — recommended for SBOM generation (`build.sbom: true`); used automatically when on `PATH`.

---

## Quick Start

```bash
# 1. Clone the repo
git clone https://github.com/Distrorun/Distrorun.git
cd Distrorun/Distrorun

# 2. Build the binary
make build
# → ./distrorun

# 3. (Optional) Install system-wide as RPM or DEB
make rpm        # → distrorun-<version>.x86_64.rpm
make deb        # → distrorun_<version>_amd64.deb
sudo dnf install ./distrorun-*.rpm   # or: sudo apt install ./distrorun_*.deb

# 4. Build your first OS
sudo ./distrorun build sample.distrorun.yaml
```

---

## What you get

After a successful build, the output filename is determined by `build.output` in your YAML:

| `build.output` | Output | Boot it with |
|---|---|---|
| `iso` (default) | `<name>.iso` | `distrorun test <name>.iso` or burn / write to USB |
| `disk` | `<name>.qcow2` | virt-manager (import existing disk) / `qemu-system-x86_64 -hda` |
| `img` | `<name>.img` | `dd` to a USB stick, then boot from it |

If `build.sbom: true`, you also get `<name>-sbom.spdx.json` (SPDX 2.3 JSON).

---

## Next steps

* **[How to write a Distrorun file](yamlfile.md)** — the YAML schema reference.
* **[CLI Reference](run.md)** — every command, every flag, with examples.
* **[Developer Guide](dev.md)** — internals, build pipeline, how to extend.

[go]: https://go.dev/dl/
[vscode]: https://flathub.org/en/apps/com.visualstudio.code/
[node]: https://nodejs.org/en/download/current
