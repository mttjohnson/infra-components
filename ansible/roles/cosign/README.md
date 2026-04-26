# cosign

Install and manage [sigstore cosign](https://github.com/sigstore/cosign) — code signing and transparency for containers and binaries — on Debian and Ubuntu hosts via the upstream `.deb` packages, with cryptographic verification of the package itself before install.

https://docs.sigstore.dev/cosign/system_config/installation/

## Requirements
 
- Ansible 2.15+
- Debian or Ubuntu target host
- Architecture: `amd64` or `arm64` (the only architectures upstream publishes `.deb` files for)
- For `cosign_verify: bundle` (default): outbound HTTPS to GitHub releases, the Sigstore TUF root (`tuf-repo-cdn.sigstore.dev`), and the Rekor transparency log (`rekor.sigstore.dev`)

## How verification works

Installing a verification tool without verifying it is the kind of bootstrapping irony that defeats the point. This role addresses that with a two-stage approach by default:

1. Download the standalone `cosign-linux-${arch}` binary and verify its SHA256 against the published `cosign_checksums.txt`.
2. Use that bootstrap binary to run `cosign verify-blob --bundle` against the `.deb`'s `.sigstore.json` sidecar, checking the certificate chain, transparency log inclusion, and signed timestamp — pinned to the upstream sigstore/cosign release workflow's OIDC identity.
3. Only after verification succeeds is the `.deb` installed.
4. The cache directory and bootstrap binary are cleaned up afterward.

The bootstrap binary is trusted because its checksum is pinned. The `.deb` is trusted because cosign verified the full Sigstore bundle with cryptographic identity binding.

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `cosign_version` | `"3.0.6"` | Version to install. Set to `"latest"` to query the GitHub API. |
| `cosign_state` | `"present"` | `present` or `absent`. |
| `cosign_verify` | `"bundle"` | `bundle`, `checksum_only`, or `none`. See verification levels below. |
| `cosign_cert_identity_regexp` | `^https://github\.com/sigstore/cosign/\.github/workflows/.*` | Regex the signing cert identity must match. |
| `cosign_cert_oidc_issuer` | `https://token.actions.githubusercontent.com` | Required OIDC issuer for the signing identity. |
| `cosign_cache_dir` | `/var/cache/ansible-cosign` | Staging dir for downloads. Cleaned up after install. |
| `cosign_download_base_url` | GitHub releases URL | Override for mirrors / air-gapped installs. |
| `cosign_github_api_url` | GitHub API URL | Used only when `cosign_version: latest`. |

## Verification levels

| Mode | Integrity | Authenticity | Network deps | Cost |
|---|---|---|---|---|
| `bundle` (default) | ✅ | ✅ (Sigstore keyless, cert chain + Rekor + signed timestamp) | GitHub + Sigstore TUF + Rekor | ~140 MB extra download for bootstrap |
| `checksum_only` | ✅ | ❌ (trusts whoever served `cosign_checksums.txt`) | GitHub | None beyond the checksums file |
| `none` | ❌ | ❌ | GitHub | None |

Use `bundle` unless you have a specific reason not to. `checksum_only` is reasonable for ephemeral CI containers where the bootstrap download cost matters, and the network path to GitHub is already trusted. `none` exists only for completeness — don't use it.

## Security note

Versions ≤ 3.0.4 are affected by CVE-2026-22703 and CVE-2026-24122 (both patched in 3.0.5). The default of `3.0.6` is past that line.

## Example Playbooks

Default (full bundle verification):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.cosign
```

Latest version, full verification:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.cosign
      vars:
        cosign_version: latest
```

Checksum-only verification (for offline-ish environments without Sigstore reachability):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.cosign
      vars:
        cosign_verify: checksum_only
```

Pin a specific version:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.cosign
      vars:
        cosign_version: "3.0.5"
```

Remove cosign:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.cosign
      vars:
        cosign_state: absent
```
 
## Used by other tools

Found as a suggested prerequisite for tenv:
https://tofuutils.github.io/tenv/

```bash
LATEST_VERSION=$(curl https://api.github.com/repos/sigstore/cosign/releases/latest | jq -r .tag_name | tr -d "v")
curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign_${LATEST_VERSION}_amd64.deb"
sudo dpkg -i cosign_${LATEST_VERSION}_amd64.deb
```

## Idempotency
 
The role calls `cosign version --json` first and short-circuits the entire download/verify/install path if the requested version is already installed. The cache dir is created and removed within a single converge.

## Notes on verifying cosign before installing

When this was developed checking SHA256 checksum of downloaded files was included as well as an option to download a standalone binary of cosign (and verifying it's checksum) and use it to verify the .deb package to be installed.

There was also another option, but that path wasn't ventured down:
Option B — Manual sigstore bundle verification with openssl/jq (OpenBao-style)
referenced (https://openbao.org/docs/install/ and https://github.com/mttjohnson/homelab/blob/main/openbao/ansible/roles/openbao/tasks/main.yml)
Parse the .sigstore.json, extract the cert and signature, validate the cert chain against the Sigstore Fulcio root, verify the signature with openssl. Also cross-check against Rekor. This is a lot of YAML and shell, brittle to bundle format changes, and the OpenBao snippet you showed is already 100+ lines and uses the old detached format — the new bundle format with embedded transparency log proofs and signed timestamps is significantly more complex to verify by hand. I'd recommend against this for cosign specifically because the other option is much cleaner.

This may be a good TODO item to follow up with later

## License
 
MIT
 