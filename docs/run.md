---
icon: lucide/square-terminal
---

# CLI Reference

The DistroRun Command Line Interface (CLI) is your primary tool for interacting with the core Go orchestration engine. It allows you to validate configurations, trigger system builds, and interact with the local registry.

## Global Flags
These flags can be used alongside any DistroRun command.

* `--debug`, `-d`: Enable verbose logging (useful for troubleshooting package manager output or system calls).
* `--help`, `-h`: Print the help menu for the current command.

---

## distrorun build

The core command of the engine. It reads your YAML file. sets up the isolated namespaces, fetches the packages, applies configurations, and generates the final bootable ISO and SBOM.

> **Privilege Escalation:** Because DistroRun utilizes low-level Linux system primitives (like mounting `/proc` and creating namespaces), the `build` command must be run with `root` privileges.

### Usage

```css
sudo distrorun build [flags]
```

### Flags
* `--no-cache`: Force a fresh download of all base images and packages, ignoring the local squashfs cache.

---

## distrorun registry

DistroRun includes built-in commands to interact with your team's Zensical-powered artifact registry. This allows you to treat OS configurations like Docker images.

### distrorun registry push

Uploads your `distro.yaml` and its generated SBOM to the registry.

**Usage:**

```css
distrorun registry push <username>/<project>:<tag> 
```

### distrorun registry pull

Retrieves a configuration from the registry and saves it locally, allowing you to seamlessly reproduce a colleague's build.

**Usage:**

```css
distrorun registry pull <username>/<project>:<tag>

```

**Example:**

```css
distrorun registry pull companyxyz/nginx-server:v1.0.4
```

