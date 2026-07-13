# Beelink Media Server — a step-by-step build you write yourself

## Context

Repurposing the **Beelink Mini S12 Pro (Intel N100)** as a dedicated media server on your
LAN. Stack: Jellyfin + the *arr apps + qBittorrent behind a VPN + Maintainerr for cleanup,
all in Docker Compose, reachable by name through Nginx Proxy Manager, with AdGuard for local
DNS and Tailscale for remote (5G) access. Debian + SSH are installed manually; everything
after that you build.

**Goal: after the Debian install, you never SSH in except to debug.** All provisioning
(package updates, Docker install, base hardening, storage mount) and all stack deployment
run from your workstation via **Ansible**. The only hands-on-machine step is installing
Debian + enabling SSH (Phase 0).

**This is still a learning path, not a finished repo** — you *author* every playbook and
the `docker-compose.yml` yourself so you understand them. You just run them via Ansible
instead of typing commands over SSH: edit a file locally → re-run the role → verify. Build
the stack up **incrementally** (one service group at a time, verify, then add the next) so
any breakage is isolated to the piece you just added. Logs are read in **Dozzle** and stacks
are driven from **Dockge** in the browser; SSH is a last resort when the tooling itself is down.

### Decisions locked in
- **Cleanup:** **Maintainerr** — supports Jellyfin natively (uses Streamystats for watch
  data + Jellyseerr/Sonarr/Radarr).
- **VPN:** **gluetun + WireGuard** using a CyberGhost WireGuard config. gluetun *runs* your
  CyberGhost config and adds the kill-switch; only qBittorrent routes through it.
- **Reverse proxy:** **Nginx Proxy Manager** (GUI, easiest to learn/debug).
- **Local DNS:** **AdGuard Home** on the Beelink resolves `*.home` → Beelink IP.
- **Remote access:** **Tailscale** (no port-forwarding, no certs). Optional split-DNS so the
  same `*.home` names work on LAN and over 5G.
- **Certs:** none for now — plain HTTP. (Traffic off-LAN rides Tailscale's encryption.)
- **IaC:** an `infra/` directory in this Homelab vault repo.

---

## Target architecture

### Services (one `docker-compose.yml`)
| Service | Network | Key detail |
|---|---|---|
| **gluetun** | `media`; publishes qBit port | VPN gateway (CyberGhost/WireGuard) + kill-switch |
| **qbittorrent** | `network_mode: service:gluetun` | **All** traffic via gluetun; no direct net |
| **prowlarr** | `media` | Indexer manager → feeds Sonarr/Radarr |
| **sonarr / radarr** | `media` | TV / movies; import from qBit by hardlink |
| **bazarr** | `media` | Subtitles (optional) |
| **jellyfin** | `media` + `/dev/dri` | Quick Sync HW transcode; **not** via VPN |
| **jellyseerr** | `media` | Request UI |
| **maintainerr** | `media` | Rule-based library cleanup / disk-space |
| **nginx-proxy-manager** | `proxy` + `media` | Ports 80/81; name-based routing |
| **glance** | `proxy` | Dashboard linking to each hostname |
| **adguardhome** | host / `proxy` | Local DNS for `*.home` (+ ad-blocking) |
| **tailscale** | host | Remote access; optionally advertises subnet/DNS |

Two Docker networks: **`media`** (app-to-app) and **`proxy`** (NPM ↔ web UIs).
**Only qBittorrent uses the VPN.** Jellyfin/the *arr apps stay on the normal network.

### Storage (the hardlink rule)
Separate media drive mounted at **`/data`** (one filesystem), same mount into every app so
Sonarr/Radarr import by **hardlink + atomic move** — instant, no double disk use, seeding
continues.
```
/data
├── torrents/{movies,tv}   # qBittorrent download dirs
└── media/{movies,tv}      # Jellyfin libraries + Sonarr/Radarr targets
```
Same `PUID`/`PGID` + `UMASK=002` on every container. App `/config` volumes live on the
system disk.

### How you'll actually reach things
1. **DNS:** AdGuard answers `jellyfin.home` → Beelink IP. Set your router's DHCP to hand out
   AdGuard as the DNS server (so every device uses it).
2. **Proxy:** that request hits NPM on `:80`; NPM routes by hostname to the right container.
3. **Remote (5G):** Tailscale on Beelink + phone. Reach it over the tailnet with no ports
   open. Enhancement: Tailscale **split-DNS** → point the `home` domain at AdGuard so the
   *same* `jellyfin.home` URL works at home and away.

---

## Phase 0 — Manual bootstrap ✅ DONE

Debian installed, SSH enabled, a **sudo user with your SSH public key** configured, key-based
login working. Ansible's prerequisites (SSH + sudo user + `python3`) are satisfied. From here
on, work happens from the workstation; SSH is debug-only.

*(Optional later: a Debian preseed/autoinstall file would automate even this — out of scope
for now, but the repo layout leaves room for it.)*

---

## Phase 1 — Ansible provisioning (no more manual setup)

From your workstation you author and run the provisioning roles. After this the machine is
fully prepared without a single hand-typed command on it.

**Repo layout** (author incrementally, not all at once):
```
infra/
├── ansible.cfg                 # become per-play (not global); vault_password_file = .vault_pass
├── requirements.yml            # community.docker, community.general, ansible.posix
├── .vault_pass                 # GITIGNORED — master key that decrypts vault.yml
├── .gitignore                  # ignores .vault_pass, *.retry
├── inventory/
│   ├── hosts.yml               # beelink; ansible_host + become pw come from the vault
│   └── group_vars/all/
│       ├── vars.yml            # (later) PUID/PGID, tz, paths, pinned image tags, hostnames
│       └── vault.yml           # ENCRYPTED: beelink IP, become pw, later CyberGhost WG key + API keys
├── playbooks/site.yml          # runs common → storage → docker → media_stack (become: true per play)
└── roles/{common,storage,docker,media_stack}/
```
**Secrets model (established in Step 1.0):** `group_vars` sits under `inventory/` so ad-hoc
commands load it too. Real secrets live in the encrypted `vault.yml` (first entries:
`vault_beelink_host`, `vault_become_password`); the inventory references them via
`{{ vault_* }}`. `.vault_pass` on the workstation makes every run unattended (no `-K`, no
vault prompt) and must never be committed.
We build this **one step at a time**, stopping to verify (and commit) after each before
starting the next:

- **Step 1.0 — Scaffold + connectivity.** ⬅️ *current* — Create `infra/` with `ansible.cfg`,
  `requirements.yml`, `inventory/hosts.yml` (Beelink IP + sudo user), and empty
  `group_vars/all/vars.yml`. Install collections (`ansible-galaxy install -r requirements.yml`).
  *Verify:* `ansible beelink -m ping` returns `pong`.
- **Step 1.1 — `common` role.** apt update/upgrade, base packages, timezone/locale,
  `unattended-upgrades`, SSH hardening (disable password + root login), optional UFW.
  *Verify:* role applies clean; a **re-run reports 0 changed**; you can still log in with your key.
- **Step 1.2 — `storage` role.** `fstab` entry for the media disk **by UUID**, mount at
  `/data`, create `torrents/{movies,tv}` + `media/{movies,tv}` owned by the app user.
  *Verify:* `df -h` shows `/data`; a test file writes there.
- **Step 1.3 — `docker` role.** Docker Engine + `docker compose` plugin from Docker's apt
  repo, enable the service, add the user to `docker`. *Verify:* `docker run hello-world`.

After Step 1.3 the Beelink is provisioned entirely by Ansible — nothing installed by hand
over SSH — and we move to Phase 2.

> **Working rhythm for every step:** you author the files (I explain each so you understand
> it), run the role, verify, then we commit before the next step. SSH only to read logs if a
> step fails.

---

## Phase 2 — Ansible-deployed media stack (you author the compose, build it incrementally)

You write `docker-compose.yml.j2` + `env.j2` yourself (so you understand every service), and
the **media_stack** role deploys them with `community.docker.docker_compose_v2`
(`state: present`). Your loop is: **edit the template locally → re-run the role → verify →
add the next service group.** Add services in this order, verifying each before moving on:

1. **VPN + qBittorrent only** (`qbittorrent` uses `network_mode: service:gluetun`; CyberGhost
   WireGuard key/endpoint in the encrypted vault). *Verify the kill-switch — most important:*
   - `docker exec qbittorrent wget -qO- ifconfig.me` → the **VPN** IP, not your ISP's.
   - `docker stop gluetun` → qBittorrent loses all net (no leak). Restart, recheck.
2. **Prowlarr → Sonarr/Radarr → Bazarr.** Wire indexers to Sonarr/Radarr; add qBittorrent as
   the download client (host = `gluetun`); keep all paths under `/data`. *Verify:* a test grab
   downloads via the VPN and Sonarr/Radarr **hardlink-imports** it — `ls -i` shows the file in
   `torrents/` and `media/` sharing one inode (no copy).
3. **Jellyfin + Jellyseerr.** Jellyfin with `/dev/dri` for Quick Sync, libraries on
   `/data/media`; Jellyseerr connected to Jellyfin + Sonarr/Radarr. *Verify:* a forced
   transcode shows **hardware** (QSV) in the Jellyfin dashboard.
4. **Maintainerr.** Connect to Jellyfin + Jellyseerr + Sonarr/Radarr (+ Streamystats for
   watch-based rules). Preview one rule before enabling deletes.
5. **NPM + AdGuard.** AdGuard rewrite `*.home` → Beelink IP; router DHCP hands out AdGuard as
   DNS. NPM Proxy Host per service (`jellyfin.home` → `jellyfin:8096`, …). *Verify:* names
   resolve and load over HTTP from a LAN device.
6. **Glance** dashboard linking each `*.home` hostname.
7. **Tailscale** on the Beelink + your phone. *Verify:* reach a service over 5G with Wi-Fi
   off. Optional: Tailscale split-DNS so `*.home` works remotely too.

**Cross-cutting for the whole stack:**
- **Secrets:** `ansible-vault` encrypts `vault.yml` (CyberGhost WireGuard key/endpoint, app
  API keys). Keep the vault password out of git.
- **Pin image tags** (not `latest`) so re-runs are reproducible.
- *Verify Ansible overall:* `ansible-playbook playbooks/site.yml --check --diff`, then a real
  run; a **second** run must report **0 changed** (idempotent).

---

## Verification summary
- **Kill-switch:** qBit's public IP = VPN IP; stopping gluetun kills qBit's net (no leak).
- **Hardlinks:** same inode in `torrents/` and `media/` after import (no duplication).
- **HW transcode:** `/dev/dri` in container; Jellyfin dashboard shows QSV on a transcode.
- **End-to-end:** request in Jellyseerr → qBit downloads via VPN → *arr hardlink-imports →
  appears in Jellyfin; Maintainerr rule previews correctly.
- **Access:** `*.home` resolves via AdGuard + loads through NPM on LAN; Tailscale reaches it
  over 5G; Ansible re-run is idempotent.

## Notes / risks
- **CyberGhost WireGuard config** is generated from their dashboard — grab the key/endpoint
  early; that's the one unknown to validate (Phase 2, step 1).
- **Don't port-forward** anything to WAN. Remote access is Tailscale-only.
- **N100 RAM** (~16 GB) is fine here; HW transcode keeps CPU low. Avoid many simultaneous
  software transcodes.
- **Future OPNsense** cleanly takes over DNS (from AdGuard) and can host a Tailscale subnet
  router later — nothing here blocks that migration.
- **AdGuard is now your DNS for the whole LAN** — if the Beelink is down, LAN DNS is down.
  Keep a secondary DNS (e.g. `1.1.1.1`) in DHCP as a fallback.

---

## Hardware reference (Beelink disks)
- **System SSD** — `sda`, 512 GB, by-id `ata-512GB_SSD_MQ37W57903857`. Holds `/`, `/home`,
  swap, `/boot/efi`, `/var`, `/tmp`. **Never targeted by automation.**
- **Media drive** — `sdb`, 931 GB Seagate Expansion USB HDD, by-id
  `usb-Seagate_Expansion_HDD_00000000NC18006G-0:0`. Wiped from factory NTFS → ext4, mounted
  at `/data`. This by-id path is what the `storage` role pins to.

## Decisions log (beyond original plan)
- **No unattended-upgrades / no auto-apt-upgrade.** User prefers controlled weekly updates via
  `ansible-playbook playbooks/update.yml` (run on a phone reminder). `common` never upgrades.
- **No avahi/mDNS.** Reach the box by vaulted IP.
- **No UFW** on the Docker host (Docker bypasses it; rely on LAN + future OPNsense).
- **Media filesystem = ext4** (hardlinks + Unix ownership required; exFAT/NTFS rejected).
- **Docker data-root = `/home/docker`** (414G partition), not the ~12G `/var`. Log rotation
  capped at 3×10 MB/container via `daemon.json`.
- **Storage format guard:** `filesystem` uses `force` only when the disk isn't already ext4
  (`force: "{{ (blkid TYPE) != ext4 }}"`) — never reformats an existing ext4 drive.
- **Stacks consolidated under `/home/stacks/<name>`** (shared `stacks_root` var) so **Dockge**
  can manage every compose project from one root. Dir *basenames* are unchanged, so the
  Compose project name stays the same and running containers are re-adopted (no rebuild).
  **appdata stays at the old `/home/<name>/appdata` paths** (absolute, referenced via `.env`) —
  live data is never moved. One-time cleanup tasks drop the stale compose files at the old paths.
- **`administration` stack ≠ `monitoring` stack.** Split by *job*: monitoring only **observes**
  (Uptime Kuma, Beszel, healthchecks heartbeat — read-only sockets); administration **acts on**
  containers (Dockge, Dozzle, Watchtower). Different blast radius, so different stack/role.
- **Management = Dockge (not Portainer).** Compose-native, points at the stacks it already owns,
  low risk of a second source of truth fighting Ansible. Needs `docker.sock` **rw** and the
  stacks dir bind-mounted at the **same path** in/out (`DOCKGE_STACKS_DIR={{ stacks_root }}`).
- **Log aggregation = Dozzle (not Loki/Grafana).** Single-host, zero-config, `docker.sock` **ro**,
  live tail across all containers. Loki+Grafana deferred — only worth it for long-term retention.
- **Watchtower = weekly auto-updater (`WATCHTOWER_SCHEDULE`, Saturday 08:00).** Pulls + recreates
  any outdated container on the host in one batch, cleans up old images (`WATCHTOWER_CLEANUP`), then
  Discord-pings a summary of what changed (shoutrrr `vault_watchtower_notification_url`, optional).
  Deliberately supersedes the earlier notify-only stance and the digest-pinning TODO: on a single
  host the convenience of hands-off weekly updates outweighs Ansible owning every image version.
  Complements Glance's GitHub releases widget.

## Progress log
- **Phase 0** ✅ done — Debian + SSH + sudo user with key.
- **Phase 1** ▶️ in progress
  - **Step 1.0** ✅ done — `infra/` scaffolded; ansible-vault (`.vault_pass`) set up; IP +
    become password vaulted; `ansible beelink -m ping -b` returns `pong` unattended.
  - **Step 1.1** ✅ done — `common` role: apt base packages, locale, timezone, SSH hardening
    (password + root login disabled, key-only) with auto-rollback safety. Idempotent
    (`changed=0`). Separate `playbooks/update.yml` for manual weekly upgrades.
  - **Step 1.2** ✅ done — `storage` role: Seagate wiped to ext4, mounted `/data` by UUID
    (`nofail`), `torrents/{movies,tv}` + `media/{movies,tv}` owned by `beelink`. Idempotent.
  - **Step 1.3** ✅ done — `docker` role: Docker Engine + compose plugin from official repo,
    data-root `/home/docker`, log rotation, `beelink` in `docker` group. `hello-world` runs.
- **Phase 1 COMPLETE** ✅ — box fully provisioned by Ansible; SSH now debug-only.
- **Phase 2** ▶️ in progress
  - **2.1** ✅ done — `media_stack` role deploys gluetun + qBittorrent via
    `community.docker.docker_compose_v2`. CyberGhost **OpenVPN** (native provider): needs
    `OPENVPN_USER`/`PASSWORD` **plus** `client.crt` + `client.key` files in `appdata/gluetun/`
    (client cert + PRIVATE key — not the CA; secrets in vault as block scalars, rendered to
    files). Kill-switch verified: VPN IP (NL) ≠ ISP IP; qBittorrent 000 when gluetun stopped.
    Handler `Recreate VPN containers` restarts gluetun+qbit when secrets change.
  - **2.2** ✅ done — Prowlarr/Sonarr/Radarr/Bazarr added (normal network). Sonarr/Radarr →
    qBittorrent at `gluetun:8080` (disable qBit host-header validation), root folders
    `/data/media/{tv,movies}`, hardlinks on. Prowlarr Apps → Sonarr/Radarr; Bazarr → both.
    Hardlink proof: same inode across `/data/torrents` and `/data/media` inside sonarr.
  - **2.3** ✅ done — Jellyfin (`/dev/dri` + group_add render=992/video=44, QSV verified in
    container) + Jellyseerr. Full transcode-on-dashboard check pending real media.
  - **2.4** ✅ done — Maintainerr (`:6246`, Jellyfin-native) for disk-space rules.
  - **2.5** ✅ done (partial) — **NPM** deployed, proxy hosts for all services (`*.home` →
    container:port; qBittorrent → `gluetun:8080`). **AdGuard deferred to OPNsense** — Livebox
    won't distribute a custom DNS server, and OPNsense's Unbound will own LAN DNS + `*.home`
    rewrites later. Interim: `/etc/hosts` entry on the workstation for name-based access.
  - **2.6** ✅ done — **Glance** in its *own* compose project + `glance` role + `dashboard.yml`
    playbook (deliberately separate from `media-stack` so it can oversee multiple stacks).
    Health checks hit `beelink-ip:port` (cross-project can't use container names); links use
    `*.home`. Reachable at `beelink-ip:8280`.
  - **2.7 Tailscale — DEFERRED to OPNsense phase.** Using the stack from home (LAN) for now.
  - **2.8** ✅ done — **`monitoring` role**: Uptime Kuma (`:3001`, up/down + alerts),
    Beszel hub + agent (`:8090`, host + per-container metrics), healthchecks.io dead-man's
    switch (root cron pings out every 5 min). Own compose project + `monitoring.yml` playbook.
  - **2.9** ✅ done — **new `administration` role/stack** (separate from `monitoring` — observe vs
    act): **Dozzle** (`:8888`, live log aggregation, `docker.sock` ro), **Dockge** (`:5001`,
    compose-stack management, `docker.sock` rw), **Watchtower** (headless, weekly auto-updater —
    Saturday 08:00). Own compose project + `administration.yml` playbook; added to `site.yml`. Prerequisite:
    **all stacks consolidated under `/home/stacks/<name>`** (see decisions log) so Dockge sees them
    from one root. Glance gained an **Administration** group (bookmarks + monitor tile) and release
    tracking. No more SSH just to read logs or bounce a stack.

## Deferred to OPNsense (future)
- **LAN-wide DNS + `*.home` wildcard resolution** (via Unbound). Until then: per-device DNS
  or `/etc/hosts`. AdGuard optional to revisit then (or use Unbound directly).
- **Tailscale** for remote/5G access (OPNsense can be the subnet router).
- **Shared external `homelab` docker network** so cross-stack services (NPM ↔ Glance ↔ future
  stacks) resolve each other by name instead of `beelink-ip:port`.

## TODO before calling it fully done
- **Pin all image tags to digests** (one pass) — currently `latest` across the stack.
- **End-to-end test** once an indexer + a download exist: Jellyseerr request → qBit (via VPN)
  → Sonarr/Radarr hardlink import → plays in Jellyfin with **HW transcode** confirmed on the
  Jellyfin dashboard.

## TODO before Phase 2 done
- **Pin all image tags to digests** in one pass at the end of Phase 2 (currently `latest`).
