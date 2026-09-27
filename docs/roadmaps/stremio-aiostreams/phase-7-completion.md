# Phase 7 Completion — Large-File Read-Ahead And Resilience Acceptance

Status: Complete  
Date: 2026-09-27  
Scope: Final operational acceptance of the self-hosted Stremio/AIOStreams path under sustained 4K playback, including large-file behaviour, seeking, resume, read-ahead/cache operation, and observed client bottlenecks.

## Outcome

Phase 7 is complete.

The final acceptance decision is based on real-world sustained 4K playback rather than an artificial bandwidth-failure injection test. Two full 4K films were watched through the deployed central path without server-side playback failures. The only recurring issue observed during very heavy UHD REMUX playback is the already-characterized `100 Mbit/s` Ethernet ceiling of the primary TV.

The accepted media path remains:

```text
Stremio client
  -> AIOStreams
  -> Torrentio P2P / non-debrid result
  -> stremio-libtorrent-server
  -> BitTorrent peers
  -> 10GiB playhead read-ahead
  -> 100GiB persistent cache
  -> trusted *.stremio.rocks:12470 media path
  -> Stremio player
```

## Real-World Resilience Evidence

The stack has now been exercised across multiple heavy 4K playback sessions, including two complete films.

Observed acceptance evidence includes:

- sustained playback of full 4K titles through the central `stremio-libtorrent-server` path;
- no repeatable server-side CPU, RAM, disk, or server-NIC bottleneck identified during the observed stalls;
- forward and backward seeking already validated during Phase 5;
- stop/reopen/resume already validated during Phase 5;
- desktop long-seek behaviour already validated;
- persistent torrent/cache activity on the Ubuntu server rather than on the TV client;
- `10GiB` read-ahead remains the configured playhead-priority target throughout the accepted deployment;
- `100GiB` cache behaviour and size-triggered LRU eviction were validated under real 4K load.

## Read-Ahead Acceptance

`STREMIOSRV_READAHEAD_BYTES=10GiB` remains the deployed configuration.

The acceptance result is operational rather than synthetic: sustained 4K playback completed successfully under normal real-world swarm/network variation. No artificial test was performed that deliberately throttled or interrupted torrent download throughput to prove a precise number of seconds/minutes of outage absorption.

Therefore the validated conclusion is:

- the configured `10GiB` read-ahead is accepted for the current deployment based on successful real-world playback;
- this does not claim that the full 10GiB window is always populated or that every possible throughput outage of a calculated duration is guaranteed to be absorbed.

## Cache Resilience

The current upstream-only cache policy remains:

```text
Read-ahead:       10GiB
Cache budget:     100GiB
Eviction grace:   1800s / 30 min upstream default
Download rate:    unlimited
Adaptive picking: false
```

Previous live testing under 4K load showed cache usage reaching approximately `101G`, after which the LRU evictor removed an older cached title while preserving the active stream, reducing usage to approximately `78G`.

This behaviour is accepted as the deployed cache-resilience model. The `100GiB` value remains a budget rather than a strict hard quota.

## Seek And Resume Acceptance

Seek/resume behaviour was already exercised during the client-acceptance phase and is carried forward into final Phase 7 acceptance:

- forward seek on the primary TV — PASS;
- backward seek on the primary TV — PASS;
- stop/reopen/resume — PASS;
- desktop seek to a distant point in a long title — PASS.

No additional repeat test was required for Phase 7 because the same deployed media path and configuration remained in use.

## Known Client-Side Limitation

The primary TV's built-in Ethernet interface is limited to `100 Mbit/s`.

Very high-bitrate UHD BluRay REMUX content can approach or exceed the practical throughput ceiling of that interface. Brief sub-second stalls observed under those conditions were previously correlated with the TV-side Fast Ethernet limit rather than a demonstrated failure of:

- the torrent engine;
- the 10GiB read-ahead configuration;
- the cache filesystem;
- server CPU/RAM;
- the server network interface.

This remains a known client-network limitation and is not a blocker for the Stremio/AIOStreams architecture.

## Controlled Throughput-Drop Test

A synthetic test that deliberately limits or interrupts BitTorrent download throughput was intentionally not performed.

Reason:

- the system has already completed two full 4K films under real operating conditions;
- seek/resume behaviour is already validated;
- the only observed playback limitation is the independently identified TV Ethernet ceiling;
- adding artificial shaping solely for a checklist item was not considered necessary for operational acceptance.

If future troubleshooting requires it, a controlled throughput-drop test can still be performed as a diagnostic exercise without reopening the architecture baseline.

## Phase 7 Acceptance

| Check | Result |
| --- | --- |
| Sustained full-length 4K playback through central server | PASS |
| Two complete 4K film sessions without server-side failure | PASS |
| `10GiB` read-ahead remains configured | PASS |
| Real-world read-ahead/resilience behaviour acceptable | PASS |
| `100GiB` cache + LRU eviction under 4K load | PASS |
| Forward seek | PASS |
| Backward seek | PASS |
| Stop/reopen/resume | PASS |
| Desktop long seek | PASS |
| Primary-TV high-bitrate REMUX behaviour | PASS with known `100 Mbit/s` client Ethernet limitation |
| Artificial controlled throughput-drop test | NOT RUN — not required for final operational acceptance |

## Final Project State

The original seven-phase Stremio/AIOStreams implementation roadmap is complete.

Completed:

1. Phase 1 — preflight;
2. Phase 2 — self-hosted AIOStreams deployment;
3. Phase 3 — central `stremio-libtorrent-server` deployment;
4. Phase 4 — P2P/non-debrid source and end-to-end torrent validation;
5. Phase 5 — TV/desktop client acceptance;
6. Phase 6 — peer-port and connectivity validation;
7. Phase 7 — large-file/read-ahead/resilience acceptance.

One independent operational follow-up remains outside the completed roadmap acceptance:

```text
Later add router forwarding:
6882/TCP+UDP -> 192.168.0.10:6882
```

When that is done, qBittorrent `6881/TCP+UDP` must remain unchanged and public forwarding must still exclude the Stremio web/API/media-management ports.
