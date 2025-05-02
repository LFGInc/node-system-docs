# Node CLI

[`lfgnode`](https://github.com/LFGInc/node-cli) is a command-line interface (CLI) tool for managing workloads and clusters in the LFG network. This document provides an overview of the commands, folder structure, and CI/CD pipeline used in this repository.

## Prerequisites

- For MacOS & Windows, [Docker Desktop](https://docs.docker.com/desktop/) is required.
  - After installing Docker Desktop. Open Docker Desktop -> Setting -> General -> Select `Use WSL 2 Based Engine` -> Select `dd the *.docker.internal names to the host's /etc/hosts file` -> Apply & Restart.
- For Linux, [Docker Engine](https://docs.docker.com/engine/install/) is required.
- [Go](https://go.dev/doc/install) 1.23 or later is required to build the CLI from source.

## How to run

1. Download packages
   ```bash
   go get
   go mod tidy
   ```
2. Build the CLI
   ```bash
   go build -o lfgnode main.go
   ```  

## Commands

The `lfgnode` CLI supports the following commands:

### Core Commands
- **`start`**: Start the LFG cluster.
- **`stop`**: Stop the LFG cluster.
- **`status`**: Check the status of the LFG cluster.
- **`update`**: Update the CLI or cluster components.
- **`info`**: Display information about the node.
- **`version`**: Print the version of the CLI.
- **`config`**: Manage configuration settings.

### Workload Management
- **`workload add`**: Add a new workload to the cluster.
- **`workload list`**: List all workloads in the cluster.
- **`workload remove`**: Remove a workload from the cluster.
- **`workload restart`**: Restart a workload.
- **`workload status`**: Check the status of a workload.

---

## Folder Structure

The repository is organized as follows:

```
.
├── cmd/                # CLI commands
│   ├── cmd.go          # Root command definition
│   ├── start.go        # Start command
│   ├── stop.go         # Stop command
│   ├── update.go       # Update command
│   ├── workload/       # Workload-related commands
│   │   ├── add.go
│   │   ├── list.go
│   │   ├── remove.go
│   │   └── status.go
│   └── config/         # Configuration-related commands
│       ├── config.go
│       └── privateKey.go
├── internal/           # Internal logic and utilities
│   ├── api/            # API integrations
│   ├── cluster/        # Cluster management (e.g., k3d)
│   ├── hooks/          # Pre-run hooks for commands
│   └── workload/       # Workload management logic
├── pkg/                # Shared packages
│   ├── config/         # Configuration utilities
│   ├── logger/         # Logging utilities
│   ├── constants/      # Constants used across the project
│   └── util/           # Helper functions
├── scripts/            # Shell scripts for installation and setup
│   ├── install.sh      # Linux installation script
│   └── install_macos.sh # MacOS installation script
├── docs/               # Documentation
│   └── design.md       # Design document
├── .github/            # GitHub workflows
│   └── workflows/
│       └── release.yml # CI/CD pipeline configuration
├── main.go             # Entry point for the CLI
├── go.mod              # Go module definition
├── go.sum              # Go module dependencies
└── README.md           # Project overview and installation guide
```

## Underlying Technology

The `lfgnode` CLI is developed using the Go programming language and is designed to simplify workload and cluster management by leveraging Kubernetes technologies. Specifically, it utilizes [k3s](https://k3s.io/), a lightweight Kubernetes distribution optimized for edge computing, and [k3d](https://k3d.io/stable), a tool that enables running Kubernetes clusters within Docker containers.

By integrating these technologies and [Node System Backend](backend.md)'s APIs, `lfgnode` provides an intuitive command-line interface that abstracts the complexities of Kubernetes, allowing users to efficiently manage workloads and clusters on their nodes. This approach ensures a seamless experience for deploying and maintaining applications in a containerized environment.
