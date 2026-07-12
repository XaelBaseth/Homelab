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
│   ├── site.yml                # full run: common → storage → docker → media_stack → glance
│   ├── media.yml               # just the media stack (fast iteration)
│   ├── dashboard.yml           # just Glance
│   └── update.yml              # manual weekly apt upgrade
└── roles/
    ├── common/                 # apt base, locale/timezone, SSH hardening (key-only)
    ├── storage/                # Seagate → ext4 → /data + media tree
    ├── docker/                 # Docker Engine + compose plugin, data-root on /home
    ├── media_stack/            # the docker-compose stack (templated) + secrets
    └── glance/                 # the dashboard (separate compose project)
```

## What's in the vault

`ansible-vault` encrypts `inventory/group_vars/all/vault.yml`. Contents:
- `vault_beelink_host` — the Beelink's LAN IP
- `vault_become_password` — sudo password (lets playbooks run unattended)
- `vault_cyberghost_user` / `vault_cyberghost_password` — CyberGhost OpenVPN credentials
- `vault_cyberghost_client_cert` — OpenVPN **client certificate** (PEM, `BEGIN CERTIFICATE`)
- `vault_cyberghost_client_key` — OpenVPN **client private key** (PEM, `BEGIN PRIVATE KEY`)

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

See [[Roadmap]] for what's intentionally left for the OPNsense phase.
