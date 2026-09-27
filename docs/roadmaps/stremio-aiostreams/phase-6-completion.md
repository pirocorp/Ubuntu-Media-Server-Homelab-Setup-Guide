# Phase 6 Completion — Router And Peer Connectivity Validation

Status: Complete, with manual router forwarding deferred  
Date: 2026-09-27  
Scope: Validate the live `6882/TCP+UDP` BitTorrent peer path for `stremio-libtorrent-server`, preserve qBittorrent on `6881/TCP+UDP`, and confirm that no management/media-control ports are exposed as general public services.

## Outcome

Phase 6 validation is complete.

The live host/container configuration for `6882/TCP+UDP` was verified, LAN reachability was proven, and real bidirectional UDP BitTorrent peer traffic with Internet peers was observed. qBittorrent remains unchanged on `6881/TCP+UDP`.

A manual router port-forward for `6882/TCP+UDP` was intentionally deferred and will be added later. As a result, inbound TCP reachability from the public Internet is not currently available and remains a known follow-up item rather than a Phase 6 blocker.

No Stremio web UI, API, trusted-media, or management port was opened as a general public Internet service.

## Host Listener Validation

The Ubuntu host was checked with:

```bash
sudo ss -lntup | grep -E '(:6882[[:space:]]|:6882$)'
```

Observed listeners:

```text
6882/UDP -> 0.0.0.0 and [::] via docker-proxy
6882/TCP -> 0.0.0.0 and [::] via docker-proxy
```

This confirms that the Docker-published BitTorrent peer port is active for both protocols and both address families on the host.

## Host Firewall Validation

UFW is active with:

```text
Default: deny (incoming), allow (outgoing), deny (routed)
```

There is no explicit UFW allow rule for `6882`.

The Docker `DOCKER-USER` chain was also checked:

```bash
sudo iptables -S DOCKER-USER
```

Observed state:

```text
-N DOCKER-USER
```

No custom `DOCKER-USER` filtering rule is currently blocking the published peer port.

A separate LAN TCP test from another host successfully reached:

```text
192.168.0.10:6882/TCP
```

This confirms that the live Docker/firewall path accepts TCP peer-port traffic from the LAN without adding a new UFW rule.

## Router And LAN Topology

Validated host routing baseline:

```text
Ubuntu host:     192.168.0.10
LAN:             192.168.0.0/24
Default gateway: 192.168.0.1
```

The Internet connection presents a public IPv4 address and no working public IPv6 path was detected during validation. The exact public IPv4 address is intentionally not recorded in repository documentation.

## Internet Peer Connectivity

### UDP

Packet capture on the host showed active bidirectional UDP traffic between:

```text
192.168.0.10:6882 <-> multiple Internet peer addresses
```

The capture included both outbound packets sourced from local port `6882` and inbound responses delivered back to local port `6882`.

This provides real-world proof that UDP BitTorrent peer connectivity is functioning through the current NAT path.

No log evidence was found for explicit UPnP/NAT-PMP port mapping by `stremio-libtorrent-server`, so the observed UDP reachability is documented only as current runtime behaviour, not as proof of a permanent router mapping.

### TCP

An external test was run from a separate mobile-network connection against the home public IPv4 address:

```powershell
Test-NetConnection <home-public-ip> -Port 6882
```

Result:

```text
PingSucceeded    : True
TcpTestSucceeded : False
```

Therefore inbound `6882/TCP` is not currently reachable from the public Internet.

This is consistent with the decision to defer the manual router forward.

## Docker Port-Binding Validation

The running container was inspected directly:

```bash
docker inspect stremio-libtorrent-server --format '{{json .HostConfig.PortBindings}}'
```

Validated bindings:

```text
6882/TCP -> all host interfaces
6882/UDP -> all host interfaces
8080/TCP -> 192.168.0.10:8081 only
11470/TCP -> 192.168.0.10:11470 only
12470/TCP -> 192.168.0.10:12470 only
```

Docker may additionally display image metadata for `6881/tcp`, but that is not a host-published binding. The qBittorrent host assignment on `6881/TCP+UDP` remains unchanged.

## Security Boundary

The only Stremio port intended to become publicly forwarded later is:

```text
6882/TCP+UDP
```

The following remain LAN-bound and must not be exposed as general public services:

```text
8081/TCP  Stremio Web UI
11470/TCP streaming-server API
12470/TCP trusted media HTTPS
```

The trusted `*.stremio.rocks:12470` media path remains the validated direct client path inside the trusted network model; it is not converted into a general public port-forward.

## Deferred Router Forward

The future router rule is intentionally documented but not yet applied:

```text
External port: 6882
Internal host:  192.168.0.10
Internal port: 6882
Protocol:      TCP + UDP
```

When this is added later:

- keep the existing qBittorrent `6881/TCP+UDP` forward unchanged;
- forward only `6882/TCP+UDP` to `192.168.0.10`;
- do not add forwards for `8081`, `11470`, `12470`, or other management/control ports;
- repeat the external TCP reachability test after the router change.

## Phase 6 Acceptance

| Check | Result |
| --- | --- |
| `6882/TCP` listener active on host | PASS |
| `6882/UDP` listener active on host | PASS |
| qBittorrent `6881/TCP+UDP` preserved | PASS |
| UFW / Docker filtering inspected | PASS |
| LAN TCP reachability to `192.168.0.10:6882` | PASS |
| Real Internet UDP peer traffic on `6882` | PASS |
| Public inbound TCP on `6882` | NOT CURRENTLY REACHABLE — manual router forward deferred |
| Web/API/media-management ports kept out of public-forward scope | PASS |
| Manual router `6882/TCP+UDP` forward | DEFERRED — to be added later |

## Remaining Work

Phase 6 is closed with the router forward recorded as a deferred operational follow-up.

Remaining roadmap work:

- later add the explicit router `6882/TCP+UDP -> 192.168.0.10:6882` forward and re-test external TCP reachability;
- Phase 7 — formal large-file resilience testing, including read-ahead behaviour, controlled throughput-drop tolerance, and final resilience acceptance.
