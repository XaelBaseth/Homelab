# Media Server — Roadmap

What's deliberately deferred on the [[Media Server]], and why. Nothing here blocks using the
stack from home today; these are the "next phase" items — most tied to standing up **OPNsense**.

---

## Deferred to the OPNsense phase

- **LAN-*wide* DNS + `*.home` names.** The Orange Livebox won't distribute a custom DNS server,
  so *automatic* LAN-wide name resolution is still on hold. **AdGuard Home now solves this
  per-device today** — its `*.home → Beelink` rewrite + manual DNS on each device lights up NPM's
  proxy hosts (see AdGuard, under *Done*). Full LAN-wide (no per-device step) waits for OPNsense
  to become the **DHCP server** — then it hands out AdGuard/Unbound as DNS to everything at once.
  *Interim for un-pointed devices:* a `/etc/hosts` entry on the workstation, or `beelink-ip:port`.
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
## Done

- **SSO for the media stack** (the `identity` role). **LLDAP** (user directory) + **Authelia**
  (login portal + forward auth), deployed and idempotent. One account instead of one per app.
  Ansible does everything up to the GUI boundary: containers, the shared `homelab` network, the
  local CA + `*.media.home` wildcard cert, the `authelia` service account seeded straight from
  the vault, and the forward-auth endpoint templated into NPM's `custom/server_proxy.conf` (so
  the nginx half lives in git, not in NPM's database). **The click-through half is [[SSO Setup]]**
  — AdGuard rewrite, your LLDAP user, the NPM proxy hosts, CA install, Jellyfin's LDAP plugin.
  - Protected hosts move to `*.media.home`: the session cookie needs a shared two-label suffix,
    and browsers reject cookies on a single-label domain like `.home`. Old `*.home` names are
    untouched.
  - Jellyfin uses **LDAP**, never forward auth — TV/phone clients can't do a browser redirect.
  - Deliberately **not** covering administration/monitoring/adguard/glance, so Dockge + Dozzle
    stay reachable on plain ports if auth ever breaks.

- **Enforcement: media-stack host ports unpublished.** Sonarr, Radarr, Prowlarr, Bazarr,
  Maintainerr and qBittorrent's WebUI no longer bind host ports — NPM is the only route, so
  Authelia can't be walked around at `beelink-ip:port`. Sonarr/Radarr/Prowlarr are set to
  **Authentication Method: External** (they trust the proxy, no second login). ⚠️ Those two
  changes are a pair: re-publishing a port without reverting its auth method leaves that app
  wide open. Jellyfin `:8096` and Seerr `:5055` stay published on purpose — TV clients and the
  mobile app need them, and neither is behind forward auth.

- **Shared external `homelab` docker network** — **done.** `npm`, `authelia`, `lldap`,
  `jellyfin`, `gluetun`, the five *arr apps, `uptime-kuma` and `glance` all resolve each other by
  container name; the subnet is pinned in `group_vars` because gluetun's firewall names it.
  Uptime Kuma's five affected monitors were repointed at container names and it now also watches
  Authelia + LLDAP (18 monitors, all green). Glance uses `check-url` to probe internally while
  its tiles still link to the `*.media.home` addresses, and lists the portal + directory under
  *Infrastructure*.

- **AdGuard Home** (the `adguard` role). Network-wide DNS **ad/tracker/malware blocking** +
  `*.home` rewrites, in its own compose project. DNS binds `:53` on the Beelink's LAN IP (dodges
  the systemd-resolved stub — no host DNS change); admin on `:3000` (NPM owns `:80`). Upstream is
  **Quad9 over DoH** (encrypted, free malware layer). **Rollout is per-device for now** — the
  Livebox can't distribute DNS, so you point each device at the Beelink manually; this all carries
  over 1:1 to OPNsense (same plugin, or the same idea in Unbound). Setup/operate: see the AdGuard
  section of the [[Runbook]]. *Escalation if impatient:* let AdGuard be the LAN DHCP server
  (disable the Livebox's) for true whole-network coverage before OPNsense exists.

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
- **Commit `infra/` to git.** Safe to commit — `vault.yml` is encrypted and `.vault_pass` is
  gitignored. Would checkpoint the whole IaC.

