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
## Tried and removed

- **SSO for the media stack (LLDAP + Authelia + forward auth)** — ran about a month, removed.
  One password instead of five, and one place to revoke access, are real benefits; they are worth
  it with several users and a public entry point, and not worth it for one person on one subnet.
  What it actually cost:
  - **It could not remove the second prompt where it mattered.** qBittorrent must keep its own
    login, so a gated qBittorrent asked twice.
  - **It made name resolution load-bearing.** Authelia issues only `Secure` cookies, so protected
    apps had to be HTTPS under `*.media.home` — names that exist only on our AdGuard. The gate's
    own logic then required unpublishing the host ports, which deleted the `IP:port` fallback. A
    Windows machine silently using the Livebox for DNS could reach none of them.
  - **A local CA to install on every device**, plus a wildcard certificate to renew by hand.
  - Removed: the `identity` role, `playbooks/identity.yml`, `certs/`, the `*.media.home` NPM
    hosts and wildcard, the forward-auth nginx snippet, and seven vault secrets. Jellyfin went
    back to local accounts; Sonarr/Radarr/Prowlarr went from `External` to Forms with
    *Disabled for Local Addresses*.
  - **Precondition if it ever returns:** DHCP-distributed DNS, so `.home` resolves everywhere
    without touching each device. That is the OPNsense phase above — not something to retrofit
    onto the Livebox.

## Done

- **Webapps stack** (the `webapps` role) — the home for self-hosted apps that aren't media.
  Own compose project, published host ports, each app on its own login — **deliberately outside
  SSO**, same reasoning as administration/monitoring/glance. **Mealie** (recipes/meal planning)
  is what's left of it.
  - **Linkding and MkDocs were removed as unused** — containers, appdata, NPM proxy hosts and the
    `docs/` sources at the repo root all went. The stack keeps its plural name as the landing
    place for the next non-media app. (Grimoire had been rejected in favour of Linkding at the
    time, over a frozen published image; moot now.)
  - **Reached by `IP:port`, like everything else.** A `mealie.home` proxy host in NPM would cost
    one form if you ever want the name; the app is already on the `homelab` network for it.

- **Every app on a published host port, each with its own login.** Sonarr, Radarr, Prowlarr,
  Bazarr, Maintainerr, qBittorrent, Jellyfin and Seerr all bind their host port; Glance links to
  them by `IP:port`. Sonarr/Radarr/Prowlarr use Forms auth with **Authentication Required:
  Disabled for Local Addresses** — a password exists (`vault_arr_password`) but is never asked
  for on the LAN. ⚠️ Never set them back to `External`: that trusts anything that reaches them,
  and it was only defensible while the ports were closed and NPM was the sole route in.

- **Shared external `homelab` docker network** — **done.** `npm`, `jellyfin`, `gluetun`, the
  five *arr apps, `mealie`, `uptime-kuma` and `glance` all resolve each other by container name;
  the subnet is pinned in `group_vars` because gluetun's firewall names it. Created by the
  `docker` role (it was the `identity` role's job until that role was removed).
  Uptime Kuma probes the containers it can't reach by host port (gluetun/qBittorrent) across it;
  Glance links every tile by `IP:port` and needs no `check-url` at all any more.

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

