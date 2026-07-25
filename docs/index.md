# Homelab

Internal documentation for the Beelink homelab, served by MkDocs Material at
`http://192.168.1.19:8081`.

## How this works

These pages are **plain markdown files** in `docs/` at the root of the homelab repo. They are
deployed by Ansible:

```bash
cd Homelab/infra
ansible-playbook playbooks/webapps.yml
```

The playbook rsyncs `docs/` onto the Beelink and the `docs` container picks the change up on its
own — MkDocs watches the directory and rebuilds in place, so there is nothing to restart. Adding a
page is: create the `.md` file, run the playbook. It appears in the sidebar automatically, because
`mkdocs.yml` has no hard-coded `nav`.

!!! warning "This is not the Obsidian vault"
    The vault under `Homelab/` is a separate thing and is **not** published here. Use ordinary
    markdown links between these pages — `[text](other-page.md)`. Obsidian `[[wikilinks]]` render
    as literal text, because making them work needs a third-party MkDocs plugin, and any plugin
    not bundled in the image would force a custom Dockerfile.

## What's here

Nothing yet beyond this page. Suggested starting points:

- the host, the network and what listens on which port
- how to recover each stack from scratch
- the things you only learn once and forget by the next time

Operational runbooks currently live in the Obsidian vault (`Homelab/Media Server/Runbook.md`).
Move them here as they stabilise, or keep the split — the vault for notes-in-progress, this site
for the settled version.
