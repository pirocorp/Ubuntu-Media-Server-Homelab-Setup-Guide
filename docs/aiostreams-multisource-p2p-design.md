# AIOStreams Multi-Source P2P Design

## Status

Design only. No runtime changes are made by this document.

## Goal

Keep the existing working AIOStreams/Torrentio/stremio-libtorrent-server architecture intact while adding a minimal, high-value supplementary P2P discovery layer.

The design deliberately avoids adding many overlapping addons. Each retained source must have a distinct discovery role and must ultimately yield usable torrent metadata / infoHash values that can be consumed by the existing P2P playback path.

## Locked baseline

The following existing components remain unchanged:

- AIOStreams v2.34.1
- Torrentio as the primary torrent source
- stremio-libtorrent-server 1.6.15 as the only torrent playback engine
- 10 GiB read-ahead
- 100 GiB persistent cache
- qBittorrent on 6881/TCP+UDP
- Stremio torrent engine on 6882/TCP+UDP
- direct trusted `*.stremio.rocks:12470` media path
- no debrid services

Torrentio is considered a locked baseline and is not subject to replacement in this project.

## Final v1 source topology

```text
                    +--------------------+
                    |     Torrentio      |
                    | targeted indexers  |
                    +---------+----------+
                              |
                              v
                       +-------------+
                       | AIOStreams   |
                       | aggregate /  |
                       | filter /     |
                       | deduplicate  |
                       | sort         |
                       +------+------+
                              ^
                              |
                    +---------+----------+
                    |       STorz        |
                    | torrent/hash DB    |
                    +----+----------+----+
                         ^          ^
                         |          |
               +---------+--+   +---+-----------+
               | BitMagnet  |   | DMM hashlists|
               | local DHT   |   | crowdsourced |
               | discovery   |   | hash coverage|
               +-------------+   +---------------+

                              |
                              v
                           Stremio
                              |
                              v
                 stremio-libtorrent-server
                              |
                              v
                       BitTorrent peers
```

## Component roles

### Torrentio

Role: fast, targeted public-indexer discovery for mainstream/current content.

Torrentio remains the primary torrent source. Existing Torrentio configuration must not be modified during the STorz rollout.

### STorz / StremThru Torz

Role: supplementary P2P source that consolidates BitMagnet and DMM-derived torrent metadata before exposing it to AIOStreams.

The v1 STorz deployment should not add additional public indexers that duplicate Torrentio providers. In particular, do not add YTS, 1337x, Rutracker, Nyaa, or similar providers already covered by Torrentio.

STorz should be used for two inputs only in v1:

1. the local BitMagnet instance;
2. DMM hashlists.

### BitMagnet

Role: autonomous local DHT discovery.

Current instance:

- version: v0.10.0
- self-hosted
- existing deployment remains authoritative for DHT crawling

StremThru currently supports BitMagnet v0.10.x, so the installed v0.10.0 version is compatible with the integration design.

The StremThru integration is not a live per-request query path. StremThru synchronizes BitMagnet torrent metadata into its own torrent database and subsequently parses/maps it for Torz results.

Expected data imported from BitMagnet includes:

- infoHash
- torrent title
- size
- seeders
- leechers
- private flag
- torrent file list

Only movie / TV content is imported by the current BitMagnet sync implementation, and torrents must contain a recognized video file (with VOB/ISO also accepted). This helps keep unrelated DHT content out of STorz results.

### DMM hashlists

Role: crowdsourced / historical torrent hash coverage that complements the local DHT view.

DMM is not intended as a low-latency discovery source. Its value is accumulated coverage, including hashes that the local BitMagnet crawler may not have observed.

## Deduplication model

Deduplication happens at two layers:

1. StremThru stores torrent information keyed by infoHash, so BitMagnet and DMM data for the same torrent converge on the same torrent record rather than producing independent copies.
2. AIOStreams remains the final cross-addon aggregation / filtering / sorting / deduplication layer for Torrentio and STorz output.

The preferred identity key for P2P torrents is infoHash.

## Freshness characteristics

The two source families intentionally have different roles:

- Torrentio: targeted, current, request-time discovery.
- BitMagnet via STorz: broad local DHT coverage, synchronized periodically.
- DMM via STorz: crowdsourced/historical coverage, synchronized periodically.

Current StremThru scheduling observed in the upstream code:

- BitMagnet sync: every 60 minutes
- DMM hashlist sync: every 6 hours
- torrent parsing: every 5 minutes
- IMDb torrent mapping: every 30 minutes

Therefore STorz is not expected to match Torrentio for immediate discovery of a newly published torrent. That is acceptable by design because Torrentio remains the fast path.

## BitMagnet integration requirements

StremThru enables the BitMagnet integration when both are configured:

- `STREMTHRU_INTEGRATION_BITMAGNET_BASE_URL`
- `STREMTHRU_INTEGRATION_BITMAGNET_DATABASE_URI`

The design should keep both BitMagnet and its PostgreSQL database on an internal Docker network accessible to StremThru. The PostgreSQL service must not be exposed publicly for this integration.

Where practical, use a dedicated least-privilege database user for StremThru if the required BitMagnet reads can be satisfied without write privileges.

## AIOStreams policy

AIOStreams remains the only top-level aggregation layer.

For STorz:

- use the P2P path;
- do not configure debrid services;
- do not add duplicated public indexers in STorz;
- preserve the existing AIOStreams quality/language/size policies initially;
- prefer infoHash-based deduplication;
- sort by seeder availability where useful, but do not initially impose an aggressive minimum-seeder cutoff because source freshness may differ.

## Explicitly excluded from v1

The following candidates were evaluated and are intentionally excluded or deferred:

### Comet

Excluded. It is unnecessary when StremThru can synchronize BitMagnet directly.

### CometNet

Excluded. It adds another metadata-discovery network and operational complexity without a demonstrated need once local BitMagnet + DMM coverage is present.

### TorrentsDB

Excluded from v1. Much of its useful mainstream coverage overlaps with Torrentio or with torrents likely to be discovered through BitMagnet. The remaining unique providers do not currently justify another permanent addon dependency.

It can be reconsidered only if empirical testing reveals a repeatable coverage gap.

### MediaFusion

Excluded from v1 because it is a much larger aggregation/scraping platform with substantial functional overlap and a higher operational footprint than this design requires.

### Peerflix

Excluded from v1 because its expected incremental value for the target library is limited and more regional/niche than the selected core.

## Implementation order

Implementation is intentionally incremental and must not disturb the existing Torrentio path.

1. Deploy/configure StremThru/STorz without changing Torrentio.
2. Enable DMM hashlist support.
3. Connect the existing BitMagnet v0.10.0 instance.
4. Verify BitMagnet sync completes and torrent records are parsed/mapped.
5. Add STorz P2P to AIOStreams.
6. Validate real STorz results expose usable infoHash values.
7. Validate playback through the existing stremio-libtorrent-server path.
8. Compare Torrentio-only versus Torrentio + STorz result sets for representative titles.
9. Keep the new source only if it adds useful unique playable torrents without materially degrading AIOStreams response time or result quality.

## Validation set

Use a representative test set rather than a single title. Include:

- current popular movie
- current TV episode
- older movie
- older TV episode
- 4K/UHD title
- title with many releases
- title with limited/rare availability

For each title record:

- Torrentio result count
- STorz result count
- unique STorz infoHashes after deduplication
- duplicate infoHashes
- metadata parsing quality
- seeder/peer metadata presence
- response latency
- whether a selected unique STorz result actually starts through stremio-libtorrent-server

No fixed percentage of unique hashes is required. A source is useful if it reliably adds meaningful playable coverage without causing operational or metadata problems.

## Acceptance criteria

STorz remains in the permanent core if all of the following hold:

- returns direct P2P torrent results with usable infoHash values;
- at least some representative titles gain meaningful unique playable coverage;
- duplicate results are handled correctly;
- movie/episode mapping is reliable enough for normal use;
- AIOStreams filtering and sorting continue to work correctly;
- no unnecessary media proxy hop is introduced;
- playback continues exclusively through stremio-libtorrent-server;
- existing Torrentio behavior remains unchanged;
- operational latency and error rate remain acceptable.

If STorz adds no meaningful incremental coverage, it can be removed without changing the established Torrentio playback architecture.

## Deferred ideas

Only reconsider additional sources after measuring the final v1 core.

Potential future investigations:

- TorrentsDB for a narrowly selected provider subset (for example 1lou or LimeTorrents) if a specific coverage gap is observed;
- alternative Torznab indexers only where they add demonstrably unique content;
- Comet/CometNet only if STorz + BitMagnet proves insufficient.

The default direction is to keep the source set small rather than add addons pre-emptively.
