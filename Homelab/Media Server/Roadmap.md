# Media Server — Roadmap

What's deliberately deferred on the [[Media Server]], and why. Nothing here blocks using the
stack from home today; these are the "next phase" items — most tied to standing up **OPNsense**.

---

## Deferred to the OPNsense phase

- **LAN-wide DNS + `*.home` names.** The Orange Livebox won't distribute a custom DNS server,
  so name-based access is on hold. OPNsense's **Unbound** will resolve `*.home` → Beelink IP
  (wildcard/host overrides), and then NPM's proxy hosts light up for every device.
  *Interim:* a `/etc/hosts` entry on the workstation, or just use `beelink-ip:port`.
- **NPM proxy hosts to create** (in the NPM UI, `:81`) once DNS resolves — forward to the Beelink
  IP + published port (no shared docker network yet). ⚠️ **enable "Websockets Support"** on the
  monitoring ones — Uptime Kuma and Beszel are realtime over WebSocket and load blank without it:

  | Domain | → Forward | Websockets |
  |--------|-----------|------------|
  | `uptime.home` | `beelink:3001` | **yes** |
  | `beszel.home` | `beelink:8090` | **yes** |
  | `jellyfin.home` / `seerr.home` / `sonarr.home` / … | the media-stack ports | as needed |
- **Tailscale** for remote/5G access. Pairs with OPNsense (which can be the Tailscale subnet
  router). No port-forwarding, no certs needed — the tailnet is encrypted.
- **AdGuard Home** (optional) — could run then for network-wide ad-blocking + the DNS rewrites,
  or just use OPNsense Unbound directly. Was dropped now because without LAN-wide distribution
  it only helps manually-pointed devices.

## Done

- **Monitoring + alerting** (the `monitoring` role). Uptime Kuma (service/container up-down +
  alerting), Beszel (host + per-container resource history + threshold alerts), and an off-box
  **healthchecks.io heartbeat** cron so a full box/uplink death still pages. Alerts go out over a
  **Discord webhook** (external on purpose — the on-box tools can't alert on their own death).
  Deploy/operate: see the Monitoring section of the [[Runbook]].

## Cleanup / polish (any time)

- **Pin all image tags to digests.** Everything is on `latest` right now — *including the three
  new monitoring images* (`uptime-kuma`, `beszel`, `beszel-agent`). One pass to lock the
  running versions for reproducibility. Capture with:
  ```bash
  ansible beelink -m command -a "docker inspect --format '{{.Config.Image}} {{.Image}}' <container>" -b
  ```
  then set the pinned `…@sha256:…` in `roles/media_stack/defaults/main.yml`.
- **Shared external `homelab` docker network** so cross-stack services (NPM ↔ Glance ↔ future
  stacks) resolve each other by container name again, instead of `beelink-ip:port`.
- **Commit `infra/` to git.** Safe to commit — `vault.yml` is encrypted and `.vault_pass` is
  gitignored. Would checkpoint the whole IaC.

