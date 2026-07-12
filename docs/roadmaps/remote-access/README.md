# Homelab Remote Access Over VPN

Status: Planned
Purpose: Record the approved phase-1 remote-access design so it can be implemented later without redesigning the rest of the homelab.
Depends on: [Current state](../../overview/current-state.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md)
Related docs: [Roadmaps index](../README.md), [Architecture](../../overview/architecture.md), [Storage and Samba](../../platform/storage-and-samba.md)

## Summary

This document captures the approved rollout plan for adding private remote access to the homelab.

Phase 1 uses Tailscale directly on `piroman-server` as the VPN entry point for the whole home network. The goal is to keep the current `*.pirocorp.com` naming pattern, avoid exposing more services publicly, and preserve an easy migration path to self-hosted WireGuard or a dedicated router/firewall later.

This is planning material only. Tailscale is not yet deployed on the live host.

Execution runbook: [Tailscale remote access runbook](../../operations/tailscale-remote-access-runbook.md)

## Goals

- Reach the full home LAN remotely, not only the Ubuntu server.
- Keep using the same `*.pirocorp.com` service URLs after connecting remotely.
- Reuse the current AdGuard Home and Nginx Proxy Manager design instead of replacing it.
- Avoid buying new edge hardware before remote access is proven useful.
- Keep the first implementation easy to add, test, and remove.

## Approved Phase-1 Design

### VPN entry point

- Install Tailscale on the Ubuntu host itself, not in Docker.
- Use `piroman-server` as the subnet router and remote-access gateway.
- Keep the ISP modem/router in place for phase 1.
- Do not add new direct public exposure for Netdata, Portainer, SMB, SSH, or other management services.

### Network reachability

- Advertise the actual homelab LAN subnet through Tailscale during implementation.
- Confirm the current LAN CIDR before enabling subnet routing instead of assuming it from the `192.168.0.x` addressing pattern alone.
- Keep the server LAN IP stable so VPN routing and DNS assumptions remain valid.

### DNS and service access model

- Preserve the current `pirocorp.com` access pattern for remote use.
- Treat `*.pirocorp.com` as split-DNS service names backed by AdGuard Home local rewrites to private LAN addresses.
- Keep Nginx Proxy Manager as the HTTPS ingress layer for the existing services after name resolution succeeds.
- Configure the VPN so remote clients use AdGuard Home as the DNS source for `pirocorp.com` queries.
- Do not treat the current service subdomains as intended public internet endpoints.

### Security defaults

- Limit phase 1 to trusted personal devices first.
- Require the VPN before using private homelab services remotely.
- Keep SSH, SMB, Netdata, Portainer, Nextcloud, Plex, Immich, and the rest on LAN/VPN access paths instead of publishing them more broadly.
- Prefer per-device enrollment and per-device revocation over shared credentials.

## Deferred Design Choices

These are intentionally not part of phase 1:

- Replacing Tailscale with self-hosted WireGuard
- Moving VPN termination to a dedicated router/firewall
- Redesigning Nginx Proxy Manager
- Making `*.pirocorp.com` publicly reachable from the internet
- Cleaning up existing qBittorrent router forwarding rules

## Implementation Checklist For Later

1. Confirm the real LAN subnet and keep `piroman-server` on a stable address.
2. Install Tailscale on the Ubuntu host.
3. Enable the host settings required for subnet routing.
4. Advertise the homelab LAN route through Tailscale.
5. Configure Tailscale DNS so `pirocorp.com` queries go to AdGuard Home.
6. Approve the advertised route and DNS behavior in the Tailscale admin controls if required.
7. Enroll one trusted laptop and one trusted phone as the first remote clients.
8. Validate name resolution, SSH, SMB, and service access from an external network.
9. Write the final operational runbook under platform/operations after the deployment is live.

## Validation Targets

- From mobile data or another external network, a trusted client can connect successfully.
- After VPN connection, `server.pirocorp.com`, `nextcloud.pirocorp.com`, `immich.pirocorp.com`, and the other existing service names resolve and open as expected.
- SSH access to `piroman-server` works through the VPN path.
- SMB shares and other LAN resources are reachable remotely.
- Local on-site access continues working without changes.
- Removing the VPN configuration later returns the homelab to its pre-VPN behavior without changing application deployments.

## Future Migration Paths

### Self-hosted fallback

If third-party control-plane dependence becomes undesirable, replace the Tailscale transport with self-hosted WireGuard while keeping the same remote-access goals:

- whole-LAN remote reachability
- AdGuard-backed DNS for `pirocorp.com`
- private access to the existing service set

### Long-term edge upgrade

If the homelab later needs VLANs, stronger east-west firewalling, site-to-site VPN, or cleaner WAN-edge separation, move VPN termination to a dedicated router/firewall and reduce `piroman-server` back to an application host.

## Assumptions And Defaults

- `*.pirocorp.com` currently resolves through AdGuard Home to private LAN addresses rather than serving as general public internet entry points.
- Whole-LAN access is the objective, not a single-app remote login.
- Phase 1 should be easy to add and easy to remove.
- Tailscale on the host is the chosen first implementation path.
- WireGuard and a dedicated router remain later options, not phase-1 blockers.
