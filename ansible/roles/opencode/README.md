# opencode

Install and manage [OpenCode](https://opencode.ai) — a terminal-based AI coding agent — for an unprivileged user on Debian/Ubuntu hosts, via the upstream `opencode-ai` npm package installed into a per-user npm prefix.

https://opencode.ai/docs
https://www.npmjs.com/package/opencode-ai
https://github.com/anomalyco/opencode/releases

## Requirements

- Ansible 2.15+
- Debian or Ubuntu target host
- Node.js and npm available system-wide (provide via the `mttjohnson.infra.nodejs` role)
- The target user (`opencode_user`) already exists with a home directory (e.g. created by `mttjohnson.infra.bot_dev`)

## Why npm (and not the install script or a raw binary)

OpenCode ships the same standalone compiled binary three ways: the `opencode-ai` npm wrapper (which pulls the correct per-platform binary as an optionalDependency), the `curl -fsSL https://opencode.ai/install | bash` script, and raw tarballs on GitHub releases. This role uses **npm** because it is the best-managed *and* the most verifiable of the three:

| Concern | npm | install.sh / GitHub tarball |
|---|---|---|
| Idempotent + `--check` safe | ✅ `community.general.npm` | ❌ `curl \| bash` always reports changed |
| Version pin / upgrade / removal | ✅ managed by npm | ⚠️ `VERSION` pin only; no upgrade/removal story |
| Platform selection (musl/baseline) | ✅ automatic via npm `os`/`cpu`/`libc` | ✅ script auto-detects |
| Integrity hash | ✅ sha512 SRI enforced on install | ❌ no checksum published for the CLI tarball |
| Authenticity signature | ✅ npm registry (Sigstore) signature | ❌ none |

**Supply-chain reality (verified against upstream):** OpenCode publishes **no** `checksums.txt` and **no** Sigstore bundle for the CLI tarballs on GitHub — only the desktop AppImage/deb/rpm get sha512 entries in the electron-builder `latest*.yml` manifests. npm is the only channel with any cryptographic material to verify against. Note that OpenCode does **not** publish npm provenance / SLSA build attestations on any channel today, so full source-to-artifact provenance (the kind the `cosign` role verifies via keyless Sigstore bundles) is not achievable here — `npm audit signatures` verifies the npm registry signature only (tamper-evidence between the registry and the host), which is the strongest assurance currently available for OpenCode.

## What the role does

1. Confirms npm is available.
2. Creates the per-user npm prefix (`~/.local/npm-global`) and bin dir, owned by the target user.
3. Sets the user's npm `prefix` so the global install stays in their home.
4. Adds the bin dir to `PATH` in the user's login profile (optional).
5. Installs `opencode-ai` globally as the target user (pinned or tracking latest).
6. Verifies the npm registry (Sigstore) signatures of the installed tree with `npm audit signatures` (optional, on by default; fails the play on a missing/invalid signature).
7. Verifies the `opencode` binary runs.

The install runs entirely as `opencode_user` — nothing is installed to system paths.

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `opencode_user` | `botdev` | Unprivileged user OpenCode is installed for and runs as. |
| `opencode_user_home` | `/home/{{ opencode_user }}` | Home dir; used for `HOME` and the npm prefix base. |
| `opencode_version` | `latest` | `latest` tracks newest; any other value pins that exact version. |
| `opencode_npm_prefix` | `{{ opencode_user_home }}/.local/npm-global` | Per-user npm global prefix. |
| `opencode_bin_dir` | `{{ opencode_npm_prefix }}/bin` | Where the `opencode` launcher is exposed. |
| `opencode_package` | `opencode-ai` | npm package name. |
| `opencode_manage_profile` | `true` | Add `opencode_bin_dir` to `PATH` in the profile file. |
| `opencode_profile_file` | `{{ opencode_user_home }}/.profile` | Profile file updated with the PATH export. |
| `opencode_verify_signatures` | `true` | Run `npm audit signatures` after install; fail on missing/invalid signature. |

## Example Playbooks

Default (install for `botdev`, track latest, verify signatures):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.opencode
```

Install for a specific user and pin a version:

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.opencode
      vars:
        opencode_user: alice
        opencode_version: "1.15.13"
```

Skip the PATH/profile management (e.g. PATH handled elsewhere):

```yaml
- hosts: all
  roles:
    - role: mttjohnson.infra.opencode
      vars:
        opencode_manage_profile: false
```

## Check mode

The role is safe under `--check`. The npm module's read-only queries predict whether an install/upgrade would happen without applying it. The signature-verification and binary-verification commands are read-only (`check_mode: false`); on a fresh host where the install was only simulated they would have nothing to act on, so their errors are ignored in check mode rather than aborting the play.

## Idempotency

`community.general.npm` reports `changed=0` on a converged system. With `opencode_version: latest` the role checks the registry each run and upgrades only when a newer version exists; with a pinned version it installs only when that version is absent.

## License

MIT
