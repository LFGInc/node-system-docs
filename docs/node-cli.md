# Node CLI Documentation

[`lfgnode`](https://github.com/LFGInc/node-cli) is a command-line interface (CLI) tool for managing workloads and clusters in the LFG network. This document provides an overview of the commands, folder structure, and CI/CD pipeline used in this repository.

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

## Underlying Technology

The `lfgnode` CLI is developed using the Go programming language and is designed to simplify workload and cluster management by leveraging Kubernetes technologies. Specifically, it utilizes [k3s](https://k3s.io/), a lightweight Kubernetes distribution optimized for edge computing, and [k3d](https://k3d.io/stable), a tool that enables running Kubernetes clusters within Docker containers.

By integrating these technologies and [Node System Backend](backend.md)'s APIs, `lfgnode` provides an intuitive command-line interface that abstracts the complexities of Kubernetes, allowing users to efficiently manage workloads and clusters on their nodes. This approach ensures a seamless experience for deploying and maintaining applications in a containerized environment.
