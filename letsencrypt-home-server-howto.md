# How To: Replace a Self-Signed Certificate with Let's Encrypt for Home Servers

## Goal

Replace a local self-signed certificate setup with a trusted Let's
Encrypt certificate.

Before:

``` text
https://service.home
        |
Self-signed certificate
        |
Browser SSL warning
```

After:

``` text
https://service.example.com
        |
Let's Encrypt certificate
        |
Trusted HTTPS
```

------------------------------------------------------------------------

## Architecture

``` mermaid
flowchart TD
    A[Client Browser / Mobile App]
    -->|HTTPS request| B[service.example.com]

    B --> C[Local DNS Resolver]

    C -->|Returns private IP| D[192.168.0.10]

    D --> E[Nginx Proxy Manager]

    E -->|Trusted Let's Encrypt SSL| F[Wildcard Certificate]

    E -->|Internal HTTP| G[Docker Application]
```

------------------------------------------------------------------------

# 1. Requirements

You need:

-   Real domain name
-   DNS provider with API support
-   Reverse proxy
-   Internal applications

Example:

``` text
Domain:
example.com

Server:
192.168.0.10

Apps:
Nextcloud :8090
Portainer :9443
AdGuard   :3000
```

------------------------------------------------------------------------

# 2. Configure Your Domain

Use a public domain instead of local-only names.

Do not use:

``` text
server.home
app.local
myserver.lan
```

Use:

``` text
nextcloud.example.com
plex.example.com
photos.example.com
```

Example:

``` text
nextcloud.pirocorp.com
```

------------------------------------------------------------------------

# 3. Create DNS Provider API Token

Example with Cloudflare:

Create a token with:

``` text
Zone -> DNS -> Edit
Zone -> Zone -> Read
```

Limit access to:

``` text
example.com
```

Save the generated token.

------------------------------------------------------------------------

# 4. Create Let's Encrypt Wildcard Certificate

In Nginx Proxy Manager:

``` text
SSL Certificates
    |
    Add Certificate
    |
    Let's Encrypt via DNS
```

Domains:

``` text
example.com
*.example.com
```

Example:

``` text
pirocorp.com
*.pirocorp.com
```

Select:

``` text
DNS Provider: Cloudflare
```

Credentials:

``` ini
dns_cloudflare_api_token=YOUR_TOKEN
```

Propagation:

``` text
120 seconds
```

Save.

------------------------------------------------------------------------

# 5. Configure Local DNS

Point your domain names internally.

Example with AdGuard / Pi-hole:

``` text
*.example.com -> 192.168.0.10
```

Example:

``` text
*.pirocorp.com -> 192.168.0.10
```

Test:

``` bash
nslookup nextcloud.example.com
```

Expected:

``` text
Address: 192.168.0.10
```

------------------------------------------------------------------------

# 6. Create Proxy Host

Example: Nextcloud

Nginx Proxy Manager:

``` text
Hosts
 |
Proxy Hosts
 |
Add Proxy Host
```

Configuration:

``` text
Domain:
nextcloud.example.com

Scheme:
http

Forward Host:
192.168.0.10

Forward Port:
8090
```

Enable:

``` text
Block Common Exploits
Websocket Support
```

------------------------------------------------------------------------

# 7. Enable SSL

SSL tab:

Select:

``` text
example.com, *.example.com
```

Enable:

``` text
Force SSL
HTTP/2
```

Wait before enabling HSTS until everything is tested.

------------------------------------------------------------------------

# 8. Update Application Configuration

Applications should know their public URL.

Example Nextcloud:

Old:

``` env
NEXTCLOUD_DOMAIN=nextcloud.home
```

New:

``` env
NEXTCLOUD_DOMAIN=nextcloud.example.com
```

Behind reverse proxy:

``` env
OVERWRITEPROTOCOL=https
OVERWRITEHOST=nextcloud.example.com
OVERWRITECLIURL=https://nextcloud.example.com
```

Restart:

``` bash
docker compose down
docker compose up -d
```

------------------------------------------------------------------------

# Request Flow

``` mermaid
sequenceDiagram
    participant User
    participant DNS
    participant NPM as Nginx Proxy Manager
    participant App

    User->>DNS: Resolve nextcloud.example.com
    DNS-->>User: 192.168.0.10

    User->>NPM: HTTPS request
    NPM-->>User: Let's Encrypt certificate

    NPM->>App: Forward HTTP traffic
    App-->>NPM: Response
    NPM-->>User: Secure response
```

------------------------------------------------------------------------

# Benefits

-   No browser warnings
-   No custom CA installation
-   Works on Windows, Linux, iPhone, Android
-   Automatic renewal
-   One wildcard certificate protects all services

Example services:

``` text
nextcloud.example.com
plex.example.com
portainer.example.com
photos.example.com
```

All covered by:

``` text
*.example.com
```
