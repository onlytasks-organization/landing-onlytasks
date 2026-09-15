# Install & operations — Only Tasks landing

Deployment, DNS, Cloudflare, and Ansible instructions for this repository. The root **README** describes the public landing page only.

---

## GitHub Pages

1. Push this repository to GitHub.
2. In the repository, go to **Settings → Pages**.
3. Under **Build and deployment**, set **Source** to **Deploy from a branch**.
4. Choose branch **`main`** and folder **`/` (root)**, then save.
5. Under **Custom domain**, enter **`onlytasks.eu`** and save. This should match the `CNAME` file in the repo root.
6. After DNS resolves, enable **Enforce HTTPS** (can take up to about 24 hours).

If this is a **project** site (URL like `https://<org>.github.io/<repo>/`), a custom apex domain still works once DNS and the custom domain setting are correct.

---

## DNS for GitHub Pages (apex `onlytasks.eu`)

Configure these at **Cloudflare** (or any DNS host) for the apex. If anything looks stale, confirm against [Managing a custom domain for your GitHub Pages site](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain).

### `A` records (name `@` or `onlytasks.eu`)

| IPv4 |
| --- |
| `185.199.108.153` |
| `185.199.109.153` |
| `185.199.110.153` |
| `185.199.111.153` |

### `AAAA` records (recommended, same name)

| IPv6 |
| --- |
| `2606:50c0:8000::153` |
| `2606:50c0:8001::153` |
| `2606:50c0:8002::153` |
| `2606:50c0:8003::153` |

### Optional `www`

GitHub recommends also configuring **`www`**. Create a **`CNAME`** for `www` pointing to your Pages default host (**`<user>.github.io`** or **`<org>.github.io`** only — do not include the repository name).

### OpenShift apps wildcard

Create a **`CNAME`** for `*.apps` pointing at the OpenShift `router-default` load balancer hostname. Leave it **DNS only** (grey cloud). This does not cover `apps.onlytasks.eu` itself.

---

## Cloudflare (recommended defaults)

- **Proxy:** set GitHub Pages **A** / **AAAA** (and usually **`www`** **`CNAME`**) to **DNS only** (grey cloud) for the simplest TLS behavior with GitHub-issued certificates. Keep **`*.apps`** DNS-only as well: the OpenShift router ELB uses PROXY protocol.
- **SSL/TLS:** if you later enable the orange cloud proxy, use **Full** (not Flexible).
- **Registrar:** the domain’s nameservers must be Cloudflare’s for the zone managed in Cloudflare.

---

## Verify DNS

```bash
dig onlytasks.eu +noall +answer -t A
dig onlytasks.eu +noall +answer -t AAAA
dig wildcard-check.apps.onlytasks.eu +noall +answer -t CNAME
```

Results should include the **A** and **AAAA** addresses listed above.

---

## Ansible — Cloudflare DNS

Playbooks under `infra/playbooks/` manage DNS for **onlytasks.eu**:

- **`cloudflare_github_pages.yml`** — apex **`A`** / **`AAAA`** and optional **`www`** **`CNAME`** for GitHub Pages.
- **`cloudflare_openshift_apps.yml`** — wildcard **`CNAME`** `*.apps.onlytasks.eu` to the OpenShift router (DNS only, grey cloud).

### Prerequisites

- Python 3 and Ansible (`pip install ansible` or your OS package).
- A Cloudflare zone for `onlytasks.eu` with the domain using Cloudflare nameservers.

### Cloudflare API token

Create **My Profile → API Tokens → Create Token** (custom token):

| Permission | Access |
| --- | --- |
| Zone → DNS | Edit |
| Zone → Zone | Read |

**Zone resources:** Include → Specific zone → `onlytasks.eu`.

Store the token only in **Ansible Vault** (see below), or export `CLOUDFLARE_TOKEN` for a one-off run (avoid committing it).

### Install collections

From the `infra/` directory:

```bash
cd infra
ansible-galaxy collection install -r collections/requirements.yml -p .collections
```

`infra/ansible.cfg` sets `collections_paths` to `.collections` relative to `infra/`.

### Vault

```bash
cd infra
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml
```

Edit the encrypted file and set `cloudflare_api_token` to your token.

`infra/group_vars/all/vault.yml` is listed in `.gitignore` so it is not committed by default.

### Variables

Edit `infra/group_vars/all/main.yml`:

- **`github_pages_default_host`** — must be **`<user>.github.io`** or **`<org>.github.io`** (no repo path). Default is `onlytasks.github.io`; change it if your GitHub Pages default host differs.
- **`cloudflare_manage_www`** — set to `false` if you do not want a `www` **CNAME**.
- **`openshift_router_cname_target`** — hostname of the OpenShift `router-default` load balancer.

GitHub’s apex **A** / **AAAA** values live in `infra/roles/cloudflare_github_pages/defaults/main.yml`; refresh them periodically from [GitHub’s apex domain documentation](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain).

### Run the playbook

```bash
cd infra
ansible-playbook -i inventory playbooks/cloudflare_github_pages.yml --ask-vault-pass
ansible-playbook -i inventory playbooks/cloudflare_openshift_apps.yml --ask-vault-pass
```

Check mode:

```bash
cd infra
ansible-playbook -i inventory playbooks/cloudflare_github_pages.yml --ask-vault-pass --check
ansible-playbook -i inventory playbooks/cloudflare_openshift_apps.yml --ask-vault-pass --check
```

If you use `CLOUDFLARE_TOKEN` in the environment instead of vault, you can omit `--ask-vault-pass` and ensure `cloudflare_api_token` is not required from vault (the module reads `CLOUDFLARE_TOKEN` when set).

### Quick start (copy-paste)

```bash
cd infra
ansible-galaxy collection install -r collections/requirements.yml -p .collections
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml
ansible-playbook -i inventory playbooks/cloudflare_github_pages.yml --ask-vault-pass
ansible-playbook -i inventory playbooks/cloudflare_openshift_apps.yml --ask-vault-pass
```

The in-repo `infra/README.md` mirrors the Ansible section for anyone browsing the repository without this file.
