# SSO Setup — the click-through half

Single sign-on for the media stack: **LLDAP** holds the users, **Authelia** runs the login
portal, **NPM** asks Authelia about every request before it reaches an app. See [[Media Server]]
for the stack itself and [[Runbook]] for day-to-day operations.

**Ansible has already done everything it can.** LLDAP and Authelia are deployed and healthy, the
`homelab` docker network exists, the local CA and the `*.media.home` wildcard certificate are
generated, and the forward-auth endpoint is templated into NPM's nginx config. What remains is
click-through work in five web UIs — that's this page.

> **Scope: the media stack only.** Dozzle, Dockge, Uptime Kuma, Beszel, AdGuard and Glance keep
> their own logins on purpose. That is deliberate: if the auth layer ever breaks, Dockge and
> Dozzle are still reachable on their plain ports and you can fix it without SSH.

## Why the names change

Every protected host moves from `sonarr.home` to **`sonarr.media.home`**.

The SSO session is one browser cookie, and a cookie is only readable by hostnames sharing its
domain. `sonarr.home` and `radarr.home` share only `home` — a single label — and browsers flatly
refuse to set cookies there. Adding one label gives them a shared `media.home` suffix, which is
legal, so one login covers the whole stack. This is also why Authelia insists on HTTPS: it only
issues `Secure` cookies, which is what the self-signed CA below is for.

Your existing `*.home` rewrite and every current URL keep working — nothing is taken away.

---

## Do the steps in this order

Each one depends on the previous. Step 5 (Jellyfin) is deliberately last because it's the only
one that touches an app your household actually uses.

### 1. AdGuard — make the new names resolve

`http://192.168.1.19:3000` → *Filters → DNS rewrites* → **Add rewrite**:

| Domain | Answer |
|---|---|
| `*.media.home` | `192.168.1.19` |

Leave the existing `*.home` rewrite alone.

**Verify:**
```bash
dig @192.168.1.19 sonarr.media.home +short     # → 192.168.1.19
dig @192.168.1.19 jellyfin.home +short         # → 192.168.1.19 (old names still fine)
```

### 2. LLDAP — create your account

`http://192.168.1.19:17170`. Log in as **`admin`**; the password is in the vault:

```bash
cd Homelab/infra
ansible-vault view inventory/group_vars/all/vault.yml | grep vault_lldap_admin_password
```

The `authelia` service account already exists — Ansible created it with a read-only role so it
can check logins but cannot alter the directory. **Don't touch it.**

Create **your own** user (*Users → Create a user*):

- **User ID** — ⚠️ use *exactly* your existing Jellyfin username. The Jellyfin LDAP plugin
  matches on this, and a mismatch creates a second, empty account instead of reusing yours.
- **Email** and **Display name** — anything sensible.
- Set a password. This becomes your one password for the whole stack.

Repeat for anyone else who needs access.

**Verify:** log out, log back in as your new user. LLDAP lets ordinary users edit their own
profile and password — that's where people change their own password from now on.

### 3. NPM — certificate, then the proxy hosts

`http://192.168.1.19:81`.

**3a. Upload the certificate.** *SSL Certificates → Add SSL Certificate → Custom*:

- **Name:** `media.home wildcard`
- **Certificate Key:** `Homelab/infra/certs/wildcard.key`
- **Certificate:** `Homelab/infra/certs/wildcard.crt`
- Leave the intermediate empty.

**3b. Create the proxy hosts.** For each row: *Hosts → Proxy Hosts → Add Proxy Host*. On the
**Details** tab set the domain, scheme `http`, the forward host/port below, and enable
**Block Common Exploits**. On the **SSL** tab pick the `media.home wildcard` certificate and
enable **Force SSL** + **HTTP/2**.

| Domain | Forward host | Port | Websockets | Gate it? |
|---|---|---|---|---|
| `auth.media.home` | `authelia` | 9091 | — | **NO** — never gate the portal |
| `sonarr.media.home` | `sonarr` | 8989 | — | yes |
| `radarr.media.home` | `radarr` | 7878 | — | yes |
| `prowlarr.media.home` | `prowlarr` | 9696 | — | yes |
| `bazarr.media.home` | `bazarr` | 6767 | — | yes |
| `qbit.media.home` | `gluetun` | 8080 | **on** | yes |
| `maintainerr.media.home` | `maintainerr` | 6246 | — | yes |
| `jellyfin.media.home` | `jellyfin` | 8096 | **on** | **NO** — LDAP handles it |

qBittorrent forwards to `gluetun` because it runs inside the VPN container's network namespace
and has no address of its own.

**3c. Gate the seven marked hosts.** Edit each → **Advanced** tab → paste exactly:

```nginx
auth_request /authelia;
auth_request_set $target_url $scheme://$http_host$request_uri;
auth_request_set $user $upstream_http_remote_user;
auth_request_set $groups $upstream_http_remote_groups;
proxy_set_header Remote-User $user;
proxy_set_header Remote-Groups $groups;
error_page 401 =302 https://auth.media.home/?rd=$target_url;
```

The heavy lifting — the `/authelia` endpoint this calls — is already in
`/data/nginx/custom/server_proxy.conf`, rendered by Ansible from
`roles/media_stack/templates/npm-server_proxy.conf.j2`. It's included into every host
automatically, so these seven lines are all you paste.

> **Why this snippet and not a `location /` block:** `auth_request` is valid at server level, so
> this leaves the Details tab fully working — including the Websockets toggle. Wrapping it in a
> `location /` would override NPM's own generated block and silently break those settings.

**Never** put this on `auth.media.home` (redirect loop) or `jellyfin.media.home` (kills every TV
and phone client instantly).

### 4. Install the CA on your devices

Without this, every page shows a certificate warning. `Homelab/infra/certs/ca.crt`, valid 10 years.

- **Linux:** `sudo cp ca.crt /usr/local/share/ca-certificates/homelab.crt && sudo update-ca-certificates`
- **Firefox** keeps its own store: *Settings → Privacy & Security → Certificates → View
  Certificates → Authorities → Import*, tick **Trust this CA to identify websites**.
- **Windows:** double-click → *Install Certificate → Local Machine → Place all in* →
  **Trusted Root Certification Authorities**.
- **Android:** *Settings → Security → Encryption & credentials → Install a certificate → CA
  certificate*.
- **iOS:** AirDrop/email it, install the profile, then *Settings → General → About → Certificate
  Trust Settings* and switch it **on** — iOS ignores the CA until you do this second step.

Only devices that open admin pages need this. TVs don't — Jellyfin stays on plain `:8096`.

### 5. Jellyfin — LDAP, not forward auth

Jellyfin is the one app that cannot use the portal: forward auth and OIDC both need a browser
redirect, and Android TV, Roku and Swiftfin only know how to POST a username and password. So it
validates against the same directory instead — same credentials, its own login form.

*Dashboard → Plugins → Catalog → **LDAP Authentication*** → install → **restart Jellyfin** →
*Dashboard → Plugins → LDAP-Auth*:

| Setting | Value |
|---|---|
| LDAP Server | `lldap` |
| LDAP Port | `3890` |
| Secure LDAP | off (container-to-container on the `homelab` network) |
| LDAP Bind User | `uid=authelia,ou=people,dc=media,dc=home` |
| LDAP Bind password | `vault_authelia_ldap_password` (read it from the vault) |
| LDAP Base DN for searches | `ou=people,dc=media,dc=home` |
| LDAP Search Filter | `(objectClass=person)` |
| LDAP Search Attributes | `uid,cn,mail` |
| LDAP Username Attribute | `uid` |
| Enable User Creation | on |

Hit **Save and Test LDAP Server Settings** — it must say the bind succeeded before you go on.

⚠️ **Keep your existing local admin account.** The plugin doesn't disable local logins, and that
account is your way back in if LDAP misbehaves. Test with a throwaway LLDAP user *before*
trusting it with your own.

### 6. qBittorrent — stop it rejecting the proxy ✅ already applied

qBittorrent needs **two** checks turned off, in *Options → WebUI*. Both are already done; this
records what and why, because the symptom is identical and thoroughly confusing.

- **Enable Host header validation** → off. Otherwise every proxied request is answered
  `Unauthorized`.
- **Enable Cross-Site Request Forgery (CSRF) protection** → off. Same `Unauthorized`, different
  cause — and this one only appears *after* a successful Authelia login, which makes it look
  like SSO is broken when it isn't.

**Why CSRF protection has to go.** When Authelia finishes authenticating you it redirects your
browser to qBittorrent, and the browser attaches the portal's address as the `Referer` header.
qBittorrent compares that against its own hostname, sees a different host, classifies the
request as cross-site and rejects it with a 12-byte `Unauthorized` body. It is behaving exactly
as designed — SSO's redirect is structurally indistinguishable from a CSRF attempt.

The tidy fix would be stripping that header at the proxy, but **it cannot be done from NPM's
Advanced tab**: NPM's own `location` block sets `add_header`/`proxy_set_header`, and nginx drops
*all* inherited directives of a type when the inner level defines any. Anything you put at server
level is silently ignored. Disabling the check in qBittorrent is the workable option.

What still protects it: no published host port, so NPM is the only route in; Authelia in front;
and qBittorrent's own login underneath. That last one is why qBittorrent keeps its second prompt
while the *arr apps don't (step 7).

> **Same caveat, harmless here:** the `Remote-User` / `Remote-Groups` lines in the step 3c
> snippet are dropped by nginx for the identical reason. Nothing reads them today — the *arr
> apps use `External` auth — so they are left in place as documentation of intent.

### 7. The *arr apps — stop the second login ✅ already applied

Forward auth only controls *reaching* an app; each app's own login sits underneath it. Left
alone, you authenticate at Authelia and then again at Sonarr. The fix is telling them to trust
the proxy: *Settings → General → Security → **Authentication Method: External***.

**This is already done** for Sonarr, Radarr and Prowlarr (set via their APIs). Bazarr and
Maintainerr never had authentication to begin with, so there was nothing to change.

⚠️ **`External` means the app authenticates nobody at all** — it trusts whatever reaches it,
completely. That is only safe because the host ports are now unpublished, so NPM (and therefore
Authelia) is the sole route in. **If you ever re-publish one of those ports, that app is wide
open to your whole LAN.** The two settings are a pair; don't change one without the other.

This is exactly why qBittorrent keeps its own login: it shares gluetun's network namespace, so
"only reachable through NPM" is harder to guarantee for it. One extra prompt on one app is the
cheaper trade.

---

## Verify

```bash
# names resolve
dig @192.168.1.19 sonarr.media.home +short

# TLS is valid and the request is being intercepted (302 = redirect to the portal)
curl -sI --cacert Homelab/infra/certs/ca.crt https://sonarr.media.home | head -1
```

Then, in a **private browser window** — the real test:

1. Open `https://sonarr.media.home` → you land on the Authelia portal.
2. Log in with your LLDAP credentials → you arrive at Sonarr.
3. Open `https://radarr.media.home` → **no second login.** That is single sign-on working.
4. Open `https://jellyfin.media.home` → Jellyfin's *own* login form (expected), and your LLDAP
   password works.
5. From your phone or TV on plain `192.168.1.19:8096` → Jellyfin still logs in normally.

Nothing else should have moved: Dozzle, Dockge, Uptime Kuma, Beszel, AdGuard and Glance all load
on their usual ports with their usual logins, and Prowlarr still syncs to Sonarr/Radarr
(container-to-container API calls never touch NPM).

---

## Troubleshooting

**Redirect loop on every page** — the `auth_request` snippet got pasted onto `auth.media.home`.
Remove it there.

**`Unauthorized` right after logging in successfully** — that page is the *app* rejecting you,
not Authelia. Confirm by checking which side answered:
```bash
# NPM's log for that host: "Sent-to <container>" with 401 and a 12-byte body = the app said no
tail -5 /home/media-stack/appdata/npm/data/logs/proxy-host-<N>_access.log
docker logs authelia --since 10m | tail -5      # a 401 here instead = Authelia said no
```
For qBittorrent this is the CSRF check — see step 6.

**502 on a gated host** — NPM can't reach Authelia. Check:
```bash
docker exec npm getent hosts authelia      # must resolve
docker logs authelia --tail 20
```

**Authelia won't start, `LDAP Result Code 49`** — the bind account or its password is wrong.
Re-run `ansible-playbook playbooks/identity.yml`; it reconciles the account from the vault.

**Everyone logged out after a reboot** — expected. Sessions live in memory (no Redis), so an
Authelia restart clears them. Log in again.

**Locked out after failed attempts** — regulation bans for 5 minutes after 3 failures in 2
minutes. Wait it out, or `docker restart authelia`.

**Password changes** — users do it themselves in LLDAP at `:17170`. Authelia is read-only
against the directory by design and cannot change passwords.

**Reading Authelia's notifications** (it has no SMTP):
```bash
docker exec authelia cat /config/notification.txt
```

---

## What changed when the ports closed

Sonarr, Radarr, Prowlarr, Bazarr, Maintainerr and qBittorrent's WebUI **no longer publish host
ports**. `192.168.1.19:8989` is refused; the only way in is `https://sonarr.media.home`, which
means through Authelia. The gate is real enforcement now, not a suggestion.

Consequences worth knowing:

- **No more IP:port fallback for those six.** If NPM or DNS breaks you cannot reach them at all.
  Dockge (`:5001`) and Dozzle (`:8888`) deliberately still publish, and that is your way back in
  — you can restart or inspect anything from Dockge without SSH.
- **Uptime Kuma and Glance now probe by container name** (`http://sonarr:8989`) over the shared
  `homelab` network. Both were moved onto it and their checks updated. Glance tiles use
  `check-url` for the probe and still *link* to the `https://…media.home` address.
- **Untouched and still on their own ports:** Jellyfin (`:8096`, TV clients need it), Seerr
  (`:5055`, mobile app), NPM, AdGuard, Uptime Kuma, Beszel, Dozzle, Dockge, Glance.
- **Internal traffic was never affected** — Prowlarr→Sonarr, Recyclarr, Seerr→Radarr and the
  rest are container-to-container API calls that never went through a host port.

If you ever need to re-publish a port temporarily, remember to set that app's Authentication
Method back to `Forms` first (step 7).
