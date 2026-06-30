# Base Server Setup

Status: Implemented
Purpose: Describe the current Ubuntu host baseline and point to the preserved installation history.
Related docs: [Infrastructure HowTo](./infrastructure-howto.md), [Docker and Portainer](./docker-and-portainer.md), [Networking and reverse proxy](./networking-and-reverse-proxy.md), [Legacy build history](../archive/legacy-root-readme.md)

Use the [Infrastructure HowTo](./infrastructure-howto.md) for the ordered rebuild sequence. This page is the concise baseline summary of the resulting host.

## Current Baseline

| Item | Value |
| --- | --- |
| Hostname | `piroman-server` |
| LAN IP | `192.168.0.10` |
| OS | `Ubuntu 26.04 LTS` |
| Kernel | `Linux 7.0.0-22-generic` |
| Architecture | `x86-64` |
| Access | SSH |
| OS role | Headless homelab and container host |
| Main admin user | `piroman` |

## Core Platform Capabilities

- Ubuntu Server is the base operating system for the homelab.
- SSH is the primary remote-management path.
- Netdata provides browser-based host monitoring at `server.pirocorp.com`.
- UFW is part of the platform hardening model.
- Docker workloads are hosted locally on the same server.

## Historical Build Coverage

The original step-by-step installation walkthrough was preserved here:

- [Legacy root README snapshot](../archive/legacy-root-readme.md)

Use the archive when you need the full installation narrative, screenshots, or the original build sequence.
