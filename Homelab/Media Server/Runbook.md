# Media Server — Runbook

Operating, deploying, and troubleshooting the [[Media Server]]. All commands run from the
**workstation** (Ansible control node), inside `~/Documents/Homelab/infra/`.

---

## Repo layout (`infra/`)

```
infra/
├── ansible.cfg                 # inventory path, become via sudo, vault_password_file
├── requirements.yml            # community.docker, community.general, ansible.posix
├── .vault_pass                 # GITIGNORED — master key that decrypts the vault
├── inventory/
│   ├── hosts.yml               # the beelink host; IP + sudo password come from the vault
│   └── group_vars/all/
│       ├── vars.yml            # non-secret config (PUID/PGID, paths, timezone, local_domain, GPU GIDs)
│       └── vault.yml           # ENCRYPTED secrets (see below)
├── playbooks/
│   ├── site.yml                # full run: common → storage → docker → media_stack → glance → monitoring → administration
│   ├── media.yml               # just the media stack (fast iteration)
│   ├── dashboard.yml           # just Glance
│   ├── monitoring.yml          # just the monitoring stack (Uptime Kuma + Beszel + heartbeat)
│   ├── administration.yml      # just the admin stack (Dozzle + Dockge + Watchtower)
│   └── update.yml              # manual weekly apt upgrade
└── roles/
    ├── common/                 # apt base, locale/timezone, SSH hardening (key-only), stacks_root
    ├── storage/                # Seagate → ext4 → /data + media tree
    ├── docker/                 # Docker Engine + compose plugin, data-root on /home
    ├── media_stack/            # the docker-compose stack (templated) + secrets
    ├── glance/                 # the dashboard (separate compose project)
    ├── monitoring/             # Uptime Kuma + Beszel + healthchecks.io cron  (OBSERVE only)
    └── administration/         # Dozzle (logs) + Dockge (mgmt) + Watchtower (notify)  (ACT on containers)
```

## What's in the vault

`ansible-vault` encrypts `inventory/group_vars/all/vault.yml`. Contents:
- `vault_beelink_host` — the Beelink's LAN IP
- `vault_become_password` — sudo password (lets playbooks run unattended)
- `vault_cyberghost_user` / `vault_cyberghost_password` — CyberGhost OpenVPN credentials
- `vault_cyberghost_client_cert` — OpenVPN **client certificate** (PEM, `BEGIN CERTIFICATE`)
- `vault_cyberghost_client_key` — OpenVPN **client private key** (PEM, `BEGIN PRIVATE KEY`)
- `vault_sonarr_api_key` / `vault_radarr_api_key` — API keys recyclarr uses to push quality
  profiles (Sonarr/Radarr → Settings → General → API Key). Optional: if unset the stack still
  deploys, recyclarr just can't sync until they're filled in (see recyclarr setup below)
- `vault_discord_webhook_url` — Discord channel webhook for alerts (reference; pasted into each tool's UI)
- `vault_healthchecks_ping_url` — the `https://hc-ping.com/<uuid>` heartbeat URL (drives the cron)
- `vault_beszel_agent_key` — Beszel hub's **ssh-ed25519 public key** (the long `ssh-ed25519 AAAA…` value, *not* the token)
- `vault_beszel_agent_token` — Beszel **registration token** (short `xxxx-xxxx-…`); both come from the Add System dialog
- `vault_github_token` — GitHub PAT (read-only public) for Glance's **App Releases** widget — lifts
  the GitHub API cap from 60→5000 req/hr so the widget stops hitting rate limits after redeploys
- `vault_watchtower_notification_url` — **optional** shoutrrr URL for Watchtower's update alerts
  (Discord format `discord://TOKEN@ID`). If unset, Watchtower still runs but only logs to its own
  container (visible in Dozzle) instead of pinging Discord

Edit secrets:
```bash
EDITOR="code --wait" ansible-vault edit inventory/group_vars/all/vault.yml
```
`.vault_pass` (in `infra/`, gitignored, `chmod 600`) is the decryption key — never commit it.
Because `ansible.cfg` points at it, every run is unattended (no vault prompt, no `-K`).

## Everyday operations

```bash
# Sanity: can Ansible reach the box?
ansible beelink -m ping -b

# Deploy / re-deploy the media stack after editing a template
ansible-playbook playbooks/media.yml

# Deploy the dashboard
ansible-playbook playbooks/dashboard.yml

# Deploy the monitoring stack (Uptime Kuma + Beszel + heartbeat cron)
ansible-playbook playbooks/monitoring.yml

# Deploy the administration stack (Dozzle logs + Dockge management + Watchtower notify)
ansible-playbook playbooks/administration.yml

# Deploy AdGuard Home (DNS ad-blocking + *.home names) — first run needs the wizard, see below
ansible-playbook playbooks/adguard.yml

# Full converge (everything)
ansible-playbook playbooks/site.yml

# Weekly OS update (run on your reminder; tells you if a reboot is needed — never auto-reboots)
ansible-playbook playbooks/update.yml
```

**Adding a new service** to the media stack:
1. Add its image tag to `roles/media_stack/defaults/main.yml`.
2. Add the service block to `roles/media_stack/templates/docker-compose.yml.j2`.
3. Add its `appdata` dir to the directory loop in `roles/media_stack/tasks/main.yml`.
4. `ansible-playbook playbooks/media.yml`.

**Secret changes auto-reload the VPN:** the `env.j2`, `client.crt`, and `client.key` tasks
`notify` a handler that restarts gluetun + qBittorrent — so a changed key is actually picked up
(a plain `compose up` does *not* recreate a container just because a mounted file changed).

## AdGuard Home — first-time setup (from the Livebox to blocking)

Network-wide DNS ad/tracker/malware blocking + local `*.home` names. The Ansible role deploys the
container; the rest is a one-time wizard. **Rollout is per-device** — the Orange Livebox can't
distribute a DNS server, so you point each device at the Beelink by hand (fully reversible; all of
this carries over to OPNsense later). See [[Roadmap]] for the LAN-wide escalation path.

**Do the steps in this order.** The first-run **wizard is a small mandatory gate** (ports + login
only) — you can't reach Settings until it's done. All the real config comes *after* it, in the
dashboard. Point your devices at AdGuard **last**, so blocking + `*.home` already work before
anything uses it.

1. **Deploy:** `ansible-playbook playbooks/adguard.yml`. DNS binds `:53` on the Beelink's LAN IP
   (dodges the systemd-resolved stub on `127.0.0.53` — no host change). Admin is on `:3000`
   because NPM owns `:80`.
2. **Wizard (minimal — just get to the dashboard)** → browse `http://<beelink-ip>:3000`:
   - Admin web interface: keep **port 3000**. DNS server: **port 53**. Create the admin login.
   - On the **"Configure your devices / Router"** screen — that's **instructions only**. We do
     per-device (and the Livebox can't set router DNS anyway), so **click straight through to
     "Open Dashboard" — do NOT set up any router here.**
3. **Upstream DNS (post-wizard, in Settings — the wizard does *not* ask for this)** →
   *Settings → DNS settings* → Upstream DNS = `https://dns.quad9.net/dns-query` (DoH — encrypted,
   free malware filter); mode **Parallel/fastest**; **Bootstrap** = `9.9.9.9`. *Apply* + *Test*.
4. **Blocklists** → *Filters → DNS blocklists*: keep the default *AdGuard DNS filter*, then **Add
   blocklist → OISD Full** (`https://big.oisd.nl`). Confirm both are enabled.
5. **Local names** → *Filters → DNS rewrites* → add `*.home` → `<beelink-ip>`. This makes every
   NPM proxy host (`jellyfin.home`, `uptime.home`, …) resolve — the DNS half the Livebox couldn't do.
6. **Point your devices LAST** → set DNS = `<beelink-ip>` on the workstation and phone (Livebox
   DHCP untouched). Watch *Query Log* light up — that's the "it works" moment.
7. **(Optional, now unblocked)** create the NPM proxy hosts from the [[Roadmap]] table in the NPM
   UI (`:81`) — remember **Websockets** on `uptime.home` / `beszel.home`.

**Verify:**
```bash
curl -sI http://<beelink-ip>:3000 | head -1     # admin UI: expect 200/302
dig @<beelink-ip> example.com +short            # real A record
dig @<beelink-ip> doubleclick.net +short        # blocked → 0.0.0.0 (or empty/NXDOMAIN)
dig @<beelink-ip> jellyfin.home +short          # → <beelink-ip> (after the *.home rewrite)
```

### Pointing a device at AdGuard (per-OS)

"Pointing a device" = replacing the Livebox (`192.168.1.1`) with the Beelink (`192.168.1.19`) as
that device's DNS. Do it one device at a time. Include the **secondary** for the failsafe below.

- **Linux (NetworkManager):** `nmcli -t -f NAME connection show --active` to get the name, then
  ```bash
  nmcli connection modify "<NAME>" ipv4.dns "192.168.1.19 192.168.1.1" ipv4.ignore-auto-dns yes
  nmcli connection modify "<NAME>" ipv6.ignore-auto-dns yes ipv6.dns ""   # or ads leak over IPv6
  nmcli connection up "<NAME>"
  ```
  Undo: `ipv4.dns "" ipv4.ignore-auto-dns no ipv6.ignore-auto-dns no`, then `connection up`.
- **Windows 11:** Settings → **Network & internet** → Wi-Fi (or Ethernet) → click the connection →
  **DNS server assignment → Edit → Manual** → toggle **IPv4 on** → Preferred `192.168.1.19`,
  Alternate `192.168.1.1` → Save.
  **IPv6 must also be handled** or Windows leaks DNS over IPv6 (AdGuard listens on IPv4 only). Do
  **not** use the Livebox's IPv6 DNS — it's link-local/dynamic and Windows rejects it. Instead
  toggle **IPv6 on** and set a *fixed filtering* resolver — **AdGuard Public DNS** Preferred
  `2a10:50c0::ad1:ff`, Alternate `2a10:50c0::ad2:ff` (keeps IPv6 + still blocks ads). *Simpler
  alternative:* disable IPv6 on the adapter entirely (adapter Properties → uncheck *Internet
  Protocol Version 6*) so only IPv4 → AdGuard is used — at the cost of IPv6 connectivity.
  *(CLI equivalent, admin PowerShell: `netsh interface ip set dns name="Wi-Fi" static 192.168.1.19`
  then `netsh interface ip add dns name="Wi-Fi" 192.168.1.1 index=2`.)*
- **iOS:** Settings → Wi-Fi → **ⓘ** → **Configure DNS → Manual** → remove existing, Add
  `192.168.1.19`, then Add `192.168.1.1` → Save.
- **Android:** long-press the Wi-Fi network → **Modify → Advanced → IP settings → Static** →
  **DNS 1 = `192.168.1.19`**, **DNS 2 = `192.168.1.1`** (leave **Private DNS = Off** — it only
  takes a DoT hostname, not a LAN IP).

**Confirm** on any device: an ad domain returns `0.0.0.0` and `jellyfin.home` returns
`192.168.1.19`; the AdGuard **Query Log** shows that client's requests.

### Failsafe / resilience — add a secondary DNS

Single-host DNS is a single point of failure: if the Beelink is down (reboot, `update.yml`, power),
a device with *only* `192.168.1.19` loses name resolution. The fix is the **secondary DNS** already
shown above (`192.168.1.19` **first**, `192.168.1.1` second) — on failure the device falls back to
the Livebox and keeps working.

- The container self-heals (`restart: unless-stopped`), so the realistic outage is the **whole box**
  down, not AdGuard crashing — the fallback mainly covers reboots/OS updates.
- **Tradeoff:** while failed over to the Livebox, that traffic is **unfiltered** and `*.home` names
  won't resolve (use `192.168.1.19:port`).
- ⚠️ **`*.home` names vs a secondary DNS (the big gotcha).** `*.home` exists *only* on our AdGuard;
  every other resolver (Livebox, Quad9, **AdGuard-public IPv6** `2a10:50c0::ad1:ff`) answers
  **NXDOMAIN** for it. **Linux/systemd-resolved** sticks to the primary so `.home` works — but
  **Windows (prefers IPv6) and Android** happily query the secondary/IPv6 resolver *even while
  AdGuard is up*, accept that NXDOMAIN, and the local name fails. So on those devices you can't have
  both cleanly: **AdGuard as the *sole* DNS → reliable `.home` but no failsafe**, or **AdGuard +
  secondary → failsafe but `.home` breaks** (reach services via `192.168.1.19:port` there). On
  Windows specifically, the usual fix is to **disable IPv6 on the adapter** (or drop the IPv6 DNS),
  then `ipconfig /flushdns`. Diagnose with `nslookup jellyfin.home 192.168.1.19` (always works —
  forces AdGuard) vs `nslookup jellyfin.home` (uses the system resolver — reveals the wrong one).
- Prefer **`9.9.9.9` (Quad9)** as the secondary instead of the Livebox if you want the degraded
  state to still block malware and hide lookups from Orange.
- **Proper HA comes with OPNsense:** two filtering resolvers (AdGuard on OPNsense + this one, or
  AdGuard + Unbound) handed out by DHCP — a Beelink outage becomes invisible *and* nothing leaks,
  and both know the `.home` zone so local names keep working on every OS. See [[Roadmap]].

**Current choice (until OPNsense):** *keep the failsafe on every device* (secondary DNS =
`192.168.1.1`) — a Beelink reboot must never look like "internet is broken". The accepted cost is
that `.home` names aren't reliable on Windows/Android; that's fine because we **don't rely on them**
— use the **Glance dashboard as the launcher** (`http://192.168.1.19:8280`, whose links are already
`IP:port`, so they work on every device regardless of DNS). Clean `.home` everywhere returns at the
OPNsense phase.

> **If the container won't bind `:53`** (rare — only if resolved is grabbing the LAN IP): add a
> `/etc/systemd/resolved.conf.d/` drop-in with `DNSStubListener=no`, and repoint the host's
> `/etc/resolv.conf` at an **upstream** (`9.9.9.9`) — *never* at AdGuard itself, so the host never
> depends on its own container. Then re-run the playbook. (Move this into the `adguard` role if it
> ever becomes necessary — it isn't today.)

## Recyclarr (VO profiles) & Bazarr (French subs) — first-time setup

Recyclarr syncs TRaSH **French VOSTFR 1080p** profiles (original audio, no dub) into Sonarr/Radarr;
Bazarr then fetches the French subtitles. Config-as-code lives in the `media_stack` role
(`recyclarr.yml.j2`); the web tools (Bazarr, and picking the profile in Seerr) are one-time UI steps.

1. **Grab the API keys** → Sonarr `:8989` and Radarr `:7878` → Settings → General → **API Key**.
2. `ansible-vault edit inventory/group_vars/all/vault.yml` → set `vault_sonarr_api_key` and
   `vault_radarr_api_key` → `ansible-playbook playbooks/media.yml` (re-renders `.env`, which
   recreates the recyclarr container with the keys).
3. **First sync is manual** — recyclarr's cron (`03:00` daily, `recyclarr_cron`) only fires on
   schedule, never on container start. Push the profiles now:
   ```bash
   ansible beelink -m command -a "docker exec recyclarr recyclarr sync" -b
   ```
   On success Sonarr gains **Bluray-WEB-1080p (French VOSTFR)** and Radarr **HD Bluray + WEB
   (French VOSTFR)** under Settings → Profiles, plus their custom formats.
4. **Point the apps at the new profile** (recyclarr *creates* it, it doesn't *assign* it):
   - **Seerr** (`:5055`) → Settings → Services → Radarr/Sonarr → set **Default Quality Profile**
     to the new VOSTFR profile, so every request uses it.
   - Existing library items keep their old profile — mass-edit them in Radarr/Sonarr (Library →
     select all → set profile) if you want them re-graded.
5. **Bazarr** (`:6767`), one-time:
   - Settings → Languages → add **French**, create a **language profile** (French only, or FR+forced).
   - Settings → Providers → enable providers (**OpenSubtitles.com** account, Podnapisi, etc.).
   - Settings → Sonarr → host `sonarr` port `8989` + API key; same for Radarr (`radarr:7878`).
     Assign the French language profile as the default → Bazarr backfills subs on the library.

> **Switching flavour/resolution later** is a one-line change: swap the profile `trash_id` in
> `roles/media_stack/templates/recyclarr.yml.j2` (e.g. VOSTFR → `multi-vo` to keep both audio
> tracks, or a `uhd-…` profile for 4K) → `media.yml` → `docker exec recyclarr recyclarr sync`.
> Grab trash_ids from the TRaSH **French** quality-profile pages
> ([Radarr](https://trash-guides.info/Radarr/radarr-setup-quality-profiles-french-en/) ·
> [Sonarr](https://trash-guides.info/Sonarr/sonarr-setup-quality-profiles-french-en/)).
> `delete_old_custom_formats: true` keeps the apps an exact mirror of the guide on every sync.
>
> **Note on template mechanics:** recyclarr's `include:` feature is *not* used here — the pinned
> config-templates repo ships an **empty `includes.json`**, so includes resolve to nothing (that
> was the original "Unable to find include template" failure). Instead we reference the profile's
> `trash_id` directly, which auto-syncs its default custom-format groups (VOSTFR +1000,
> "Language: Not Original" −10000, quality tiers, LQ/x265 penalties). Verified working 2026-07-14.

## Monitoring & Administration (two stacks, on purpose)

Split by **job, not tool** (see [[Roadmap]] for the why): `monitoring` only **observes** (read-only
sockets, can't change anything), `administration` **acts on** containers (control + updates). Keeping
them apart means the observe layer has no power to break anything. All the web-UI tools have **no
declarative config** — monitors, the Discord webhook, Beszel's agent key, and the Dozzle/Dockge admin
accounts are set up once in each UI after the first deploy; the roles only stand up the containers.

**`monitoring` stack — observe (`docker.sock` read-only):**

| Tool | URL | Covers |
|------|-----|--------|
| Uptime Kuma | `http://<beelink>:3001` | Service/container up-down + alerting (reach-in) |
| Beszel (hub) | `http://<beelink>:8090` | Host + per-container resource metrics **with history** + threshold alerts |
| healthchecks.io | (SaaS, off-box) | Dead-man's-switch — alerts if the Beelink stops pinging (the only "whole box is dead" alarm) |

**`administration` stack — act (`docker.sock` read-write, except Dozzle):**

| Tool | URL | Covers |
|------|-----|--------|
| Dozzle | `http://<beelink>:8888` | **Live log aggregation** across every container (socket `:ro` — reads logs, never controls) |
| Dockge | `http://<beelink>:5001` | **Compose-stack management** — start/stop/restart/update/edit every stack under `/home/stacks` |
| Watchtower | (headless) | **Weekly auto-updater** (Saturday 08:00) — pulls + recreates outdated containers, cleans old images, summary → Discord |

> ⚠️ Uptime Kuma and Beszel both run **on the Beelink they monitor** — if the box dies, they die
> with it and stay silent. That blind spot is exactly why the off-box healthchecks.io heartbeat
> exists. Keep alert channels **external** (Discord webhook), not a self-hosted notifier.

**First-time setup — monitoring (once, after `ansible-playbook playbooks/monitoring.yml`):**

1. **Uptime Kuma** (`:3001`): create the admin user → Settings → Notifications → add **Discord**
   (paste `vault_discord_webhook_url`) → add monitors: HTTP for each *arr/Jellyfin/Seerr, and **Docker**
   monitors for `gluetun` + `qbittorrent` (qBittorrent has no port of its own — it rides gluetun's network
   — so a container-level check is the reliable signal). The Docker monitors work because the socket is
   mounted `:ro` into the container. Uptime Kuma is a *separate* compose project, so all HTTP monitors use
   the **Beelink LAN IP:port**, not container names.
2. **Beszel bootstrap** (chicken-and-egg — the hub generates the agent credentials):
   - `:8090` → create user → **Add System**. Host = the Beelink's LAN IP, Port = `45876`.
   - The dialog's compose snippet shows **two** values — copy **both**: the long
     `KEY: 'ssh-ed25519 AAAA…'` (→ `vault_beszel_agent_key`) **and** the short
     `TOKEN: 'xxxx-…'` (→ `vault_beszel_agent_token`). Do **not** put the token in the key field —
     the agent will crash-loop with `ssh: no key found`.
   - Do **not** copy the compose or run the binary installer from the dialog — the agent is already
     deployed by this role; those are just alternative install methods.
   - `ansible-vault edit inventory/group_vars/all/vault.yml` → set both vars →
     `ansible-playbook playbooks/monitoring.yml` again. The agent dials out to the hub over
     WebSocket (`HUB_URL`), so no hub→agent networking to configure; the system goes green. Then
     add the `discord://TOKEN@ID` notification + a disk-space alert in Beszel.
3. **healthchecks.io**: create a check (period 5 min, small grace) → copy its ping URL into
   `vault_healthchecks_ping_url` → add the **Discord** integration on their side → re-run
   `monitoring.yml` to install the cron (`crontab -l` as root shows `healthchecks-heartbeat`).
4. **Daily status digest → Discord** (positive heartbeat: reports what's fine, not only incidents):
   - In **Uptime Kuma** → Settings → **API Keys** → add a key → copy it into `vault_uptime_kuma_api_key`.
     (This unlocks the `/metrics` endpoint the script reads; Basic auth, empty user + key as password.)
   - **Beszel** → `vault_beszel_user` / `vault_beszel_password`. Beszel (PocketBase) has no static
     API key — reads need a user token minted from credentials, so the script logs in like the UI does.
     Prefer a **dedicated read-only Beszel user** for this over reusing your admin login. The hub's API
     gives per-host CPU/MEM and root-disk %, plus the **`/data` media drive** via its extra-filesystem
     stats (the `.beszel` marker dir mounted at `/extra-filesystems/data`).
   - Create a **new dedicated Discord webhook** for routine reports (keep it out of the alert channel)
     → `vault_status_report_webhook` (raw `https://discord.com/api/webhooks/…` URL, not shoutrrr).
   - `ansible-vault edit inventory/group_vars/all/vault.yml` → set all four → re-run `monitoring.yml`.
     Installs `status-report-daily` in root's crontab (fires once a day, default 08:00); the
     script lives at `/home/stacks/monitoring/status-report.py` and logs to `/var/log/status-report.log`.
   - Test on demand without waiting for cron: `sudo python3 /home/stacks/monitoring/status-report.py`.
     A green embed = all monitors up and every host under 90% CPU/MEM/DISK; red = something needs a look.

**First-time setup — administration (once, after `ansible-playbook playbooks/administration.yml`):**

5. **Dozzle** (`:8888`): zero-config — open it and every container's live logs are there, with
   search and multi-container tail. (No auth by default; fine on the LAN. Add `DOZZLE_AUTH_*`
   env vars if it's ever exposed beyond the LAN.)
6. **Dockge** (`:5001`): create the admin user on first open. It lists every stack under
   `/home/stacks` (`media-stack`, `monitoring`, `glance`, `administration`) and can
   start/stop/restart/update or edit each. **Ansible stays the source of truth** — treat Dockge for
   actions (bounce a stack, pull an update, read a stack's logs), not for permanent config edits, or
   the next role run overwrites them. Anything you want to keep goes back into the role template.
7. **Watchtower** (headless — nothing to open): runs on `WATCHTOWER_SCHEDULE` (every **Saturday 08:00**),
   pulls + recreates any outdated container on the host, cleans up old images (`WATCHTOWER_CLEANUP`), and
   posts a **summary of what changed** to Discord. Set the optional `vault_watchtower_notification_url`
   (shoutrrr `discord://TOKEN@ID`) and re-run `administration.yml` to get the message; without it, the run
   summary just logs to Watchtower's own container (readable in Dozzle). To pin/exclude a service from the
   weekly update, give its container `com.centurylinklabs.watchtower.enable=false` and converge.
   > **Image = `nickfedor/watchtower`, not `containrrr/watchtower`.** The original is unmaintained and
   > its old Docker API client (1.25) crash-loops against modern Engine (`client version 1.25 is too
   > old`). The `nickfedor` fork is the maintained drop-in.
   > **Log gotcha:** `notify=no` on the "Update session completed" line is **not** an error — it's an
   > internal trigger-source flag. A working send shows `channel_status=sent` / `Notification send
   > completed successfully total_urls=1` just above it. Test the channel with a one-off:
   > `docker run --rm --env-file /home/stacks/administration/.env -v /var/run/docker.sock:/var/run/docker.sock nickfedor/watchtower:latest --run-once --monitor-only`

**Stacks layout:** all compose projects live under a single `stacks_root` (`/home/stacks/<name>`)
so Dockge can manage them from one root. Per-service **appdata stays at `/home/<name>/appdata`**
(live data, referenced by absolute path in `.env`) and is *not* moved. The one-time migration is
handled by cleanup tasks in each role that drop the old compose files at `/home/<name>/` — safe to
delete those tasks after the first post-consolidation converge.

**Adding a service to the monitoring stack** mirrors the media-stack flow: image tag in
`roles/monitoring/defaults/main.yml` → service block in the compose template → any `appdata` dir
in the tasks loop → `ansible-playbook playbooks/monitoring.yml`.

## Troubleshooting

```bash
# Container status
ansible beelink -m command -a "docker ps" -b

# Logs (the #1 debugging tool — SSH is only for this)
ansible beelink -m command -a "docker logs --tail 40 <container>" -b

# Restart one service
ansible beelink -m command -a "docker restart <container>" -b
```

If gluetun changes (new key/creds) don't take effect: `docker restart gluetun qbittorrent`
(or just re-run `media.yml` — the handler does it when secrets changed).

## Verification tests (re-run any time)

**VPN kill-switch — the important one:**
```bash
# IP as seen through the VPN (should be the CyberGhost/NL address):
ansible beelink -m command -a "docker run --rm --network=container:gluetun curlimages/curl -s https://ipinfo.io/ip" -b
# the Beelink's real ISP IP (must be DIFFERENT):
ansible beelink -m command -a "curl -s https://ipinfo.io/ip" -b
# kill-switch: stop the VPN and confirm qBittorrent goes fully offline (expect 000):
ansible beelink -m command -a "docker stop gluetun" -b
ansible beelink -m command -a "curl -s -m 5 -o /dev/null -w '%{http_code}\n' http://localhost:8080" -b
ansible beelink -m command -a "docker start gluetun" -b
ansible beelink -m command -a "docker restart qbittorrent" -b
```

**Hardlinks work (no doubled disk usage on import):**
```bash
ansible beelink -m shell -a "docker exec sonarr sh -c 'echo t > /data/torrents/tv/hltest && ln /data/torrents/tv/hltest /data/media/tv/hltest && ls -li /data/torrents/tv/hltest /data/media/tv/hltest && rm -f /data/torrents/tv/hltest /data/media/tv/hltest'" -b
```
Pass = both lines show the **same inode** number.

**Quick Sync device present in Jellyfin:**
```bash
ansible beelink -m command -a "docker exec jellyfin ls -l /dev/dri" -b   # expect renderD128
```
Then in Jellyfin: Dashboard → Playback → Transcoding → Intel QuickSync (QSV), device
`/dev/dri/renderD128`. Full confirmation = play a file that forces a transcode and check the
dashboard shows a **hardware** session.

## Gotchas we hit (lessons learned)

- **CyberGhost OpenVPN needs a client cert + private key**, not just user/password. gluetun
  reads `client.crt` + `client.key` from `appdata/gluetun/`. Watch the mapping: `client.key`
  must be the **PRIVATE KEY** (`BEGIN PRIVATE KEY`), *not* a certificate — mixing them gives
  `Cannot load private key file / PKCS8`. The `<ca>` block is not needed (gluetun has it).
- **A changed mounted file doesn't recreate a container.** `docker compose up` only recreates
  on config changes. Fixed with the secret-change → restart handler.
- **`community.general.filesystem` with `force: true` reformats every run.** It's only a no-op
  when *not* forced. We made `force` conditional: true only when the disk isn't already ext4.
- **`parted` wasn't installed** on minimal Debian — the storage role installs it (+ e2fsprogs).
- **`[privilege_escalation]` needs the underscore** in `ansible.cfg`; with a space it's ignored
  and sudo silently doesn't apply.
- **Port conflicts:** NPM owns 80/443, qBittorrent owns 8080 (via gluetun) → Glance published
  on **8280**, AdGuard would need its UI off 80.
- **Glance in a container** can't see host disks with `type: local` unless host paths are
  bind-mounted in (`/data`, `/home` → `/mnt/host/...:ro`) and referenced as mountpoints.
- **Cross-project container names don't resolve.** Glance (its own project) reaches the media
  services by `beelink-ip:port`, not by container name.
- **Docker's `data-root` does NOT move the containerd image store.** With the containerd
  snapshotter (`docker info` → `driver-type: io.containerd.snapshotter.v1`), image layers live at
  containerd's `root` (default `/var/lib/containerd`, on the **12 GB `/var`**) — *not* under
  `/home/docker`. It silently filled `/var` and blocked all pulls (a large image pull was the one
  that hit it). Fixed by pinning `root = /home/containerd` in `/etc/containerd/config.toml` (now in
  the **docker role**). **One-time migration on an existing box** (the role can't do this safely on
  its own — a config flip + restart without moving data first breaks every running container): stop
  `docker.socket docker.service` then `containerd` → `cp -a /var/lib/containerd/. /home/containerd/`
  (preserves hardlinks + overlay xattrs as root; `rsync` wasn't installed and `apt` can't run with
  `/var` full) → write the config → start `containerd` then `docker` → verify `containerd config dump
  | grep root` and `docker ps` count → then `rm -rf /var/lib/containerd/*`. Done 2026-07-14; `/var`
  100% → 4%. Fresh installs get it right from the role before the first pull, no migration needed.

See [[Roadmap]] for what's intentionally left for the OPNsense phase.
