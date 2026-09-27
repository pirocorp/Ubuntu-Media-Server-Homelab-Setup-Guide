# Phase 4 Completion — AIOStreams Sources And End-to-End P2P Validation

Status: Complete  
Date: 2026-09-26  
Scope: Torrentio P2P/non-debrid source configuration, AIOStreams addon installation, end-to-end torrent proof, initial TV validation, and cache-policy tuning

## Outcome

Phase 4 is complete.

The live path is now validated as:

```text
Stremio client
  -> AIOStreams v2.34.1
  -> Torrentio P2P / non-debrid
  -> torrent infoHash + fileIdx
  -> stremio-libtorrent-server 1.6.15
  -> BitTorrent peers
  -> /mnt/data/stremio-libtorrent-server cache
  -> Stremio player
```

No paid debrid provider is configured and AIOStreams built-in media proxy remains disabled.

## AIOStreams Source Configuration

Torrentio is configured as the initial upstream source in P2P/non-debrid mode.

Validated policy:

- resource: `Stream` only;
- no debrid services;
- no provider/service restrictions required for the baseline;
- no multiple instances;
- CAM/SCR/TS/TC releases excluded;
- 3D visual tag excluded;
- preferred resolutions include 2160p, 1440p, 1080p and lower fallbacks;
- preferred qualities prioritize BluRay REMUX, BluRay, WEB-DL, WEBRip and HDRip classes;
- AIOStreams deduplication remains enabled;
- built-in media proxy is not used (`proxiedServices` and `proxiedAddons` empty);
- background stream preloading and next-episode precache remain disabled;
- autoplay matching uses resolution and quality attributes;
- statistics output is enabled for troubleshooting.

The generated private AIOStreams configuration URL/token is intentionally not recorded in Git.

## Stremio Addon Baseline

The configured AIOStreams addon was installed into the Stremio account and synchronized to clients.

During troubleshooting, optional third-party addons were removed because one or more of them polluted the visible stream/detail flow with deep-link style entries. The stable baseline retained:

- Cinemeta;
- Local Files;
- AIOStreams.

After this cleanup, normal movie selections invoked AIOStreams and returned Torrentio P2P streams with usable `infoHash` and `fileIdx` data.

## API And Stream Result Validation

AIOStreams stream API responses were inspected directly and confirmed normal P2P results for multiple titles. Returned streams contained torrent hashes, file indexes, filenames, video sizes, and seeder information.

Examples observed during validation included:

- Guardians of the Galaxy;
- Mayday;
- The Whisper Man;
- Project Hail Mary;
- The Love Hypothesis;
- Back Roads;
- Obsession.

This closed the Phase 4 requirement that AIOStreams/Torrentio provide torrent information usable by the central streaming engine.

## End-to-End Torrent Engine Proof

Desktop playback of `Mayday` produced live cache activity under:

```text
/mnt/data/stremio-libtorrent-server
```

The server created and updated:

- the movie media file;
- the matching `.fastresume` record;
- `.resume/index.json`;
- the torrent `.parts` file;
- `.evictor-owner`.

This proves that the selected torrent was downloaded and cached by `stremio-libtorrent-server` rather than by the local desktop client.

## Cache Behaviour Discovered During Validation

The `10GiB` read-ahead value is a playhead priority window, not a maximum amount downloaded. Upstream `stremio-libtorrent-server` continues filling the selected file after playback closes until the wanted file completes.

A custom fork was explicitly rejected to avoid maintaining patched images across upstream releases.

The upstream-only policy was therefore changed to:

```text
STREMIOSRV_READAHEAD_BYTES=10GiB
STREMIOSRV_CACHE_SIZE=100GiB
STREMIOSRV_CACHE_EVICT_GRACE=1800   # upstream default, 30 minutes
```

`STREMIOSRV_CACHE_EVICT_GRACE` is not a TTL. It prevents recently served/modified entries from eviction. Actual cleanup remains size-triggered LRU eviction.

The active cache can temporarily exceed `100GiB`; the budget is not a hard filesystem quota.

### Live eviction proof

During 4K playback the cache reached approximately `101G` with these media entries:

```text
The Whisper Man       ~21G
The End Of Oak Street ~22G
Mayday                ~24G
In The Grey           ~36G (active)
```

The evictor then removed the older `Mayday` cache entry while preserving the active `In The Grey` stream, reducing total usage to approximately `78G`.

This validates the chosen `100GiB` upstream-only cache policy.

If routine use begins to include individual REMUX files larger than `100GiB`, revisit the cache budget because upstream recommends a budget larger than the largest expected file.

## TV DNS Compatibility Workaround

The primary TV did not reliably use the local AdGuard DNS rewrite even when a manual DNS server was configured.

To keep hostname-based access without hard-coded service IPs, Cloudflare public DNS now contains:

```text
*.pirocorp.com  A  192.168.0.10
Proxy status: DNS only
```

This publishes only an RFC1918/private destination address; it does not make the services Internet-routable. It allows clients that bypass local DNS to resolve homelab service hostnames while Nginx Proxy Manager continues routing by Host header.

Local explicit AdGuard rewrites remain the normal LAN DNS model.

External resolver validation:

```text
aio.pirocorp.com -> 192.168.0.10
```

was confirmed through Google DNS (`8.8.8.8`).

## Initial TV Playback Validation

The TV successfully reached AIOStreams and played P2P content through the central server path.

Observed examples included:

- a completed ~22.61 GB stream with active peers;
- `In The Grey` 2160p UHD BluRay REMUX downloading at roughly 26 MB/s during early playback with more than 30 peers.

This is sufficient to close Phase 4 and establishes the starting point for Phase 5 client acceptance and Phase 7 resilience testing.

### Primary TV Ethernet bottleneck identified

Sustained playback testing of `In The Grey` exposed brief sub-second playback stalls in a high-bitrate section. The server, torrent engine, storage path, and LAN diagnostics did not show a correlated fault:

- `stremio-libtorrent-server` logs showed no relevant warning/error around the event;
- CPU and RAM retained substantial headroom;
- disk sampling on `/dev/sdc` showed no saturation or deep queue at the stall;
- the server Ethernet interface showed no TX errors or drops;
- continuous ping to the TV showed no packet loss or latency spike correlated with the stall.

The primary TV is connected through a `100 Mbit/s` Ethernet interface. `ffprobe` analysis of the affected two-minute interval (`01:21:45`–`01:23:45`) showed the video stream alone running mostly around `80–100 Mbit/s`, including observed peaks of `126.99 Mbit/s` and `115.32 Mbit/s`. These figures exclude audio/container overhead.

Live `iftop` measurement of the actual server-to-TV flow showed approximately `92–93 Mbit/s` sustained with roughly `95.8–97.1 Mbit/s` observed peak throughput, effectively reaching the practical ceiling of the TV's Fast Ethernet link.

The operational bottleneck is therefore recorded as the primary TV's `100 Mbit/s` Ethernet connection for very high-bitrate UHD BluRay REMUX playback. The brief stalls are consistent with the client buffer being unable to absorb sustained/burst demand above the available Ethernet throughput.

The TV's Wi-Fi interface is faster but considered less stable. No Wi-Fi A/B test was performed in this session because the TV requires physically disconnecting the Ethernet cable before switching to Wi-Fi. No server-side tuning is planned for this symptom at this point.

## Phase 4 Acceptance

| Check | Result |
| --- | --- |
| Torrentio configured through AIOStreams | PASS |
| P2P/non-debrid only | PASS |
| AIOStreams returns usable infoHash/fileIdx streams | PASS |
| AIOStreams addon installed in Stremio account | PASS |
| Optional conflicting source addons removed from stable baseline | PASS |
| Desktop playback invokes AIOStreams | PASS |
| Central server cache activity proves server-side torrent download | PASS |
| Built-in AIOStreams media proxy remains off | PASS |
| `10GiB` read-ahead preserved | PASS |
| Cache budget tuned to `100GiB` without custom image | PASS |
| LRU eviction observed above budget | PASS |
| Primary TV can resolve and use `aio.pirocorp.com` | PASS |
| Initial TV P2P playback through central path | PASS |
| Primary TV 100 Mbit/s Ethernet limit characterized | PASS |

## Remaining Work

Phase 4 is closed. Remaining roadmap work is:

- Phase 5 — complete primary-TV/client acceptance, including seek/resume behaviour and sustained playback confidence; the known `100 Mbit/s` TV Ethernet ceiling should be treated as a client-network limitation for very high-bitrate REMUX files;
- Phase 6 — router forwarding/inbound peer validation for `6882/TCP+UDP`;
- Phase 7 — formal large-file read-ahead, seek, and resilience testing.
