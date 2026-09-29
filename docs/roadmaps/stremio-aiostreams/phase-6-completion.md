# Phase 6 Completion — Router And Peer Connectivity Validation

Status: Complete  
Date: 2026-09-27  
Router-forward follow-up completed: 2026-09-29  
Scope: Validate the live `6882/TCP+UDP` BitTorrent peer path for `stremio-libtorrent-server`, preserve qBittorrent on `6881/TCP+UDP`, and confirm that no management/media-control ports are exposed as general public services.

## Outcome

Phase 6 validation and its router-forward follow-up are complete.

The live host/container configuration for `6882/TCP+UDP` was verified, LAN reachability was proven, real bidirectional UDP BitTorrent peer traffic with Internet peers was observed, and the explicit router forward for `6882/TCP+UDP` was added to `192.168.0.10`. qBittorrent remains unchanged on `6881/TCP+UDP`.

Both peer protocols are operationally validated:

```text
6882/TCP: PASS
6882/UDP: PASS
```

Public inbound TCP reachability was independently confirmed from the Internet after the router change. UDP is also accepted as operational based on the active UDP listener, Docker DNAT/forwarding rules, the router's TCP+UDP mapping, and observed live bidirectional Internet UDP peer traffic on local port `6882`.

No Stremio web UI, API, trusted-media, or management port was opened as a general public Internet service.

## Host Listener Validation

The Ubuntu host was checked with:

```bash
sudo ss -lntup | grep 6882
```

Observed listeners:

```text
6882/UDP -> 0.0.0.0 and [::] via docker-proxy
6882/TCP -> 0.0.0.0 and [::] via docker-proxy
```

This confirms that the Docker-published BitTorrent peer port is active for both protocols and both address families on the host.

## Host Firewall And Docker Validation

UFW remains active. There is no dedicated UFW allow rule for `6882`; Docker's published-port NAT/forwarding path handles this traffic.

The active Docker rules were verified directly:

```bash
sudo iptables -t nat -S DOCKER | grep 6882
sudo iptables -S DOCKER | grep 6882
```

Validated rules include:

```text
TCP 6882 -> 172.29.0.2:6882 DNAT
UDP 6882 -> 172.29.0.2:6882 DNAT
TCP 6882 -> 172.29.0.2:6882 ACCEPT
UDP 6882 -> 172.29.0.2:6882 ACCEPT
```

A separate LAN TCP test also successfully reached:

```text
192.168.0.10:6882/TCP
```

This confirms that the live Docker/firewall path accepts peer-port traffic without adding a new UFW rule.

## Router And LAN Topology

Validated host routing baseline:

```text
Ubuntu host:     192.168.0.10
LAN:             192.168.0.0/24
Default gateway: 192.168.0.1
```

The Internet connection presents a public IPv4 address. The exact public IPv4 address is intentionally not recorded in repository documentation.

The router now contains the explicit peer-port mapping:

```text
External port: 6882
Internal host:  192.168.0.10
Internal port: 6882
Protocol:       TCP + UDP
Status:         enabled
```

The pre-existing qBittorrent mapping on `6881/TCP+UDP` remains unchanged.

## Internet Peer Connectivity

### UDP

Packet capture on the host previously showed active bidirectional UDP traffic between:

```text
192.168.0.10:6882 <-> multiple Internet peer addresses
```

The capture included both outbound packets sourced from local port `6882` and inbound responses delivered back to local port `6882`.

After the explicit router rule was added, the end-to-end UDP path is recorded as operational: the application listens on `6882/UDP`, Docker DNAT and forwarding accept `6882/UDP`, the router maps `6882/UDP` to the host, and live Internet UDP peer traffic has been observed on that port.

An external UDP scanner reported `open or filtered`, which is expected to be non-definitive for a connectionless protocol and is not treated as a failure.

### TCP

After the explicit router mapping was added, external Internet port checks against the home public IPv4 address reported:

```text
6882/TCP -> Open
```

This confirms that public inbound TCP now reaches the peer listener through:

```text
Internet -> router 6882 -> 192.168.0.10:6882 -> Docker -> stremio-libtorrent-server
```

The earlier pre-forward `TcpTestSucceeded : False` result is therefore superseded by the completed router-forward validation.

## Docker Port-Binding Validation

The running container was inspected directly and validated as publishing:

```text
6882/TCP -> all host interfaces
6882/UDP -> all host interfaces
8080/TCP -> 192.168.0.10:8081 only
11470/TCP -> 192.168.0.10:11470 only
12470/TCP -> 192.168.0.10:12470 only
```

Docker may additionally display image metadata for `6881/tcp`, but that is not a host-published binding. The qBittorrent host assignment on `6881/TCP+UDP` remains unchanged.

## Security Boundary

The only Stremio port intentionally forwarded for public peer traffic is:

```text
6882/TCP+UDP
```

The following remain outside the public-forward scope:

```text
8081/TCP  Stremio Web UI
11470/TCP streaming-server API
12470/TCP trusted media HTTPS
```

The trusted `*.stremio.rocks:12470` media path remains the validated client path and is not converted into a general public router forward.

## Phase 6 Acceptance

| Check | Result |
| --- | --- |
| `6882/TCP` listener active on host | PASS |
| `6882/UDP` listener active on host | PASS |
| qBittorrent `6881/TCP+UDP` preserved | PASS |
| Docker DNAT for `6882/TCP` | PASS |
| Docker DNAT for `6882/UDP` | PASS |
| Docker forwarding/ACCEPT for `6882/TCP+UDP` | PASS |
| LAN TCP reachability to `192.168.0.10:6882` | PASS |
| Real Internet UDP peer traffic on `6882` | PASS |
| Manual router `6882/TCP+UDP -> 192.168.0.10:6882` forward | PASS |
| Public inbound TCP on `6882` | PASS — externally reported open |
| `6882/UDP` operational peer path | PASS |
| Web/API/media-management ports kept out of public-forward scope | PASS |

## Remaining Work

No router-forward follow-up remains for Phase 6. Both `6882/TCP` and `6882/UDP` are recorded as operationally OK.

Phase 7 has also been completed separately. Future Stremio work continues under the multi-source P2P expansion roadmap rather than this baseline connectivity phase.
