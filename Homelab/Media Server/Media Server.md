# Media Server (Beelink)

My first self-hosted media server: **Jellyfin + *arr stack + VPN-protected downloads**, running
in Docker on a Beelink Mini S12 Pro (Intel N100), **entirely provisioned and managed by Ansible**.
After the one-time Debian install, the box is never touched by hand except to read logs.

> See also: [[Runbook]] (how to operate/deploy/troubleshoot) · [[Roadmap]] (what's deferred to OPNsense).

---

## Hardware

- **Beelink Mini S12 Pro** — Intel **N100** (has Quick Sync iGPU → hardware transcoding).
- **System SSD** — 512 GB (`/`, `/home`, swap, `/boot/efi`). by-id `ata-512GB_SSD_MQ37W57903857`.
- **Media drive** — 931 GB Seagate Expansion USB HDD, wiped to **ext4**, mounted at `/data`.
  by-id `usb-Seagate_Expansion_HDD_00000000NC18006G-0:0`.
- Sits behind an Orange **Livebox** (OPNsense firewall planned later).

## Architecture

```
Internet
   │
 Livebox (router, DHCP) ──LAN── Workstation (Ansible control node)
   │
 Beelink · Debian 13 · Docker
   │
   ├── compose project: "media-stack"  (/home/media-stack)
   │     ├── gluetun ───VPN(OpenVPN)──▶ CyberGhost (Amsterdam)
   │     │     └── qbittorrent   (network_mode: service:gluetun — ALL traffic via VPN)
   │     ├── prowlarr → sonarr / radarr → bazarr   (recyclarr syncs their quality profiles)
   │     ├── lazylibrarian → qBittorrent(VPN)   (audiobooks: Prowlarr indexers → hardlink import)
   │     ├── jellyfin (/dev/dri Quick Sync) · seerr · maintainerr
   │     └── npm   (Nginx Proxy Manager — reverse proxy :80/:443, admin :81)
   │
   ├── compose project: "audiobookshelf" (/home/stacks/audiobookshelf)
   │     └── audiobookshelf  (:13378 — reads /data/media/audiobooks; standalone, no VPN)
   │
   └── compose project: "dashboard"    (/home/glance)
         └── glance   (:8280 — status board + host stats)

Storage:  /data (ext4)  →  torrents/{movies,tv,audiobooks} + media/{movies,tv,audiobooks}  (same fs ⇒ hardlinks)
          /home/docker            = Docker data-root (images/volumes)
          /home/media-stack/appdata = per-service /config bind mounts
```

**Golden rule (why it all hangs together):** every container that touches media mounts the
**whole `/data`** at the same path, so Sonarr/Radarr import downloads by **hardlink** (instant,
no second copy, seeding continues). And **only qBittorrent** uses the VPN — everything else is
on the normal network.

## Services

| Service | Purpose | Port | Config (`appdata/…`) |
|---|---|---|---|
| **gluetun** | VPN gateway (CyberGhost) + kill-switch | — (publishes qBit's ports) | `gluetun/` (holds `client.crt`+`client.key`) |
| **qbittorrent** | Torrent client, forced through gluetun | 8080 | `qbittorrent/` |
| **prowlarr** | Indexer manager → feeds Sonarr/Radarr | 9696 | `prowlarr/` |
| **sonarr** | TV automation | 8989 | `sonarr/` |
| **radarr** | Movie automation | 7878 | `radarr/` |
| **bazarr** | Subtitles (French) | 6767 | `bazarr/` |
| **recyclarr** | Syncs TRaSH quality profiles + custom formats → Sonarr/Radarr (cron, VO/French VOSTFR) | — (CLI/cron) | `recyclarr/` |
| **lazylibrarian** | Audiobook/ebook automation (Sonarr-for-books) → Prowlarr + qBittorrent | 5299 | `lazylibrarian/` |
| **audiobookshelf** | Audiobook/podcast server — **own stack** (`/home/stacks/audiobookshelf`) | 13378 | `audiobookshelf/appdata/` (own stack) |
| **jellyfin** | Media server (Quick Sync HW transcode) | 8096 | `jellyfin/` |
| **seerr** | Request UI (formerly Jellyseerr) | 5055 | `seerr/` |
| **maintainerr** | Rule-based library cleanup | 6246 | `maintainerr/` |
| **npm** | Reverse proxy (name-based access) | 80/443/81 | `npm/` |
| **glance** | Dashboard (separate project) | 8280 | `/home/glance/config/glance.yml` |

Access today: `http://<beelink-ip>:<port>`. Clean `*.home` names via NPM are on hold until
OPNsense provides LAN DNS — see [[Roadmap]].

## Key decisions

- **VPN protocol = OpenVPN**, not WireGuard. gluetun supports CyberGhost *natively only for
  OpenVPN*; WireGuard would need custom mode. CyberGhost OpenVPN needs **username + password
  AND a client certificate + private key** (files in `appdata/gluetun/`).
- **Media filesystem = ext4** — hardlinks + Unix ownership are mandatory; exFAT/NTFS rejected.
- **Docker data-root = `/home/docker`** (414 GB) instead of the tiny 12 GB `/var`. Logs capped.
- **No auto-updates** — updates are a deliberate weekly `ansible-playbook playbooks/update.yml`.
- **No UFW** on the Docker host (Docker bypasses it; the LAN/OPNsense is the firewall).
- **Glance is its own compose project** so it can oversee multiple stacks, not just this one.
- **Cleanup tool = Maintainerr** (it supports Jellyfin natively).
- **Audiobooks: split acquisition from serving.** **LazyLibrarian** (not Readarr — retired 2026,
  dead metadata server) does the *arr-style grabbing and lives **in media-stack** because it's coupled
  (Prowlarr indexers + qBittorrent-via-VPN + `/data` hardlink import). **Audiobookshelf** is the
  *server* and is a **separate stack** — no pipeline coupling, independently restartable. The rule:
  co-locate what shares a private-network dependency or lifecycle; isolate standalone apps.
- **Grab VO, not dubs.** Recyclarr pushes TRaSH's **French VOSTFR** profiles (original audio +
  FR subs) at **1080p** — 4K/Remux would swamp the single 931 GB drive. Bazarr then fetches the
  French subtitles. Config-as-code (`recyclarr.yml.j2`); switching flavour (`multi-vo`) or
  resolution is a one-line change + re-sync. Recyclarr runs on the **normal network**, not the VPN.

## Build history

The full step-by-step build log, decisions, and disk map live in `PLAN.md` at the repo root
(`~/Documents/Homelab/PLAN.md`). The infrastructure-as-code is in `infra/` — see [[Runbook]].
