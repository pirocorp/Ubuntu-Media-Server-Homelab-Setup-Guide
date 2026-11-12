# AIOStreams Language-Priority Sorting — Live Acceptance (2026-11-12)

Status: **Implemented and functionally validated — accepted as-is**  
Date: 2026-11-12  
Scope: Live AIOStreams/Torrentio P2P configuration; language-priority ranking; end-to-end playback on the TV and a second Stremio client. This is a configuration acceptance note, not a claim that every release's audio language can be identified perfectly.

## Configuration of Record

- Runtime: self-hosted **AIOStreams v2.34.1**, installed as a Stremio addon.
- Stream source: **Torrentio** in **P2P/non-debrid** mode; no paid debrid provider.
- **Live exported configuration, saved after a successful UI save:** [`configs/aiostreams-config-2026-11-12-updated.json`](./configs/aiostreams-config-2026-11-12-updated.json) (originally introduced in merged PR #34).
- **Pre-change recovery snapshot:** [`backups/aiostreams-config-2026-11-12.19-15-28.json`](./backups/aiostreams-config-2026-11-12.19-15-28.json) (introduced in merged PR #33).
- The live configuration was imported/edited through the **already installed AIOStreams addon** and saved successfully. Do not confuse it with any separate/test configuration. The JSON file is a snapshot of the accepted configuration; it is not an automatic deployment mechanism.

Only the deployed, tested state is documented here. **No tuning, re-import, or runtime change is requested by this acceptance note.**

### Implemented Ranking

The active global sort keys, all descending and in effective priority order, are:

1. `streamExpressionScore`
2. `language`
3. `subtitle`
4. `resolution`
5. `quality`
6. `seeders`
7. `visualTag`
8. `size`

The four enabled ranked stream expressions assign the following **additive** scores when matched:

| Score | Expression intent | Actual rule basis |
| ---: | --- | --- |
| +1000 | Prefer Bulgarian audio | Bulgarian language from `mediaInfoQuality(streams, 'probe', 'indexer', 'addon')`, **or** filename matching explicit BG Audio/BG Dub variants |
| +100 | Prefer Bulgarian subtitles | `subtitle(streams, 'Bulgarian')`, **or** filename BG Sub/BG Subs variants |
| +50 | Lower-confidence Bulgarian fallback | Bulgarian from `mediaInfoQuality(streams, 'unknown')`, **or** filename `ENG+BG` |
| +10 | English fallback | `language(streams, 'English')` |

Preferred languages and subtitles are both **Bulgarian, then English**. Resolutions prioritize **2160p > 1440p > 1080p** and lower fallbacks; quality preference begins **BluRay REMUX > BluRay > WEB-DL**. Exclude CAM/TS/TC/SCR and 3D; keep deduplication enabled. The `seeders` sort key is active **after** language/subtitle/resolution/quality in this newer configuration (unlike the original Phase 4 baseline).

The precise rule expressions and full remaining options are in the linked JSON snapshot; do not reconstruct a replacement configuration solely from this prose. Torrentio remains the only enabled stream-source preset, and the AIOStreams media proxy is not used for the P2P byte path.

## Live Acceptance Evidence

### Test A — Disclosure Day (2026), primary TV

- AIOStreams/Torrentio returned several results labelled `Bulgarian` **ahead of English-only releases**, including a **2160p BluRay REMUX (~84 GB)** before lower-language-priority results.
- The selected torrent reached **stremio-libtorrent-server**, which created cache/media and `.resume` / `.parts` state under `/mnt/data/stremio-libtorrent-server`.
- The Stremio statistics screen showed a snapshot of **5.27 GB downloaded (6.27%), 116 peers and 11.00 MB/s**. These are measurements at one moment, not throughput guarantees or a complete download.
- Initial client/player investigation: **ExoPlayer did not start this release**; **VLC produced video but initially lacked sound**; **MPV produced momentary sound then stopped**. After **restarting the TV and selecting VLC**, video **and sound** worked. Root cause of the transient player/audio issue was not established.
- **Bulgarian subtitles were actually present. Bulgarian audio was not present** in the player's audio-track list.

Result: **PASS** for stream discovery, expected language-labelled ranking, centralized P2P cache/download evidence and eventual TV playback with sound/subtitles; **NOT a pass** for the claim that the `Bulgarian` label guarantees Bulgarian *audio*. No formal sustained-playback test of this exact title was reported.

### Test B — National Treasure: Book of Secrets (2007), another Stremio client

- A **1080p WEB-DL (~4.74 GB)** result labelled `Bulgarian | English | Multi` appeared **above 2160p releases marked English/Russian**. This verifies the intended language-before-resolution ordering for this title.
- Playback succeeded on the second client.
- Its **Audio Tracks** menu explicitly listed **`Български` (selected) and `English`**, confirming **real Bulgarian audio** for this release.
- Bulgarian subtitles were not checked for this title. The screenshots establish client playback and ranking; they do not independently prove that this second client's torrent engine was the centralized server rather than a local engine.

Result: **PASS** for finding and prioritizing a release with **confirmed Bulgarian audio**, and playable output on a second client.

## Acceptance Decision And Known Limitation

**Accepted as implemented and functionally validated on 2026-11-12. Keep the present live configuration unchanged.**

The language-first ordering is useful in practice: one release with verified Bulgarian audio was prioritized over 4K English/Russian alternatives, while another Bulgarian-labelled release provided Bulgarian subtitles only. Hence:

- A `Bulgarian` badge from torrent/addon/indexer metadata **must not be interpreted as proof of a Bulgarian audio track**. It may reflect subtitles or filename-level language signals.
- The `+1000` expression currently allows `indexer` and `addon` data as well as `probe`; **false positives for audio remain possible**. This is a known metadata-classification limitation, not a streaming failure.
- Do **not** silently replace the working rule with a probe-only version: that alternative was discussed but **not deployed or validated** and might demote genuine Bulgarian-audio releases.
- Heavy HEVC/Dolby Vision/TrueHD REMUX playback may be client/player dependent. In the observed TV test, VLC after a TV restart worked, but the exact cause of the preceding Exo/MPV/VLC failures is unknown.

Further metadata/ranking refinement is **optional future work**, not required for acceptance of the current working baseline.

## Recovery / Revalidation

1. Use the linked **updated** exported JSON as the configuration-of-record for any later manual import or comparison; retain the original **pre-change** backup for rollback.
2. If re-importing, use the correct **existing installed addon configuration**; do not create an unrelated test instance or leak any addon configuration URL containing private tokens.
3. Confirm the global sort order and enabled four expressions using the JSON, then **Save** and ensure Stremio loads the intended addon.
4. Functional smoke tests: check one known Bulgarian-audio release (e.g., the tested *National Treasure: Book of Secrets* candidate) ranks over higher-resolution non-Bulgarian releases; verify `Български` in **Audio Tracks**. Check a subtitle-only labelled release separately to avoid mistaking metadata for audio proof.
5. Verify P2P transfer/streaming independently from the language metadata using Stremio statistics and `stremio-libtorrent-server` cache observations. AIOStreams selects sources; the torrent server downloads media.

Related: [AIOStreams runbook](./README.md) · [stremio-libtorrent-server runbook](../stremio-libtorrent-server/README.md) · [Phase 4 baseline](../../roadmaps/stremio-aiostreams/phase-4-completion.md).
