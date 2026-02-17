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

## `distrorun build`

The core command of the engine. It reads your YAML file. sets up the isolated namespaces (chroot), fetches the packages, applies configurations, and generates the final bootable ISO and SBOM.

> **Privilege Escalation:** Because DistroRun utilizes low-level Linux system primitives (like mounting `/proc` and creating namespaces), the `build` command must be run with `root` privileges.

### Usage
```bash
sudo distrorun build [flags]
```

### Flags
* `-f, --file string`: Path to the configuration file (default: `./distro.yaml`)
* `-o, --out-dir string`: Output directory for the generated ISO and SBOM (default: `./build`)
* `--no-cache`: Force a fresh download of all base images and packages, ignoring the local squashfs cache.

### Example

```bash
# Build the OS with debug logs enabled
sudo distrorun build -f ./production/web-node.yaml --out-dir ./artifacts -d

# Expected Output:
# [INFO] Parsing web-node.yaml... Valid.
# [INFO] Target OS: fedora (server)
# [INFO] Creating isolated build namespace...
# [INFO] Fetching packages via DNF...
# [INFO] Generating SBOM (SPDX format)...
# [SUCCESS] ISO generated at ./artifacts/web-node.iso
```

---

## `distrorun validate`

Parses and validates your declarative YAML file without executing a build. This command checks the file against the DistroRun JSON Schema to catch typos, missing required fields (like passwords), or invalid package configurations.

*Note: Unlike `build`, this command runs purely statically and does **not** require root privileges.*

### Usage

```bash
distrorun validate -f <path-to-yaml>

```

### Example

```bash
distrorun validate -f ./distro.yaml

# Expected Output:
#  Validation passed. The configuration is strictly compliant.

```

> ** CI/CD Tip:** Add `distrorun validate` to your GitHub Actions or GitLab CI pipelines to automatically test pull requests before attempting to build them.

---

## `distrorun registry`

DistroRun includes built-in commands to interact with your team's Zensical-powered artifact registry. This allows you to treat OS configurations like Docker images.

### `distrorun registry push`

Uploads your `distro.yaml` and its generated SBOM to the registry.

**Usage:**

```bash
distrorun registry push <username>/<project>:<tag> -f <path-to-yaml> --sbom <path-to-sbom>

```

**Example:**

```bash
distrorun registry push marsa-maroc/core-web:v1.0.4 -f ./distro.yaml --sbom ./build/sbom.spdx.json

```

### `distrorun registry pull`

Retrieves a configuration from the registry and saves it locally, allowing you to seamlessly reproduce a colleague's build.

**Usage:**

```bash
distrorun registry pull <username>/<project>:<tag>

```

**Example:**

```bash
distrorun registry pull marsa-maroc/core-web:v1.0.4
# Downloads core-web.yaml to your current directory
```

