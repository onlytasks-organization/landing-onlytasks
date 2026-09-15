# Ansible — Cloudflare DNS

Playbooks under `playbooks/` manage DNS for **onlytasks.eu**:

- **`cloudflare_github_pages.yml`** — apex **`A`** / **`AAAA`** and optional **`www`** **`CNAME`** for GitHub Pages.
- **`cloudflare_openshift_apps.yml`** — wildcard **`CNAME`** `*.apps.onlytasks.eu` to the OpenShift router (DNS only).

The repository **README** describes the public landing only. This file documents the Ansible layout; see the root **`INSTALL.md`** for GitHub Pages, DNS, Cloudflare, and Ansible steps in one place.

## Prerequisites

- Python 3 and Ansible (`pip install ansible` or your OS package).
- A Cloudflare zone for `onlytasks.eu` with the domain using Cloudflare nameservers.

## Cloudflare API token

Create **My Profile → API Tokens → Create Token** (custom token):

| Permission | Access |
| --- | --- |
| Zone → DNS | Edit |
| Zone → Zone | Read |

**Zone resources:** Include → Specific zone → `onlytasks.eu`.

Store the token only in **Ansible Vault** (see below), or export `CLOUDFLARE_TOKEN` for a one-off run (avoid committing it).

## Install collections

From this `infra/` directory:

```bash
ansible-galaxy collection install -r collections/requirements.yml -p .collections
```

`ansible.cfg` sets `collections_paths` to `.collections` relative to `infra/`.

## Vault

```bash
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml
```

Edit the encrypted file and set `cloudflare_api_token` to your token.

`group_vars/all/vault.yml` is listed in the repository `.gitignore` so it is not committed by default.

## Variables

Edit [`group_vars/all/main.yml`](group_vars/all/main.yml):

- **`github_pages_default_host`** — must be **`<user>.github.io`** or **`<org>.github.io`** (no repo path). Default is `onlytasks.github.io`; change it if your GitHub Pages default host differs.
- **`cloudflare_manage_www`** — set to `false` if you do not want a `www` **CNAME**.
- **`openshift_router_cname_target`** — hostname of the OpenShift `router-default` load balancer.

GitHub’s apex **A** / **AAAA** values live in [`roles/cloudflare_github_pages/defaults/main.yml`](roles/cloudflare_github_pages/defaults/main.yml); refresh them periodically from [GitHub’s apex domain documentation](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain).

## Run

```bash
cd infra
ansible-playbook -i inventory playbooks/cloudflare_github_pages.yml --ask-vault-pass
ansible-playbook -i inventory playbooks/cloudflare_openshift_apps.yml --ask-vault-pass
```

Check mode:

```bash
ansible-playbook -i inventory playbooks/cloudflare_github_pages.yml --ask-vault-pass --check
ansible-playbook -i inventory playbooks/cloudflare_openshift_apps.yml --ask-vault-pass --check
```

If you use `CLOUDFLARE_TOKEN` in the environment instead of vault, you can omit `--ask-vault-pass` and ensure `cloudflare_api_token` is not required from vault (the module reads `CLOUDFLARE_TOKEN` when set).
