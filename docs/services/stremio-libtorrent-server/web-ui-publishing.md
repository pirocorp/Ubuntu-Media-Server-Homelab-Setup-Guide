# Stremio Web UI Publishing Through AdGuard And Nginx Proxy Manager

Status: Implemented  
Purpose: Document the validated internal HTTPS publishing path for the Stremio Web UI served by `stremio-libtorrent-server`.  
Depends on: [stremio-libtorrent-server runbook](./README.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md)  
Related docs: [AIOStreams](../aiostreams/README.md), [Current state](../../overview/current-state.md), [Stremio implemented architecture](../../overview/stremio-streaming-architecture.md)

## Scope

This runbook documents only the browser-facing Stremio Web UI.

The deployed `stremio-libtorrent-server` exposes two different client-facing concepts that must not be confused:

```text
Web UI
  https://stremio.pirocorp.com
  -> AdGuard Home
  -> Nginx Proxy Manager
  -> http://192.168.0.10:8081

Media / streaming-server endpoint
  https://<generated>.stremio.rocks:12470
  -> direct trusted HTTPS
  -> stremio-libtorrent-server
```

The Web UI is published through the normal homelab DNS/reverse-proxy pattern for convenience. The media path remains direct and does not traverse Nginx Proxy Manager.

## Current Deployment

| Item | Value |
| --- | --- |
| Preferred Web UI URL | `https://stremio.pirocorp.com` |
| AdGuard rewrite | `stremio.pirocorp.com -> 192.168.0.10` |
| NPM upstream scheme | `http` |
| NPM upstream host | `192.168.0.10` |
| NPM upstream port | `8081` |
| Container Web UI port | `8080/tcp` |
| Host mapping | `192.168.0.10:8081 -> 8080/tcp` |
| TLS certificate | existing `pirocorp.com, *.pirocorp.com` Let's Encrypt certificate |
| Force SSL | enabled |
| HTTP/2 | enabled |
| HSTS | disabled |
| WebSockets | enabled |
| Cache Assets | disabled |
| Block Common Exploits | enabled |

The final HTTPS path has been validated with both a browser and `curl`, returning `HTTP 200`.

## Architecture

```text
Browser / trusted client
        |
        | https://stremio.pirocorp.com
        v
AdGuard Home explicit DNS rewrite
        |
        | stremio.pirocorp.com -> 192.168.0.10
        v
Nginx Proxy Manager :443
        |
        | HTTP
        v
192.168.0.10:8081
        |
        | Docker port mapping
        v
stremio-libtorrent-server :8080
        |
        v
Stremio Web UI
```

This path is for the Stremio browser interface only.

The actual torrent media stream remains:

```text
Stremio client
        |
        v
https://<generated>.stremio.rocks:12470
        |
        v
stremio-libtorrent-server
```

Do not replace the direct `*.stremio.rocks:12470` media route with NPM unless there is a demonstrated requirement and Range/seek behavior is retested.

## 1. Confirm Direct Web UI

Before creating DNS or reverse-proxy configuration, confirm that the Web UI already works directly:

```text
http://192.168.0.10:8081
```

The Compose mapping is:

```text
192.168.0.10:8081 -> 8080/tcp
```

The host port is `8081` because qBittorrent already uses host port `8080`.

## 2. Add The AdGuard Home DNS Rewrite

Open:

```text
AdGuard Home -> Filters -> DNS rewrites
```

Add:

```text
Domain: stremio.pirocorp.com
Answer: 192.168.0.10
```

The homelab intentionally uses explicit rewrites rather than wildcard local DNS.

Validate from the Ubuntu server or another client using AdGuard DNS:

```bash
getent hosts stremio.pirocorp.com
```

Validated result:

```text
192.168.0.10    stremio.pirocorp.com
```

## 3. Create The Nginx Proxy Manager Host

Open:

```text
Nginx Proxy Manager -> Hosts -> Proxy Hosts -> Add Proxy Host
```

### Details

Use:

```text
Domain Names:        stremio.pirocorp.com
Scheme:              http
Forward Hostname/IP: 192.168.0.10
Forward Port:        8081
Access List:         Publicly Accessible
Cache Assets:        OFF
Block Common Exploits: ON
Websockets Support:  ON
```

`Publicly Accessible` here is the NPM access-list mode and means that NPM is not adding an authentication gate. It does not by itself create public DNS or router exposure. The hostname is currently supplied by the internal AdGuard rewrite.

### SSL

Select the existing certificate covering:

```text
pirocorp.com
*.pirocorp.com
```

Enable:

```text
Force SSL:      ON
HTTP/2 Support: ON
HSTS:           OFF
```

No Custom Locations or Advanced configuration were required for the validated deployment.

## 4. Validate The Final HTTPS Endpoint

From the Ubuntu server:

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  https://stremio.pirocorp.com/
```

Validated result:

```text
HTTP 200
```

The same URL successfully loaded the Stremio Web UI in a browser:

```text
https://stremio.pirocorp.com
```

## 5. Keep Web UI And Media Paths Separate

The convenience hostname is:

```text
https://stremio.pirocorp.com
```

It is for the Stremio Web UI.

The Stremio streaming-server endpoint remains the instance-generated trusted URL under:

```text
https://<generated>.stremio.rocks:12470
```

Reasons for keeping the media route direct:

- the upstream project already provides trusted HTTPS for Stremio clients;
- large media transfers avoid an unnecessary NPM proxy hop;
- server-side read-ahead and cache remain directly connected to the player;
- Range/seek behavior can be validated against the streaming service itself;
- the Web UI hostname can change independently from the streaming-server TLS identity.

## Security Notes

- Do not create public DNS for `stremio.pirocorp.com` unless public exposure is explicitly intended and reviewed.
- Do not forward `8081/tcp`, `11470/tcp`, or `12470/tcp` from the Internet router as general public services.
- Only `6882/TCP+UDP` is a candidate for future public inbound forwarding for BitTorrent peer connectivity.
- The generated `*.stremio.rocks:12470` endpoint resolves to the configured LAN address and is not a replacement for public service publishing.
- Keep `certificates.pem` and generated TLS state out of Git.

## Troubleshooting

### DNS Does Not Resolve

Check:

```bash
getent hosts stremio.pirocorp.com
```

Expected:

```text
192.168.0.10    stremio.pirocorp.com
```

If it does not resolve, verify that the client is actually using AdGuard Home for DNS and that the rewrite is enabled.

### HTTPS Returns 502 Or 504

First confirm direct access:

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  http://192.168.0.10:8081/
```

Then test from inside the NPM container if required:

```bash
docker exec nginx-proxy-manager \
  curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  --connect-timeout 5 http://192.168.0.10:8081/
```

If direct host access works but NPM cannot connect, inspect UFW and Docker network reachability.

### Browser Loads But Streaming Fails

Do not troubleshoot the Web UI proxy first. Playback uses the separate streaming-server path. Verify the generated trusted endpoint on `12470` and the Stremio client streaming-server configuration.

## Operational Validation Checklist

```bash
getent hosts stremio.pirocorp.com
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://192.168.0.10:8081/
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' https://stremio.pirocorp.com/
docker compose -f /srv/docker/stremio-libtorrent-server/compose.yaml ps
```

Expected final state:

```text
DNS -> 192.168.0.10
Direct Web UI -> HTTP 200
NPM HTTPS Web UI -> HTTP 200
stremio-libtorrent-server -> healthy
```
