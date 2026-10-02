# Roadmaps

Status: Mixed
Purpose: Track planned additions and deployed roadmap designs that remain useful as architecture references.
Related docs: [Overview](../overview/README.md), [Services](../services/README.md)

## Implemented Roadmap Designs

- [Remote access over VPN roadmap](./remote-access/README.md)
- [Homepage dashboard architecture roadmap](./homepage/README.md) — deployed baseline retained as the original architecture decision record; see the [Homepage runbook](../services/homepage/README.md) for the live implementation and remaining validation items.
- [Stremio + AIOStreams self-hosted streaming architecture roadmap](./stremio-aiostreams/README.md) — seven-phase baseline completed and retained as the original implementation decision record. See the [as-built architecture](../overview/stremio-streaming-architecture.md) and the phase completion records in the same roadmap directory.

## Planned Work

- [Taiwan–Russia Risk Watch Discord Bridge](./riskwatch-discord-bridge.md) — .NET 10 self-hosted publishing bridge from ChatGPT to Discord through Cloudflare Tunnel + Cloudflare Access Service Token, without router port forwarding.
- [Binary Artifact Upload PoC](./artifact-upload-poc.md) — single-container ASP.NET Core feasibility test for direct binary transfer from the ChatGPT sandbox to a self-hosted endpoint, validated by file size and SHA-256.
- [AIOStreams multi-source P2P expansion](./stremio-aiostreams/multi-source-p2p-expansion.md) — follow-up source-layer design; preserves Torrentio and evaluates STorz with local Bitmagnet + DMM coverage.
- [Usenet architecture roadmap](./usenet/README.md)
- [ShadowBroker OpenClaw integration roadmap](./shadowbroker-openclaw-integration.md)
