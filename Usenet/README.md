# Usenet Homelab Architecture
## Locked Decision Record and AI Implementation Brief

**Status:** Approved concept; implementation details still to be supplied  
**Version:** 1.0  
**Date:** 2026-06-25

## 1. Purpose

This document preserves the architecture decisions already agreed for adding Usenet discovery and downloading to an existing homelab. A future administrator or AI implementation agent must treat the items marked **LOCKED** as requirements, not as suggestions to redesign.

The target experience is a centralized web page for searching, browsing, filtering, and selecting indexed Usenet releases, with two distinct download paths:

1. a shared central downloader that writes completed content directly to selected application-consumption folders; and
2. a private downloader per user that writes completed content to that user's private staging area.

## 2. Scope boundary

### In scope

- NZB-based Usenet search and download workflow.
- NZBHydra2 as the centralized search and browsing interface.
- NZBGeek as the initial NZB indexer.
- Eweka as the Usenet provider.
- SABnzbd as the downloader for the central and per-user paths.
- Routing completed downloads through SABnzbd categories and mounted destinations.
- Isolation of personal queues, histories, configurations, and staging areas.

### Out of scope

The existing homelab, LAN, storage, Samba, drives, application folders, DNS, reverse proxy, authentication platform, backups, and consumer applications already exist and work. Do not redesign them unless explicitly requested during implementation.

Also out of scope unless later requested: Sonarr, Radarr, Prowlarr, Jellyfin, torrent integration, automated media acquisition, and a replacement for Plex or any other existing application.

## 3. Mental model

| BitTorrent concept | Usenet equivalent in this design |
|---|---|
| Search/index UI such as Bitmagnet | NZBHydra2 |
| DHT-derived/indexed metadata source | NZBGeek or another NZB indexer |
| Magnet link or `.torrent` file | `.nzb` file |
| qBittorrent-like downloader | SABnzbd |
| Peer swarm | Eweka's Usenet servers |

The analogy is conceptual. NZBHydra2 is a metasearch frontend; it does not build the NZBGeek index. NZBGeek supplies indexed release metadata and NZB download instructions. SABnzbd consumes NZBs and retrieves article parts from Eweka.

## 4. Locked high-level architecture

```text
                              SHARED HOMELAB SERVICES

  +---------------------+       +--------------------+
  | NZBHydra2           | ----> | NZBGeek           |
  | search / browse /   |       | initial indexer    |
  | filter / select     |       +--------------------+
  +----------+----------+
             |
             | two user-selectable download paths
             |
       +-----+---------------------------------------------+
       |                                                   |
       v                                                   v
  +---------------------+                           +---------------------+
  | Central SABnzbd     |                           | Per-user SABnzbd   |
  | shared downloader   |                           | one instance/user  |
  +----------+----------+                           +----------+----------+
             |                                                 |
             | Eweka                                          | Eweka
             v                                                 v
  application destination folders                  private staging/<user>
  on one or more existing drives                   on existing storage
```

### LOCKED decisions

1. NZBHydra2 is the single centralized user-facing space for Usenet search, browsing, filtering, and result selection.
2. NZBGeek is the initial indexer connected to NZBHydra2. Additional NZB indexers may be added later without changing the architecture.
3. Eweka is the Usenet provider used by SABnzbd.
4. End-user PCs and laptops are browser clients. No local Usenet downloader is required on those devices.
5. There is one shared central SABnzbd instance.
6. There is one private SABnzbd instance per user who needs a personal queue and staging area.
7. The central SABnzbd path has no completed staging layer. After its required temporary processing, completed content is placed directly in the selected application destination folder.
8. Each private SABnzbd instance places completed content only in that user's private staging area.
9. Each per-user SABnzbd instance has a private queue, history, configuration, and staging mount.
10. Search is centralized; download state is separated between the shared central path and private per-user paths.
11. Destination folders may reside on different drives. The implementation must support multiple mounts and category-to-destination mappings.
12. Existing application folders may be actively used by other apps. The architecture does not redesign those apps or storage layouts.

## 5. Components and responsibilities

| Component | Deployment | Responsibility | Must not become |
|---|---|---|---|
| NZBHydra2 | One shared homelab service | Centralized search, browse, filter, deduplicate, and result actions | A downloader or long-term file store |
| NZBGeek | External indexer service | Supply indexed release metadata and NZB files through its supported interface/API | The Usenet provider |
| Eweka | External Usenet provider | Serve the Usenet article data requested by SABnzbd | The search UI or indexer |
| Central SABnzbd | One shared homelab service | Download selected jobs and route completed output directly to category-specific application folders | A per-user private staging service |
| Per-user SABnzbd | One homelab instance per user | Maintain that user's private queue/history and download into that user's staging area | A shared queue or direct writer to application folders by default |
| End-user browser | Each PC/laptop | Access NZBHydra2 and the user's SABnzbd web UI | A required local downloader |

## 6. User experience

### 6.1 Centralized discovery

The user opens NZBHydra2 in a browser and performs all normal discovery there:

- keyword search;
- category selection;
- age and size filters;
- sorting and result comparison;
- viewing indexer-provided metadata; and
- selecting an NZB result.

NZBHydra2 is not a conventional filesystem browser and Eweka is not exposed as a directory tree. The UI presents indexed Usenet releases returned by NZBGeek and any future configured NZB indexers.

### 6.2 Download path A: central/shared download

1. The user finds a result in NZBHydra2.
2. The user sends the result to the central SABnzbd instance.
3. A SABnzbd category identifies the intended final destination.
4. Central SABnzbd obtains the data from Eweka, performs its normal verification/repair/unpack workflow, and writes the completed result directly to the category's application destination.
5. There is no separate completed staging directory for this path.

Example logical mappings:

```text
central category     final destination mount
-----------------    -------------------------------
app-a                /destinations/app-a
app-b                /destinations/app-b
shared-library       /destinations/shared-library
archive              /destinations/archive
```

Important nuance: SABnzbd still requires an internal temporary/incomplete working directory. "No staging" means there is no additional completed-review area between SABnzbd and the final application folder.

### 6.3 Download path B: personal/private download

1. The user finds a result in NZBHydra2.
2. The user obtains the `.nzb` from the centralized interface and submits it to the user's own SABnzbd web UI. A later implementation may streamline this handoff, but the architectural boundary remains the same.
3. The user's SABnzbd instance downloads through Eweka.
4. The completed result is written to the user's private staging area.
5. What the user does with staged content afterward is outside this Usenet architecture.

Example isolation:

```text
sab-alice  -> /staging/alice
sab-bob    -> /staging/bob
sab-carol  -> /staging/carol
```

## 7. Storage and processing behavior

### 7.1 Central downloader

The central instance must have:

- one internal temporary/incomplete working location;
- one mount for every permitted final destination, even when destinations are on different drives; and
- SABnzbd categories that map to those final destinations.

The central instance must not use a user staging area. Its completed folder/category targets are the approved application destination folders.

### 7.2 Per-user downloaders

Each user's instance must have:

- its own persistent SABnzbd configuration;
- its own queue/history state;
- its own incomplete working location or isolated subdirectory; and
- write access only to that user's private staging area unless additional access is explicitly approved later.

A user's staging path may be placed on any existing drive. The actual host paths and mount mechanism are implementation inputs, not architecture changes.

### 7.3 Isolation rule

A per-user SABnzbd instance must not share its configuration directory, queue state, history database, or completed staging directory with another user.

## 8. NZBHydra2-to-downloader routing

The preferred initial routing is deliberately simple:

```text
NZBHydra2 "send to downloader" action
    -> central SABnzbd

NZBHydra2 "download NZB" action
    -> user submits NZB to personal SABnzbd
```

This avoids forcing the shared NZBHydra2 instance to dynamically identify and route to every user's private downloader. A future enhancement may add user-specific downloader shortcuts, browser integration, or authenticated routing, but such an enhancement must preserve queue and staging isolation.

## 9. Credential boundaries

- NZBHydra2 holds the indexer configuration and NZBGeek API credential.
- SABnzbd instances hold the Usenet provider configuration needed to connect to Eweka.
- End-user browsers should not need direct NZBGeek credentials for normal use.
- Whether all SABnzbd instances share one Eweka account or use separate provider credentials is **not yet decided**. The implementation agent must verify the provider's current account, connection, and concurrency rules before choosing.
- API keys used between NZBHydra2 and the central SABnzbd must be treated as secrets.

## 10. Explicit non-goals and rejected alternatives

The implementation agent must not reintroduce these as default recommendations:

- Do not replace NZBHydra2 with Prowlarr for the user-facing search experience.
- Do not add Sonarr or Radarr automation unless explicitly requested later.
- Do not add Jellyfin; the existing consumer applications are outside this design.
- Do not use one shared SABnzbd queue for all personal downloads.
- Do not install SABnzbd on every laptop or PC as a requirement.
- Do not send personal downloads directly into application-consumption folders by default.
- Do not force central downloads through a user review/staging area.
- Do not redesign Samba, storage pools, drive layouts, or existing app libraries.
- Do not treat Usenet as a browsable remote filesystem.

## 11. Decisions intentionally left for implementation

These values are required before generating deployment files, but choosing them does not change the architecture:

- host operating system and container/runtime platform;
- container image source and version pinning policy;
- actual service names, ports, DNS names, and URLs;
- list of users requiring private SABnzbd instances;
- authentication mechanism for NZBHydra2 and each SABnzbd UI;
- exact host paths and container mount paths;
- central category names and their destination mappings;
- private staging path for each user;
- incomplete/temporary storage placement;
- resource limits and download speed/connection limits;
- whether Eweka credentials are shared or separate;
- backup/restore method for service configuration; and
- exact handoff convenience for personal NZBs (manual upload, watched folder, browser helper, or another approved method).

## 12. Implementation guardrails for an AI agent

When asked to implement this design, the AI agent must:

1. Start from this document and preserve every LOCKED decision.
2. Ask only for missing implementation values listed in Section 11.
3. Verify current official documentation and current stable versions before producing commands or manifests.
4. Produce a deployment plan before producing final configuration.
5. Use placeholders for unknown paths, users, credentials, domain names, and ports; never invent them.
6. Keep NZBHydra2 shared and keep per-user SABnzbd state isolated.
7. Configure the integrated NZBHydra2 downloader action for the central SABnzbd path.
8. Configure central SABnzbd categories to map to approved final application destinations across one or more drives.
9. Configure each personal SABnzbd instance to write completed jobs only to its user's private staging area.
10. Keep the central temporary/incomplete area distinct from its final destination mounts.
11. Do not add unrelated homelab components or media automation.
12. Explain any software limitation that prevents an exact behavior before proposing a workaround.

## 13. Acceptance criteria

The implementation is accepted only when all applicable checks pass:

- [ ] Users can open one shared NZBHydra2 web interface.
- [ ] NZBHydra2 can search NZBGeek and display searchable/filterable results.
- [ ] Users can obtain an NZB from the result page.
- [ ] A result can be sent to the central SABnzbd instance.
- [ ] Central jobs can be assigned to approved destination categories.
- [ ] Central SABnzbd downloads through Eweka and places completed output in the selected final destination without a completed staging layer.
- [ ] Central destinations can reside on different drives/mounts.
- [ ] Each user has a separate SABnzbd web UI, queue, history, and configuration.
- [ ] A user's personal job completes only in that user's private staging area.
- [ ] One user cannot see or control another user's personal SABnzbd queue through the normal UI.
- [ ] End-user PCs and laptops require only a browser for the self-hosted workflow.
- [ ] The implementation does not install or introduce Sonarr, Radarr, Prowlarr, Jellyfin, or other unrelated components.

## 14. Minimal validation scenarios

### Scenario A: central destination on drive 1

Search in NZBHydra2, send a test NZB to central SABnzbd, choose category `app-a`, and verify the completed output appears only in the mapped `app-a` destination.

### Scenario B: central destination on another drive

Repeat with a category mapped to a destination on a different drive. Verify correct routing and permissions.

### Scenario C: personal staging isolation

Submit a test NZB to User A's SABnzbd. Verify it appears in User A's queue/history and staging area, and not in User B's UI or staging area.

### Scenario D: shared search, separate download state

Two users search in the same NZBHydra2 service while using separate SABnzbd instances. Verify centralized discovery works without merging their personal queues.

## 15. Reference behavior verified from official project documentation

- NZBHydra2 is a metasearch tool for Newznab-compatible indexers and can send results to SABnzbd.
- SABnzbd consumes NZB files, provides a web interface, supports adding NZBs by upload/URL, and supports categories.
- SABnzbd uses a temporary download folder and a completed download folder; category and path configuration are implementation mechanisms for routing output.

Official references:

- NZBHydra2 project README: https://github.com/theotherp/nzbhydra2
- NZBHydra2 indexer tutorial: https://github.com/theotherp/nzbhydra2/wiki/Tutorial-%28Indexers%2C-newznab%2C-API%2C-%2Aarr%2C-etc.%29
- SABnzbd FAQ: https://sabnzbd.org/wiki/faq
- SABnzbd usage guide: https://sabnzbd.org/wiki/introduction/usage
- SABnzbd folder setup: https://sabnzbd.org/wiki/advanced/directory-setup
- SABnzbd API reference: https://sabnzbd.org/wiki/advanced/api

## 16. One-paragraph implementation prompt

Implement the locked Usenet homelab architecture in this document. Deploy one shared NZBHydra2 connected initially to NZBGeek; one shared central SABnzbd connected to Eweka and configured with categories that write completed jobs directly to approved application destination folders on potentially different drives; and one isolated SABnzbd instance per user, each with private configuration, queue/history, incomplete workspace, and completed staging directory. End-user devices are browser clients only. The shared NZBHydra2 interface is the centralized place for search, browsing, filtering, and result selection. Its integrated downloader action targets central SABnzbd, while personal downloads are handed off as NZB files to the relevant user's SABnzbd. Do not add Sonarr, Radarr, Prowlarr, Jellyfin, local client requirements, or redesign existing storage/network/application services. Ask for the missing implementation values listed in Section 11 before producing final deployment files.

