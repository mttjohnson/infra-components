# packer

Install [HashiCorp Packer](https://www.packer.io/) system-wide on Debian and Ubuntu hosts via the upstream HashiCorp apt repository (`apt.releases.hashicorp.com`), with a pinned signing-key checksum and fingerprint and a per-repo (not system-wide) keyring.

https://developer.hashicorp.com/packer/install

## Why this role exists

Packer is a build-time CLI for creating machine images (cloud AMIs, QEMU disks, Vagrant boxes, etc.). It's a single binary; HashiCorp's apt repo is the official distribution channel for Debian-family hosts. This role wraps the install with two security guarantees beyond HashiCorp's published shell-snippet:

1. **The signing key is pinned.** Both its SHA256 and its primary GPG fingerprint are declared in role defaults. The role aborts if the downloaded key doesn't match — a silent key swap is impossible.
2. **The signing key is scoped to this one repo.** It's installed at `/usr/share/keyrings/hashicorp.gpg` and explicitly referenced via `signed_by` on the HashiCorp apt source. apt will only trust this key for HashiCorp packages, not system-wide for every repo configured on the host.

## Install scope

- **System-wide binary** at `/usr/bin/packer` via apt. Available to all users on the host.
- **Repo and key**:
    - Apt source written to `/etc/apt/sources.list.d/hashicorp.sources` (deb822 format) via `ansible.builtin.deb822_repository`, scoped to the host's dpkg architecture.
    - Dearmored signing key installed at `/usr/share/keyrings/hashicorp.gpg`, referenced from the source via `signed_by` so its trust is **scoped to this single repo only**.
    - Armored key staged at `/root/hashicorp.asc` for repeatable verification on future runs.
- **No plugins managed here**. Packer plugins (`amazon`, `qemu`, `vagrant`, …) are project-scoped and should be declared in each project's `required_plugins` block and installed via `packer init`.
- **No shell init managed here**. `PACKER_LOG`, `PACKER_CACHE_DIR`, `PACKER_PLUGIN_PATH`, etc. are set per-invocation or per-project (e.g. via a project `.envrc` if you're using `direnv`).

## How verification works

On every run:

1. The HashiCorp armored key is downloaded with `get_url` enforcing `packer_apt_key_checksum`. If HashiCorp ever serves different bytes at the key URL without the operator updating this checksum, the download fails.
2. `gpg --show-keys --with-colons --with-fingerprint` parses the downloaded key. The role extracts the primary-key fingerprint and asserts it equals `packer_apt_key_fingerprint`. If HashiCorp rotates the key (and the operator updated the checksum but not the fingerprint, or the key was substituted upstream), the assertion fires with a message pointing at HashiCorp's published trust page.
3. Only after both checks pass is the key dearmored into the keyring. The keyring is referenced from the apt source via `signed_by`, scoping its trust to this one repo.

The fingerprint check is technically redundant when the checksum is pinned — both will fail together on rotation. The fingerprint pin is there because it's the operator-readable identity gate: the hex string you compare against HashiCorp's trust page when reviewing a rotation, rather than a 64-char SHA256 with no semantic meaning.

## Requirements

- Ansible 2.15+ (uses `ansible.builtin.deb822_repository`, added in 2.15)
- Debian or Ubuntu target host
- Outbound HTTPS to `apt.releases.hashicorp.com`
- The role installs `gnupg`, `ca-certificates`, and `curl` as prerequisites if they're not already present

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `packer_version` | `"1.15.3"` | Bare semver (no `v` prefix, no `-1` apt-revision suffix). The role appends `-1` internally. Set to `"latest"` to track whatever HashiCorp's apt repo lists as current at run time. |
| `packer_apt_key_url` | `https://apt.releases.hashicorp.com/gpg` | URL of HashiCorp's ASCII-armored apt signing key. |
| `packer_apt_key_download_path` | `/root/hashicorp.asc` | Where the armored key is staged on the target. |
| `packer_apt_key_checksum` | `sha256:cafb01…` | SHA256 of the downloaded key file. Pinned. |
| `packer_apt_key_fingerprint` | `798AEC65…E701` | Primary GPG fingerprint of the key. Pinned. Asserted on every run. |
| `packer_apt_keyring_path` | `/usr/share/keyrings/hashicorp.gpg` | Where the dearmored key is installed for `signed_by` reference. |
| `packer_apt_repo_url` | `https://apt.releases.hashicorp.com` | HashiCorp apt repo base URL. |
| `packer_apt_suite` | `{{ ansible_facts['distribution_release'] }}` | apt suite (release codename). |

### Pinned vs latest version

| `packer_version` | Behavior |
|---|---|
| `"1.15.3"` (or any specific version) | apt installs `packer=1.15.3-1` exactly. Re-running with the same version is a no-op; bumping the variable triggers an upgrade (or a downgrade — `allow_downgrade: true` is set so the role enforces the pinned version even if a newer one is currently installed). |
| `"latest"` | apt runs with `state: latest`. Re-running upgrades to whatever is now newest in the repo, no-op if already at newest. The role does not record which concrete version got installed; check with `dpkg -s packer` or `packer --version` after the run if you need to know. |

## Idempotency

- The keyring download is checksum-verified; `get_url` skips re-download when the local file matches.
- The dearmor step runs only when the armored key changed or the keyring file is missing (handles the operator-deleted-the-keyring case).
- The fingerprint assertion runs on every converge — continuous identity verification is cheap and worth it.
- `deb822_repository` is idempotent on rendered `.sources` content. The "Update apt cache" handler fires only when the repo entry changes; `meta: flush_handlers` forces it to run before the install task.
- The install task itself is naturally idempotent under both pinned and `latest` modes.

## When the keys rotate

When HashiCorp rotates their signing key, the role will fail on the get_url checksum check (and would fail on the fingerprint assertion if you bumped only the checksum). The fail message points at https://www.hashicorp.com/trust/security as the source of truth for the current key fingerprint. The expected workflow:

1. Verify the new fingerprint on HashiCorp's trust page (and ideally cross-check via a second source).
2. Recompute the checksum:
   ```
   curl -fsSL https://apt.releases.hashicorp.com/gpg | sha256sum
   ```
3. Update `packer_apt_key_checksum` and `packer_apt_key_fingerprint` in role defaults (or override per-environment).
4. Re-run the role; the new key gets dearmored into the keyring and the apt source picks it up automatically.

## Example Playbooks

Default (install pinned Packer):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.packer
```

Track latest (auto-upgrades on each run):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.packer
      vars:
        packer_version: latest
```

Pin a specific version per environment:

```yaml
# group_vars/dev.yml
packer_version: "1.15.3"

# group_vars/prod.yml
packer_version: "1.14.3"
```

## License note

Packer is distributed under HashiCorp's [Business Source License (BSL)](https://github.com/hashicorp/packer/blob/main/LICENSE) since the 2023 license change. Installing it via this role is unaffected — HashiCorp's apt repo remains the official distribution channel — but worth being aware of for downstream usage decisions.

## License

MIT
