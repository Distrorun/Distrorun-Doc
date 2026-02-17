---
icon: lucide/download
---

# Installation

Welcome to **DistroRun**. Because DistroRun interacts directly with low-level system primitives (like `chroot` and namespaces) to build immutable images, **this engine is designed exclusively for Linux environments.**

## Prerequisites

Before building your first OS image, ensure your system has the following dependencies installed:

### Core Engine
* **[Go (v1.25.0+)][go]**  Required to run and compile the DistroRun core orchestration engine.

### Tooling & Registry
* [Node.js (v22.21.1+)][node]  Required to run the local artifact Registry and the Language Server Protocol (LSP).
* [Visual Studio Code][vscode]  The officially supported IDE for writing your declarative `distro.yaml` files.

> **System Note:** DistroRun relies on native Linux tools to generate bootable artifacts (like `grub-mkrescue`). We recommend running it on a modern Linux host.

---

## Quick Start

Get DistroRun up and running in your terminal in just a few commands:

```bash
# 1. Clone the repository
$ git clone https://github.com/Distrorun/Distrorun.git

# 2. Navigate to the project directory
$ cd Distrorun

# 3. Run the automated installer (compiles the Go binary and sets up the CLI)
$chmod +x install$ ./install
```
[go]: https://go.dev/dl/
[vscode]: https://flathub.org/en/apps/com.visualstudio.code/
[lsp]: https://marketplace.visualstudio.com/vscode/
[node]: https://nodejs.org/en/download/current
