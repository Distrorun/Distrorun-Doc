---
icon: lucide/code
---

# Developer Guide

This document explains **how the DistroRun engine works internally** — the build pipeline, the per-distro divergences, and the design decisions behind each. If you want to contribute, extend the engine to a new distribution, or just understand what happens when you type `distrorun build`, start here.

---

## Project Structure

```text
Distrorun/
├── main.go                         # CLI entry point + pipeline orchestration
├── go.mod / go.sum
├── Makefile                        # build / rpm / deb targets
├── nfpm.yaml                       # RPM/DEB packaging config
├── distrorun.1                     # man page (groff)
├── sample.distrorun.yaml           # example config
└── internal/
    ├── config/
    │   ├── config.go               # YAML structs + accessor methods
    │   ├── validate.go             # validation
    │   └── config_test.go
    ├── rootfs/
    │   ├── bootstrap.go            # Alpine bootstrap + shared helpers
    │   ├── fedora.go               # Fedora bootstrap (dnf --installroot, dracut, init patching)
    │   ├── packages.go             # apk/dnf user-package install
    │   ├── users.go                # user creation + chpasswd hashing
    │   ├── services.go             # rc-update / systemctl enable
    │   ├── initramfs.go            # custom live-CD init script + Alpine initramfs patching
    │   └── cleanup.go              # safe unmounting + rootfs cleanup
    ├── bootloader/
    │   ├── syslinux.go             # ISOLINUX (BIOS) — used by Alpine
    │   └── grub.go                 # GRUB2 BIOS — used by Fedora ISO and disk/img modes
    ├── iso/
    │   ├── build.go                # squashfs + xorriso (ISO mode)
    │   └── img.go                  # raw USB image with persistence (img mode)
    ├── disk/
    │   └── build.go                # qcow2 disk image (disk mode)
    ├── registry/
    │   ├── client.go               # push/pull HTTP client
    │   └── credentials.go          # login/logout token storage
    ├── sbom/
    │   └── sbom.go                 # SPDX 2.3 JSON (Trivy or apk fallback)
    └── ui/
        └── ui.go                   # lipgloss styling + spinner
```

Every package lives under `internal/` — the standard Go convention for "implementation, not API". The only exported entry point is `main.go`.

---

## CLI dispatch

`main.go` switches on `os.Args[1]`:

* `build` → `runBuild()` — the focus of this guide.
* `test` → `runTest()` — launches QEMU.
* `login` / `logout` / `push` / `pull` → registry commands.
* `version`, `help` — informational.

Run with `-d` or `--debug` on `build` to stream package-manager output instead of showing a spinner.

---

## The Build Pipeline

When you run `sudo distrorun build config.yaml`, the engine executes a sequence of steps. The exact step count is **8** (or **9** if `build.sbom: true`), and several steps fork on `cfg.Distro.Base` and `cfg.OutputMode()`.

### Step 1 — Parse configuration (`internal/config`)

YAML → Go structs via `gopkg.in/yaml.v3`:

```go
type Config struct {
    Version  string    `yaml:"version"`
    Name     string    `yaml:"name"`
    Distro   Distro    `yaml:"distro"`
    Packages []string  `yaml:"packages"`
    Users    []User    `yaml:"users"`
    Services *Services `yaml:"services"`
    Build    *Build    `yaml:"build"`
}

type Build struct {
    SBOM        bool   `yaml:"sbom"`
    Output      string `yaml:"output"`        // "iso" | "disk" | "img"
    DiskSize    string `yaml:"disk_size"`
    PersistSize string `yaml:"persist_size"`
}
```

Validation collects errors and returns them all at once. Required:

* `version`, `name`, `distro.base` are present.
* `distro.base ∈ {alpine, fedora}`.
* If `distro.base == fedora`, `distro.type` (when set) is `server` or `workstation`.
* At least one user with both `name` and `password`.
* `build.output` (when set) is one of `iso`, `disk`, `img`.

### Step 2 — Check host dependencies

Forks on `cfg.OutputMode()`:

| Output mode | Function | Checks |
|---|---|---|
| `disk` | `disk.CheckDiskDeps` | `qemu-img`, `sfdisk`, `losetup`, `mkfs.ext4`, `grub2-install`, `grub2-mkconfig` |
| `img` | `iso.CheckImgDeps` | `qemu-img`, `sfdisk`, `losetup`, `mkfs.ext2`, `mkfs.ext4`, `mksquashfs`, `grub2-install` |
| `iso` (Fedora) | `iso.CheckFedoraDeps` | `xorriso`, `mksquashfs`, `dnf`, `grub2-mkimage` |
| `iso` (Alpine) | `iso.CheckHostDeps` | `xorriso`, `mksquashfs`, ISOLINUX files (`isohdpfx.bin` is optional but warned) |

### Step 3 — Bootstrap rootfs (`internal/rootfs`)

Forks on `cfg.Distro.Base` and `cfg.OutputMode()`.

**Alpine** (`Bootstrap`):

1. Discover the latest minirootfs filename by parsing `https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/<arch>/latest-releases.yaml`.
2. Download tarball (~3 MB) → `/tmp/distrorun-<name>/minirootfs.tar.gz`.
3. Extract into the rootfs dir.
4. Bind-mount `/proc`, `/dev`, `/sys` into the rootfs.
5. Copy host `/etc/resolv.conf` (so `apk` can resolve mirrors).
6. Configure `/etc/apk/repositories`, run `apk update`, install `alpineBasePackages` (`alpine-base`, `linux-lts`, `linux-firmware-none`, `mkinitfs`, `openrc`, `e2fsprogs`, `bash`, `shadow`).
7. Configure networking (`/etc/network/interfaces` with DHCP on eth0).
8. Configure mkinitfs features for live CD (`features="ata base cdrom scsi squashfs usb virtio loop network"`).
9. Generate initramfs via `mkinitfs`.
10. Patch the initramfs (`PatchInitramfs`): replace `/init` with our live-CD init script, repack as gzip cpio.

**Fedora** (`BootstrapFedora` for ISO, `BootstrapFedoraDisk` for disk):

1. Bind-mount `/proc`, `/dev`, `/sys` first (RPM scriptlets need them during `dnf --installroot`).
2. `dnf install --installroot=<rootfs> --releasever=43 --use-host-config --setopt=install_weak_deps=False --setopt=tsflags=nodocs --nogpgcheck -y <packages>`.
   * Server packages: kernel, systemd, dracut, NetworkManager, openssh, sudo, busybox, dnf, util-linux, etc.
   * Workstation = server packages + GNOME stack (gdm, gnome-shell, firefox, xorg).
   * Without `-d`, dnf runs with `-q` and stdout is dropped — a spinner is shown via `ui.NewSpinner`.
3. Copy host `/etc/resolv.conf`.
4. Write a NetworkManager DHCP connection at `/etc/NetworkManager/system-connections/dhcp.nmconnection`. Enable NetworkManager via `systemctl enable`.
5. Set hostname from the first user's name.
6. Write a custom `/etc/os-release`, `/etc/issue`, `/etc/motd` (DistroRun branding).
7. Write `/etc/selinux/config` with `SELINUX=disabled` (avoids autorelabel hang on first boot).
8. **ISO mode only:** generate a fresh initramfs via `dracut --compress=gzip --no-hostonly --add-drivers "squashfs loop iso9660 overlay sr_mod cdrom ata_piix ahci virtio_blk virtio_pci virtio_scsi"`, then patch it: extract → inject busybox + applet symlinks → replace `/init` with the live-CD script → repack.
9. **Disk mode** skips the initramfs patching step entirely — dracut's stock initramfs (built by the kernel's `%posttrans` scriptlet) is already correct for normal disk boot.

> The Fedora `--releasever` is currently pinned to 43. To support a different release, change it in `internal/rootfs/fedora.go` and `internal/rootfs/packages.go`.

### Step 4 — Install user packages (`internal/rootfs/packages.go`)

* **Fedora:** `dnf install --installroot=<rootfs> ... <packages>` from the host (dnf isn't reliably present inside a freshly installed Fedora rootfs). Quiet by default; spinner during install.
* **Alpine:** `chroot <rootfs> apk add --no-cache <packages>`. With `-d`, output is parsed by a custom `apkWriter` that prints styled `(N/M) name version` lines.

### Step 5 — Set up users (`internal/rootfs/users.go`)

For each user in the config:

* Non-root users:
  * Fedora: `useradd -m -s /bin/bash -G wheel <name>` — the `wheel` group grants sudo via the default `/etc/sudoers` rule.
  * Alpine: `adduser -D -s /bin/bash <name>`.
* Root: switch shell to bash via `sed` on `/etc/passwd`.
* All users: password set via `echo 'name:password' | chpasswd` (SHA-512).

The first user's name is also written to `/etc/hostname`.

### Step 6 — Enable services (`internal/rootfs/services.go`)

* Fedora: `chroot <rootfs> systemctl enable <svc>` for each.
* Alpine: `chroot <rootfs> rc-update add <svc> default`.

### Step 7 (optional) — Generate SBOM (`internal/sbom`)

Runs only if `build.sbom: true`. Output: `<name>-sbom.spdx.json`.

* **Trivy path** (preferred): if `trivy` is on `PATH`, runs `trivy rootfs --format spdx-json -o <out> <rootfs>`.
* **Alpine fallback**: parses `chroot <rootfs> apk info -v`, builds SPDX 2.3 JSON with `pkg:apk/alpine/<name>@<version>` purls and a root `SPDXRef-rootfs` package.

### Step 8 — Set up bootloader / prepare staging (`internal/bootloader`)

Right before this step the engine calls `rfs.Unmount()` then `rfs.CleanupRootfs()` (clear `/var/cache/{apk,dnf}`, `/dev/*`).

Then forks on output mode:

* **`disk`** — `disk.Build` does its own staging directly on a loop device (see Step 9).
* **`iso` (Alpine)** — `bootloader.Setup`: copies ISOLINUX files (`isolinux.bin`, `ldlinux.c32`, `libcom32.c32`), copies `vmlinuz-lts` and the patched `initramfs-lts` into `staging/boot/`, writes `isolinux.cfg`.
* **`iso` (Fedora)** — `bootloader.SetupGrub`: copies `vmlinuz-<kver>` and `initramfs-<kver>.img`, generates `eltorito.img` via `grub2-mkimage -O i386-pc-eltorito`, writes `grub.cfg`.
* **`img`** — staging is built and bootloader installed inside `iso.BuildImg` (see Step 9).

### Step 9 — Build final artifact

Forks on output mode:

#### `iso` (Alpine — `iso.Build`)

```text
mksquashfs rootfs staging/rootfs.squashfs -comp xz -no-xattrs -noappend
xorriso -as mkisofs \
        -o <out>.iso \
        -b isolinux/isolinux.bin -c isolinux/boot.cat \
        -no-emul-boot -boot-load-size 4 -boot-info-table \
        [-isohybrid-mbr <isohdpfx.bin>] \
        staging/
```

If `isohdpfx.bin` is found, the ISO is also USB-bootable (BIOS).

#### `iso` (Fedora — `iso.BuildFedora`)

```text
mksquashfs rootfs staging/rootfs.squashfs -comp xz -no-xattrs -noappend
xorriso -as mkisofs \
        -o <out>.iso \
        -V DISTRORUN \
        -b boot/grub2/i386-pc/eltorito.img \
        -no-emul-boot -boot-load-size 4 -boot-info-table \
        staging/
```

#### `disk` (`disk.Build`)

1. `qemu-img create -f raw disk.img <disk_size>`.
2. `sfdisk` writes a single bootable ext4 partition (1 MB BIOS boot gap).
3. `losetup -fP --show disk.img` attaches a loop device.
4. `mkfs.ext4 -L DISTRORUN <loopdev>p1`.
5. `cp -a <rootfs>/. /mnt/`.
6. Write `/etc/fstab` from the partition UUID.
7. Bind-mount `/proc`, `/sys`, `/dev`, mount tmpfs on `/run`.
8. `chroot grub2-install --target=i386-pc <loopdev>` (BIOS only — no UEFI yet).
9. `chroot grub2-mkconfig -o /boot/grub2/grub.cfg`.
10. `qemu-img convert -f raw -O qcow2 disk.img <out>.qcow2`.

#### `img` (`iso.BuildImg`)

1. Build squashfs from rootfs.
2. Compute boot-partition size = squashfs + kernel/initramfs + 512 MB headroom, rounded up.
3. `qemu-img create` → raw image of `boot + persist + 10 MB`.
4. `sfdisk` writes two partitions: bootable ext2 (`DISTRORUN_BOOT`) and ext4 (`DISTRORUN_PERS`).
5. Loop-mount, format, copy squashfs + kernel + initramfs.
6. Bind-mount pseudo-fs, run `grub2-install --target=i386-pc`, write `grub.cfg`.
7. The result is a raw `.img` — `dd` it to a USB stick.

---

## The custom live-CD init

`internal/rootfs/initramfs.go` defines `customInit` — a single shell script that replaces `/init` inside the initramfs (Alpine and Fedora ISO modes both use it).

What it does:

1. `/bin/busybox --install -s` — populate symlinks for `mount`, `modprobe`, `sleep`, etc.
2. Mount `devtmpfs`, `proc`, `sysfs`.
3. `modprobe` the modules needed to find a boot device: `loop`, `squashfs`, `isofs`, `overlay`, `sr_mod`, `cdrom`, `ata_piix`, `ahci`, `virtio_*`, `e1000`, etc.
4. Scan `/dev/sr0`, `/dev/sd[ab]1`, `/dev/vda1`, `/dev/nvme0n1p1`, `/dev/sda`, `/dev/vda` for `rootfs.squashfs` (15-second timeout, 1 s polling).
5. Mount the squashfs as `/lower` (read-only).
6. Look for a `DISTRORUN_PERS`-labeled partition (`/dev/sda2`, `/dev/vda2`, etc.). If found, mount it and use it as the overlay upper layer (persistent). Otherwise, fall back to a tmpfs.
7. `mount -t overlay overlay -o lowerdir=/lower,upperdir=...,workdir=... /sysroot`.
8. Move `/dev`, `/proc`, `/sys` into `/sysroot`.
9. `exec switch_root /sysroot /sbin/init`.

This is exactly how Ubuntu, Fedora, and most live distros work — squashfs read-only + overlayfs writable upper.

**Why patch the initramfs at all?** Fedora's stock dracut initramfs assumes `root=` on the kernel cmdline points at a real disk partition. For a live CD it doesn't — we have to find the squashfs ourselves and build the overlay before handing off to systemd. Replacing `/init` is the cleanest way.

---

## Cleanup safety

The most dangerous operation in the codebase is rootfs cleanup. The `/dev` bind mount means the rootfs's `/dev` IS the host's `/dev`. Deleting the rootfs while `/dev` is still mounted destroys real device nodes (we have hit this — `/dev/null` had to be recreated with `mknod`).

Order (in `main.go` and the cleanup methods):

```text
1. Unmount()        — reads /proc/mounts, unmounts deepest-first
2. CleanupRootfs()  — clears /var/cache/{apk,dnf}, /dev/*
3. defer Cleanup(true) — removes the working dir; runs even on errors
```

`CleanupRootfs` only clears `/dev/*` *after* `Unmount()` — that ordering is invariant.

---

## UI System (`internal/ui`)

Built on charmbracelet/lipgloss.

Key components:

* `StepHeader(step, total, msg)` — `[3/9]` badge + bold message.
* `SubStep`, `Detail`, `URL`, `Info`, `InfoPath` — labeled lines.
* `PackageItem(idx, total, name, version)` — `(3/28) nginx 1.26.3-r0`.
* `ServiceItem`, `UserItem` — bullets for users/services.
* `Spinner` — rotating Braille spinner (`⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏`) on stderr. Used during dnf/apk runs in non-debug mode. Frame rate 80 ms (12.5 fps); cleans its line on `Stop()`.

Color palette:

| Color | Hex | Usage |
|---|---|---|
| Teal | `#06b6d4` | step badges, arrows, accents |
| Cyan | `#67e8f9` | values, package names |
| Green | `#22c55e` | success checkmarks |
| Red | `#ef4444` | error badges |
| Yellow | `#eab308` | warnings, sizes |
| Purple | `#a78bfa` | paths |
| Dim | `#64748b` | labels, counters |

---

## Packaging

### RPM / DEB (nfpm)

```bash
make rpm    # → distrorun-<version>.x86_64.rpm
make deb    # → distrorun_<version>_amd64.deb
```

`nfpm.yaml` declares dependencies (`xorriso`, `squashfs-tools`) and recommends (`qemu-system-x86`, `syslinux`). Installs to `/usr/bin/distrorun` plus the man page at `/usr/share/man/man1/distrorun.1`.

### Man page

Groff format (`distrorun.1`). View with `man distrorun` after installing the package.

---

## Key Design Decisions

### Why chroot instead of containers?

Containers (Docker/Podman) add a layer of abstraction we don't need. `chroot` is the same primitive used by `debootstrap`, `pacstrap`, Fedora's `mock`, and Alpine's `setup-disk`. It's simpler, faster, and produces identical results.

### Why GRUB2 for Fedora and ISOLINUX for Alpine?

ISOLINUX is dead-simple for BIOS-only CD-ROM booting and matches Alpine's own live ISO toolchain. Fedora's RPM ecosystem ships GRUB2 already, and `grub2-mkimage -O i386-pc-eltorito` produces a single boot image we can drop into the ISO without juggling separate `boot.cat` files. Both produce BIOS-only ISOs today; UEFI is on the roadmap.

### Why overlayfs for live CDs?

The ISO is read-only (ISO9660). We need writes (package installs at boot, log files, user state). Overlayfs layers a writable upper (tmpfs by default, or the `DISTRORUN_PERS` partition if present) on top of the read-only squashfs. Standard live-distro pattern.

### Why dynamic Alpine version discovery?

Alpine's CDN URL includes the exact version (`alpine-minirootfs-3.21.3-x86_64.tar.gz`). Hardcoded URLs 404 when patch versions roll forward. Parsing `latest-releases.yaml` always gets the current filename. The Fedora `releasever` is hardcoded for now because Fedora's archive structure is more stable per-release.

### Why disable SELinux on Fedora?

Fedora's targeted policy expects relabeled filesystems. Our cross-host build (`dnf --installroot` on a non-Fedora host, or even on Fedora with different policy versions) can produce contexts that don't match. Without `SELINUX=disabled`, first boot can hang in autorelabel. Disabling is a tradeoff — users who need SELinux can turn it back on inside the running system.

### Why patch the initramfs ourselves?

Stock dracut initramfs assumes `root=<disk>`. For a live CD we have to find `rootfs.squashfs` on whatever device we ended up booting from, build an overlay, then hand off to systemd. Patching `/init` is cleaner than maintaining a custom dracut module.

---

## Adding a new base distribution

The architecture is module-based. To add (say) Debian:

1. **`config/validate.go`** — allow `"debian"` in `distro.base` validation.
2. **`rootfs/debian.go`** (new) — implement `BootstrapDebian` / `BootstrapDebianDisk` mirroring `fedora.go`. Use `debootstrap` instead of `dnf --installroot`.
3. **`rootfs/packages.go`** — add a `r.distro == "debian"` branch using `apt-get install`.
4. **`rootfs/services.go`** — Debian uses systemd, so the `systemctl enable` branch already covers it.
5. **`rootfs/users.go`** — `useradd -G sudo` (Debian uses `sudo` group, not `wheel`).
6. **`bootloader/`** — reuse `grub.go` (Debian also uses GRUB2).
7. **`sbom/sbom.go`** — add a `dpkg-query -W` branch.
8. **`iso/build.go`** — reuse `BuildFedora` (it's distro-agnostic given the GRUB2 staging layout).

Most of the "new distro" work is one new `<distro>.go` file in `internal/rootfs/`; the rest of the pipeline is generic.

---

## Running tests

```bash
# Unit tests (config validation)
go test ./...

# Vet & build
go vet ./... && go build -o distrorun .

# Full integration: build + boot
sudo ./distrorun build sample.distrorun.yaml
./distrorun test my-alpine-server.iso -r 1024
```

For a Fedora workstation daily-driver VM:

```bash
sudo ./distrorun build fedora.distrorun.yaml      # → MyOS.qcow2 (when build.output: disk)
# Then: virt-manager → import existing disk → MyOS.qcow2 → BIOS firmware
```

---

## Environment requirements

| Requirement | Minimum | Notes |
|---|---|---|
| Go | 1.25+ | `gopkg.in/yaml.v3`, `lipgloss` |
| OS | Linux (x86_64) | `chroot`, `mount`, `losetup` are Linux-only |
| Privileges | root | required for `build` |
| Disk space | 1.5 GB | Alpine ISO; ~6–8 GB for Fedora workstation qcow2 |
| RAM (host) | 1 GB+ | XZ squashfs is memory-hungry on large rootfs |
