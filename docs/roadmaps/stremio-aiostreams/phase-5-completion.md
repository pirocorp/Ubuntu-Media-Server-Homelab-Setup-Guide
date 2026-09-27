# Phase 5 Completion — Stremio Client Acceptance

Status: Complete  
Date: 2026-09-27  
Scope: Primary-TV and desktop-client validation against the self-hosted AIOStreams + `stremio-libtorrent-server` path.

## Outcome

Phase 5 is complete.

The primary TV and desktop client were both validated against the central Stremio streaming stack. The TV successfully uses AIOStreams for stream discovery and `stremio-libtorrent-server` for media delivery. Client playback, forward/backward seeking, and stop/reopen/resume behaviour were exercised successfully.

The validated media path is:

```text
Stremio client
  -> AIOStreams
  -> Torrentio P2P / non-debrid result
  -> stremio-libtorrent-server
  -> BitTorrent peers
  -> /mnt/data/stremio-libtorrent-server cache
  -> trusted HTTPS :12470
  -> Stremio player
```

## Primary TV Acceptance

The primary TV successfully:

- loaded normal AIOStreams results;
- selected P2P torrent streams;
- played through the central server path;
- received the media stream from the Ubuntu server over trusted HTTPS on port `12470`;
- performed forward seek;
- performed backward seek;
- resumed playback after stop/reopen;
- sustained long-form playback under real 4K REMUX load.

During troubleshooting, repeated backward/forward seeks around the same high-bitrate section were used to test whether brief stalls reproduced at a fixed timestamp. Those operations themselves worked and therefore also served as practical seek acceptance tests.

## Server-Side Torrent Engine Proof

The client is not relied upon as the torrent engine for the validated path.

Evidence collected across Phases 4-5 includes:

- selected releases produced live media, `.fastresume`, `.parts`, and resume-state activity under `/mnt/data/stremio-libtorrent-server`;
- the cached movie files grew to completion on the Ubuntu server;
- the TV media connection was observed as a server-to-TV TCP flow on port `12470`;
- Stremio statistics showed active peer/torrent state while playback was served through the central server path;
- completed media remained available from the server cache after the torrent reached `100%`.

This satisfies the client-acceptance requirement that selected torrents are handled by `stremio-libtorrent-server` rather than depending on the TV to join the swarm directly.

## Desktop Client Acceptance

Desktop Stremio was also used successfully during end-to-end validation.

Observed desktop behaviour included:

- normal playback through the configured stack;
- torrent statistics showing the selected release completed at `100%`;
- seeking from near the beginning of a title to approximately the one-hour mark while playback continued successfully.

Desktop validation is therefore sufficient for Phase 5 functional acceptance. Any future Stremio-desktop release-specific behaviour around persistence of a remote streaming-server setting remains a client-maintenance concern rather than a blocker for the validated architecture.

## Known Client-Network Limitation

A known primary-TV limitation was identified during sustained UHD BluRay REMUX playback.

The TV's built-in Ethernet interface is `100 Mbit/s`. A tested 2160p UHD BluRay REMUX showed video-only bitrate peaks above the practical Fast Ethernet ceiling, while live server-to-TV throughput reached approximately the maximum usable bandwidth of that link.

The observed brief sub-second stalls were therefore recorded as a client-network bandwidth limitation for very high-bitrate REMUX files, not as a failure of the central torrent/cache architecture.

The server-side diagnostics around the event did not show a correlated CPU/RAM, disk, server-NIC, or LAN-latency fault.

The TV's Wi-Fi interface is faster but considered less stable. A Wi-Fi A/B test was intentionally not required to close Phase 5.

## Phase 5 Acceptance

| Check | Result |
| --- | --- |
| AIOStreams addon available to primary TV | PASS |
| TV receives normal stream results | PASS |
| TV uses central `stremio-libtorrent-server` media path | PASS |
| Server-side torrent/cache activity validated | PASS |
| Client does not depend on local TV torrent downloading | PASS |
| Forward seek on TV | PASS |
| Backward seek on TV | PASS |
| Stop/reopen/resume on TV | PASS |
| Sustained primary-TV playback | PASS, with known 100 Mbit/s Ethernet ceiling |
| Desktop playback | PASS |
| Desktop seek | PASS |

## Remaining Work

Phase 5 is closed. Remaining roadmap work is:

- Phase 6 — router forwarding and inbound-peer validation for `6882/TCP+UDP`, without disturbing qBittorrent on `6881/TCP+UDP`;
- Phase 7 — formal large-file resilience testing, including read-ahead behaviour, controlled throughput-drop tolerance, and final resilience acceptance.
