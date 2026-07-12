# Media Server — Roadmap

What's deliberately deferred on the [[Media Server]], and why. Nothing here blocks using the
stack from home today; these are the "next phase" items — most tied to standing up **OPNsense**.

---

## Deferred to the OPNsense phase

- **LAN-wide DNS + `*.home` names.** The Orange Livebox won't distribute a custom DNS server,
  so name-based access is on hold. OPNsense's **Unbound** will resolve `*.home` → Beelink IP
  (wildcard/host overrides), and then NPM's proxy hosts light up for every device.
  *Interim:* a `/etc/hosts` entry on the workstation, or just use `beelink-ip:port`.
- **Tailscale** for remote/5G access. Pairs with OPNsense (which can be the Tailscale subnet
  router). No port-forwarding, no certs needed — the tailnet is encrypted.
- **AdGuard Home** (optional) — could run then for network-wide ad-blocking + the DNS rewrites,
  or just use OPNsense Unbound directly. Was dropped now because without LAN-wide distribution
  it only helps manually-pointed devices.

## Cleanup / polish (any time)

- **Pin all image tags to digests.** Everything is on `latest` right now. One pass to lock the
  running versions for reproducibility. Capture with:
  ```bash
  ansible beelink -m command -a "docker inspect --format '{{.Config.Image}} {{.Image}}' <container>" -b
  ```
  then set the pinned `…@sha256:…` in `roles/media_stack/defaults/main.yml`.
- **Shared external `homelab` docker network** so cross-stack services (NPM ↔ Glance ↔ future
  stacks) resolve each other by container name again, instead of `beelink-ip:port`.
- **Commit `infra/` to git.** Safe to commit — `vault.yml` is encrypted and `.vault_pass` is
  gitignored. Would checkpoint the whole IaC.

## End-to-end test still to do

Once an **indexer** is added in Prowlarr and something is requested:
1. Request in **Jellyseerr** →
2. **Sonarr/Radarr** grab it →
3. **qBittorrent** downloads it **through the VPN** →
4. **hardlink import** into `/data/media` →
5. plays in **Jellyfin** with **hardware transcode** confirmed on the dashboard.

That's the final proof the whole pipeline flows end to end.
