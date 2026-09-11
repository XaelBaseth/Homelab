# Media Server — Roadmap

What's deliberately deferred on the [[Media Server]], and why. Nothing here blocks using the
stack from home today; these are the "next phase" items — most tied to standing up **OPNsense**.

---

## Deferred to the OPNsense phase

- **DNS high availability.** AdGuard is the LAN's DHCP server and its only DNS, so a Beelink
  outage takes name resolution down for the house. The fix is a second filtering resolver handed
  out by DHCP — AdGuard on OPNsense, or a Raspberry Pi kept identical with `adguardhome-sync`. Both
  know `*.home`, which is what makes a secondary safe (the Livebox as secondary is what broke
  `.home` on Windows/Android). Until then: the rollback in the [[Runbook]] — Livebox DHCP back on.
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
  Bazarr, Maintainerr, qBittorrent, Jellyfin and Seerr all bind their host port — the route that
  always works, DNS or not. Sonarr/Radarr/Prowlarr use Forms auth with **Authentication Required:
  Disabled for Local Addresses** — a password exists (`vault_arr_password`) but is never asked
  for on the LAN. ⚠️ Never set them back to `External`: that trusts anything that reaches them,
  and it was only defensible while the ports were closed and NPM was the sole route in.

- **Shared external `homelab` docker network** — **done.** `npm`, `jellyfin`, `gluetun`, the
  five *arr apps, `mealie`, `uptime-kuma` and `glance` all resolve each other by container name;
  the subnet is pinned in `group_vars` because gluetun's firewall names it. Created by the
  `docker` role (it was the `identity` role's job until that role was removed).
  Uptime Kuma probes the containers it can't reach by host port (gluetun/qBittorrent) across it;
  Glance links every tile by its `*.home` name and probes `IP:port` through `check-url` (containers
  can't resolve `*.home`).

- **AdGuard Home** (the `adguard` role). The LAN's **DNS and DHCP server**: ad/tracker/malware
  blocking + `*.home` rewrites for every device in the house, in its own compose project. Host
  network (DHCP is broadcast and can't cross Docker's NAT); admin on `:3000` (NPM owns `:80`).
  Upstream is **Quad9 over DoH**. The Livebox can't distribute a DNS server, so its DHCP is off and
  AdGuard hands itself out. Setup/operate: see the AdGuard section of the [[Runbook]].
  - **Per-device DNS came first and was replaced.** Each device kept the Livebox as a secondary
    resolver (the failsafe); Windows and Android took its NXDOMAIN for `*.home`, so names only
    worked reliably on the workstation.
  - **Knock-ons:** the Beelink is on a static IP set on the host (`common` role — it can't lease
    from itself), its own `/etc/resolv.conf` points at the Livebox + Quad9 (never at AdGuard), and
    AdGuard is pinned and excluded from Watchtower (a bad auto-update would cut the whole house).
  - **NPM proxy hosts** — one per app, forwarding to `192.168.1.19:<published port>`, Websockets on.

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

