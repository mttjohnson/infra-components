# tenv

Install [tenv](https://github.com/tofuutils/tenv) — a single-binary version manager for Terraform, OpenTofu, Terragrunt, Terramate, and Atmos — on Debian and Ubuntu hosts via the upstream `.deb` package, with cosign signature verification of the package itself before install.

https://tofuutils.github.io/tenv/

## Why this role exists

A developer machine often needs to bounce between several Terraform / OpenTofu versions: one for ongoing production maintenance, one for testing the next release, sometimes a third for a specific module pinned to an older version. `tenv` makes that switch a `tenv use <version>` away and respects `.terraform-version` / `.opentofu-version` / `.terragrunt-version` files in the repo.

## Install scope: system binary, per-user state

The `.deb` installs only the `tenv` binary (system-wide, to `/usr/bin/tenv`). The actual managed Terraform / OpenTofu / Terragrunt versions live in `TENV_ROOT`, which defaults to `${HOME}/.tenv` — so each user gets an independent cache of tool versions even though the binary is shared.

There is no built-in `tenv self-update`. To get a newer `tenv` (which may be required to install brand-new Terraform / OpenTofu releases), an admin re-runs this role with a bumped `tenv_version`. tenv releases tend to be infrequent (~monthly), and a stale tenv only blocks installing very recent tool releases — existing pinned versions keep working.

## How verification works

Each release artifact (including the `.deb`) is published with its own `.sig` and `.pem` files signed via Sigstore keyless signing through the project's GitHub Actions release workflow. With `tenv_verify: signature` (the default), this role:

1. Downloads the `.deb`, its `.sig`, and its `.pem` into a cache directory.
2. Calls `cosign verify-blob` against those three files, pinning the certificate identity to the upstream release workflow at the requested tag (`https://github.com/tofuutils/tenv/.github/workflows/release.yml@refs/tags/v<version>`) and the OIDC issuer to GitHub Actions (`https://token.actions.githubusercontent.com`).
3. Only after verification succeeds is the `.deb` installed via `apt`.
4. The cache directory is removed.

The role does **not** install cosign itself — it expects `cosign` on PATH (or at the path you set in `tenv_cosign_binary`). Install it first with the [`cosign`](../cosign/) role in this repo, or skip verification with `tenv_verify: checksum_only` or `tenv_verify: none`.

## Requirements

- Ansible 2.15+
- Debian or Ubuntu target host
- Architecture: `amd64` or `arm64`
- For `tenv_verify: signature` (default): a working `cosign` binary on the target, plus outbound HTTPS to GitHub releases, the Sigstore TUF root, and the Rekor transparency log.

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `tenv_version` | `"4.11.1"` | Version to install (no `v` prefix). `"latest"` queries the GitHub API. |
| `tenv_state` | `"present"` | `present` or `absent`. |
| `tenv_verify` | `"signature"` | `signature`, `checksum_only`, or `none`. See verification levels below. |
| `tenv_cosign_binary` | `"cosign"` | Path/name of cosign binary used in signature mode. |
| `tenv_cert_identity` | upstream workflow URL (templated) | `{version}` is substituted at runtime. |
| `tenv_cert_oidc_issuer` | `https://token.actions.githubusercontent.com` | OIDC issuer for the signing identity. |
| `tenv_cache_dir` | `/var/cache/ansible-tenv` | Staging dir for downloads. Cleaned up after install. |
| `tenv_download_base_url` | GitHub releases URL | Override for mirrors / air-gapped installs. |
| `tenv_github_api_url` | GitHub API URL | Used only when `tenv_version: latest`. |
| `tenv_setup_shell_init` | `true` | Whether to write the per-user TENV_ROOT/PATH/completion block. |
| `tenv_user` | `{{ ansible_user_id }}` | User to configure shell init for. Set `tenv_setup_shell_init: false` to skip per-user setup entirely. |
| `tenv_group` | `{{ ansible_user_id }}` | Group ownership for files written into the user's home. |
| `tenv_root` | `""` (→ `~/.tenv`) | Value to set for `TENV_ROOT`. |
| `tenv_shell_rc` | `""` (→ `~/.bashrc`) | Shell rc file the init block is written into. |
| `tenv_completion_path` | `""` (→ `~/.tenv.completion.bash`) | Where the bash completion script is written. |

## Verification levels

| Mode | Integrity | Authenticity | Network deps | Cost |
|---|---|---|---|---|
| `signature` (default) | ✅ | ✅ (Sigstore keyless, cert chain + Rekor + signed timestamp, identity pinned to the release tag) | GitHub + Sigstore TUF + Rekor | Requires cosign on the target |
| `checksum_only` | ✅ | ❌ (trusts whoever served `tenv_v<version>_checksums.txt`) | GitHub | None beyond the checksums file |
| `none` | ❌ | ❌ | GitHub | None |

Use `signature` unless you have a specific reason not to.

## Per-user shell init

When `tenv_setup_shell_init: true` (default), the role writes:

- A bash completion script at `~/.tenv.completion.bash` (generated via `tenv completion bash`).
- An `# ANSIBLE MANAGED BLOCK - tenv` block in the user's `~/.bashrc` that:
    - exports `TENV_ROOT`,
    - prepends `$TENV_ROOT/bin` to `PATH` (so `terraform`, `tofu`, etc. resolve to the active version), and
    - sources the completion script.

Re-run the role per user if you want to configure shell init for multiple unprivileged users on the same machine.

## Example Playbooks

Default (signature verification, configure shell init for the connecting user):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.cosign  # ensures cosign is available
    - role: mttjohnson.infra.tenv
```

Latest version, signature verification, configure shell init for a specific dev user:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.cosign
    - role: mttjohnson.infra.tenv
      vars:
        tenv_version: latest
        tenv_user: developer
        tenv_group: developer
```

Pin a specific version:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.tenv
      vars:
        tenv_version: "4.11.0"
```

Install the binary system-wide but skip all per-user shell setup (e.g. for CI images):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.tenv
      vars:
        tenv_setup_shell_init: false
```

Checksum-only verification (when cosign isn't available or reachable):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.tenv
      vars:
        tenv_verify: checksum_only
```

Remove tenv:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.tenv
      vars:
        tenv_state: absent
```

## Idempotency

`tenv version` is called first; the download/verify/install path is skipped entirely when the requested version already matches what's installed. The cache directory is created and removed within a single converge.

The shell init step is idempotent on its own: `blockinfile` matches its marker, and the completion script is rewritten only if its content differs.

## Using tenv after install

The shell init takes effect on the user's next login (or after `source ~/.bashrc`). Common operations:

```bash
tenv tf install 1.10.0          # install a Terraform version
tenv tf use 1.10.0              # set the active default
tenv tofu install latest        # install latest OpenTofu
tenv tofu use latest
tenv tf list                    # show installed Terraform versions
tenv tf list-remote             # show installable versions
```

In a project directory, dropping a `.terraform-version` or `.opentofu-version` file pins that project to a specific version automatically. Set `TENV_AUTO_INSTALL=true` to have tenv install a missing pinned version on first use.

## License

MIT
