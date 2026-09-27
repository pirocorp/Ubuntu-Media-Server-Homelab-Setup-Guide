# TV Ethernet Bottleneck Diagnosis

Status: Validated  
Date: 2026-09-27  
Scope: Reproduce and verify the primary-TV Fast Ethernet bottleneck during very high-bitrate UHD BluRay REMUX playback.

## Purpose

This runbook documents how the `100 Mbit/s` Ethernet bottleneck on the primary TV was identified, so the same diagnosis can be repeated after network, TV, player, or server changes.

The observed symptom was a brief sub-second playback stall during a high-bitrate 2160p UHD BluRay REMUX. The goal is to distinguish a client-link bottleneck from torrent starvation, server CPU/RAM pressure, storage latency, server-NIC faults, or ordinary LAN packet loss.

## Test Conditions

Use the same media source throughout the test. Do not change quality, release, player, or stream source between measurements.

Validated test case:

- media: `In The Grey` 2160p UHD BluRay REMUX HDR / TrueHD Atmos / H.265;
- affected test window: approximately `01:21:45`–`01:23:45`;
- server: `192.168.0.10`;
- TV observed during the validation: `192.168.0.207`;
- Stremio trusted HTTPS media port: `12470/TCP`;
- server LAN interface: `enp2s0`;
- cache disk: `/dev/sdc`;
- TV wired interface: `100 Mbit/s` Fast Ethernet.

Treat the TV IP as dynamic unless it is reserved in DHCP; rediscover it when repeating the test.

## Diagnostic Tools

Install only the missing packages:

```bash
sudo apt update
sudo apt install -y sysstat tcpdump iftop ffmpeg
```

The tools used are:

- `iostat` from `sysstat` for storage latency/queueing;
- `tcpdump` to identify the active TV connection;
- `ping` for client-path latency/loss correlation;
- `ffprobe` from `ffmpeg` for media bitrate analysis;
- `iftop` for live server-to-TV throughput.

## 1. Confirm Server Headroom

While the problem stream is playing, inspect the torrent-engine container:

```bash
docker stats --no-stream stremio-libtorrent-server
```

During the validated event, CPU and RAM retained substantial headroom. A playback stall without a corresponding resource spike weakens the server-resource hypothesis.

Also inspect recent server logs:

```bash
docker logs --since 10m stremio-libtorrent-server 2>&1 | grep -Ei 'error|warn|timeout|stall|exception|failed|slow'
```

The validated stalls produced no relevant warning/error output.

## 2. Check Storage Latency During Playback

Run:

```bash
iostat -xz sdc 1
```

Watch especially:

- `r_await` / `w_await`;
- `aqu-sz`;
- `%util`;
- system `%iowait`.

During the validated stall, `/dev/sdc` showed read latency in roughly the tens-of-milliseconds range, but no sustained `100%` utilization and no deep queue. This did not match a saturated storage device.

Do not conclude that a single elevated `r_await` value proves a disk bottleneck; correlate it with queue depth, utilization, and the actual playback event.

## 3. Check Server Ethernet Counters

Run:

```bash
watch -n 1 'ip -s link show enp2s0'
```

For the media path `server -> TV`, focus on TX counters:

- `errors`;
- `dropped`;
- `carrier`;
- `collisions`.

The validation showed no TX errors or TX drops correlated with the stalls.

RX drops are not by themselves evidence of a problem in the server-to-TV media path because the media stream leaves the server on TX.

## 4. Identify the Active TV IP

If the TV address is unknown, capture the Stremio media connection:

```bash
sudo tcpdump -ni enp2s0 tcp port 12470 -c 20
```

Look for traffic of the form:

```text
192.168.0.10.12470 > 192.168.0.X.<client-port>
```

During the validated test, the active TV was `192.168.0.207`.

The short capture also reported:

```text
0 packets dropped by kernel
```

## 5. Correlate LAN Latency With the Stall

Run a frequent timestamped ping to the TV:

```bash
ping -D -i 0.2 192.168.0.207
```

Keep it running while reproducing the affected scene.

During the validated stalls there was no packet loss and no latency spike correlated with the event. Typical samples were below approximately `20 ms`.

A clean ping does not prove that TCP throughput is sufficient; it only helps rule out a simple latency/loss event.

## 6. Locate the Active Media File

For the validated test:

```bash
find /mnt/data/stremio-libtorrent-server -type f \( -iname "*In*The*Grey*.mkv" -o -iname "*In*The*Grey*.mp4" \) -print
```

Validated file:

```text
/mnt/data/stremio-libtorrent-server/In.The.Grey.2026.2160p.UHD.BluRay.REMUX.HDR.MULTi.TrueHD.Atmos.H265-BTM/In.The.Grey.2026.2160p.UHD.BluRay.REMUX.HDR.MULTi[Ben The Men].mkv
```

## 7. Measure Video Bitrate Around the Problem Window

For a two-minute window centered near `01:22:45`, run:

```bash
ffprobe -v error \
-read_intervals "01:21:45%+120" \
-select_streams v:0 \
-show_entries packet=pts_time,size \
-of csv=p=0 \
"/mnt/data/stremio-libtorrent-server/In.The.Grey.2026.2160p.UHD.BluRay.REMUX.HDR.MULTi.TrueHD.Atmos.H265-BTM/In.The.Grey.2026.2160p.UHD.BluRay.REMUX.HDR.MULTi[Ben The Men].mkv" \
| awk -F',' '{s=int($1); b[s]+=$2} END {for(s in b) printf "%02d:%02d:%02d  %7.2f Mbps\n", int(s/3600), int((s%3600)/60), s%60, b[s]*8/1000000}' \
| sort
```

Important observed values included:

```text
01:21:52  124.43 Mbps
01:22:11  126.99 Mbps
01:22:26  115.32 Mbps
```

The video stream alone spent much of the interval around `80–100 Mbit/s`.

These values do not include the full muxed-stream overhead such as audio and container data, so the required transport bandwidth can be higher than the `v:0` values alone.

## 8. Measure Actual Server-to-TV Throughput

Run:

```bash
sudo iftop -i enp2s0 -f "host 192.168.0.207"
```

Repeat the same playback section without changing the source.

During the validated test, the server-to-TV flow showed approximately:

```text
~92–93 Mbit/s sustained
~95.8–97.1 Mbit/s observed peak
```

This is effectively the practical ceiling of a `100 Mbit/s` Fast Ethernet link after protocol overhead.

## Interpretation / Acceptance Criteria

Record the TV Ethernet link as the operational bottleneck when all of the following are true in the same test:

1. the media bitrate reaches or exceeds the practical Fast Ethernet ceiling;
2. `iftop` shows the server-to-TV flow pinned near that ceiling;
3. the stall occurs without correlated CPU/RAM exhaustion;
4. the storage device is not saturated or deeply queued;
5. the server interface has no correlated TX errors/drops;
6. ping shows no correlated packet loss or major latency spike;
7. torrent download/buffer state is otherwise healthy.

For the validated `In The Grey` REMUX test, these conditions were met. The primary TV's `100 Mbit/s` Ethernet interface is therefore the recorded client-side bandwidth bottleneck for very high-bitrate UHD REMUX playback.

## Optional A/B Confirmation

A useful follow-up is to disconnect the Ethernet cable, switch the same TV to a faster 5 GHz Wi-Fi connection, and replay the exact same source and scene.

This A/B test was **not performed** during the validated session because the TV does not allow direct switching to Wi-Fi while Ethernet remains connected. The TV Wi-Fi interface is known to be faster but less stable, so it is not currently recorded as the preferred permanent transport.

Do not change server settings solely to compensate for this symptom unless a later test disproves the client-link bottleneck.
