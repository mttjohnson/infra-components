# Restic Role

Installs [restic](https://restic.net/) from upstream GitHub releases and configures
scheduled backup, retention (forget), prune, and integrity-check jobs against a
restic repository.

## Overview

- Downloads the `restic` binary archive from [restic/restic](https://github.com/restic/restic)
  releases. The release's `SHA256SUMS` file is authenticated against restic's PGP signing
  key (pinned by SHA256 and active-primary fingerprint) before any checksum from it is
  trusted; the binary archive is then verified against the authenticated checksum,
  decompressed, and installed to `/usr/local/bin/restic`.
- Resolves `latest` from the GitHub API by default; pin via `restic_version`.
- Generates a long random repository password on the remote host and stores it
  in `/etc/restic/password`. Optionally copies the password file back to the
  Ansible control node for disaster-recovery storage.
- Writes `/etc/restic/env` with `RESTIC_REPOSITORY`, `RESTIC_PASSWORD_FILE`, and
  any backend credentials defined in `restic_env`.
- Initializes the repository (`restic init`) if it does not already exist.
- Deploys five job scripts in `/etc/restic/`: `restic-backup.sh`,
  `restic-forget.sh`, `restic-prune.sh`, `restic-check.sh`,
  `restic-check-data.sh`.
- Schedules those scripts via systemd `.service`+`.timer` pairs (default) or via
  `/etc/cron.d/restic`, selected by `restic_scheduler`.

The default retention policy keeps backups **daily for a week, weekly for a
month, monthly for a year, yearly for three years**.

## Usage

```yaml
- ansible.builtin.import_role:
    name: mttjohnson.infra.restic
  tags: restic
```

### Local-disk repository (default)

Backups land in `/data/backups/` on the host. The directory is created with
mode `0700` owned by root.

### Remote repository

Set the repository URL and any backend credentials. The role does not create
the remote bucket; provision it ahead of time.

```yaml
restic_repository: "s3:s3.us-east-1.amazonaws.com/my-bucket/prod-host"
restic_env:
  AWS_ACCESS_KEY_ID: "AKIA..."
  AWS_SECRET_ACCESS_KEY: "..."
```

### Saving the password to the control node

Strongly recommended — without the password, the backups are unrecoverable.

```yaml
restic_fetch_password_to_control: true
restic_control_password_dir: "~/restic-passwords"
```

The first run generates the password on the remote host and fetches it to
`~/restic-passwords/<inventory_hostname>.password` on the control node with
mode `0600`. Subsequent runs re-fetch the existing file (idempotent).

### Switching scheduler

```yaml
restic_scheduler: cron   # default is "systemd"
```

Switching back and forth is supported: the systemd path removes any leftover
`/etc/cron.d/restic`, and the cron path stops and removes any leftover
`restic-*.timer` units.

### Failure notifications

Set a shell command and the systemd jobs gain `OnFailure=restic-backup-failure.service`,
which runs the command whenever a scheduled job exits non-zero. Useful for
healthchecks.io / Dead Man's Snitch / etc.

```yaml
restic_failure_command: "curl -fsS --retry 3 https://hc-ping.com/<uuid>/fail"
```

Not wired up in cron mode (cron has no equivalent of `OnFailure=`).

## Variables

### Installation

| Variable | Default | Description |
|---|---|---|
| `restic_version` | `latest` | Version to install, or `latest` to auto-detect |
| `restic_github_repo` | `restic/restic` | GitHub repository used to resolve releases and downloads |
| `restic_binary_install_dir` | `/usr/local/bin` | Where to install the `restic` binary |
| `restic_config_dir` | `/etc/restic` | Directory for env file, password file, scripts |
| `restic_log_dir` | `/var/log/restic` | Directory for cron-mode job logs |
| `restic_force_reinstall` | `false` | Force re-download and reinstall even if the correct version is present |

### Release signature verification

The role establishes the following trust chain before installing the binary:

1. Download the restic signing key from `restic_signing_key_url`, with the bytes
   pinned by `restic_signing_key_checksum` (defense against MITM on the key URL).
2. Walk the key's `gpg --with-colons` listing, skip revoked primaries, and assert
   the active primary fingerprint matches `restic_signing_key_fingerprint`. This
   is the operator-readable identity gate — compare against restic's published
   trust info at https://restic.readthedocs.io/en/stable/020_installation.html
3. Import the key into a dedicated GnuPG home (`restic_gnupg_home`, default
   `/etc/restic/gnupg`) so the trust scope stays restic-only.
4. Download `SHA256SUMS` and `SHA256SUMS.asc` for the target release, and run
   `gpg --verify` against the dedicated keyring. Any failure aborts the play.
5. Extract the per-file SHA256 from the now-authenticated `SHA256SUMS` and hand
   it to `get_url` for the `.bz2` download.

The key file at `restic_signing_key_url` currently bundles a legacy revoked
primary (`7BC3013F8489FADC424A07B9141138DDA3E45D66`) alongside the active one
(`CF8F18F2844575973F79D4E191A6868BD3F7A907`); the fingerprint check explicitly
ignores revoked primaries.

| Variable | Default | Description |
|---|---|---|
| `restic_verify_signature` | `true` | Authenticate the release SHA256SUMS via PGP. Strongly discouraged to disable on a backup tool. |
| `restic_signing_key_url` | `https://restic.net/gpg-key-alex.asc` | URL serving the ASCII-armored signing key |
| `restic_signing_key_checksum` | `sha256:e4f09134173fdd60ece4454d30e38459d172f3ea9a97b32a37324b4f0515a289` | SHA256 of the bytes at `restic_signing_key_url` |
| `restic_signing_key_fingerprint` | `CF8F18F2844575973F79D4E191A6868BD3F7A907` | Expected active-primary fingerprint |
| `restic_signing_key_local_path` | `/etc/restic/restic-signing-key.asc` | Where to stage the downloaded key on the target |
| `restic_gnupg_home` | `/etc/restic/gnupg` | Dedicated GnuPG homedir for the restic keyring |

Requires `gnupg` on the target host (installed by the role when signature
verification is enabled).

#### Rotating the pinned key

When restic publishes a new signing key:

1. Verify the new fingerprint on https://restic.readthedocs.io/en/stable/020_installation.html
2. Recompute the byte checksum:
   ```bash
   curl -fsSL https://restic.net/gpg-key-alex.asc | sha256sum
   ```
3. Inspect the downloaded key to confirm the active primary fingerprint:
   ```bash
   curl -fsSL https://restic.net/gpg-key-alex.asc | gpg --show-keys --with-fingerprint
   ```
4. Update `restic_signing_key_checksum` and `restic_signing_key_fingerprint` in
   the role defaults (or override in host/group vars) and re-run.

### Repository

| Variable | Default | Description |
|---|---|---|
| `restic_repository` | `/data/backups` | Repository URL (anything restic understands — local path, `s3:...`, `b2:...`, `rclone:...`, etc.) |
| `restic_manage_local_repository_dir` | `true` | Create the local repository directory on the remote host (only meaningful for filesystem paths) |
| `restic_local_repository_owner` | `root` | Owner for the local repository directory |
| `restic_local_repository_group` | `root` | Group for the local repository directory |
| `restic_local_repository_mode` | `0700` | Mode for the local repository directory |

### Password

| Variable | Default | Description |
|---|---|---|
| `restic_password_file` | `/etc/restic/password` | Path to the repository password file on the remote host |
| `restic_password_length` | `64` | Length of the generated password (only used on first run) |
| `restic_fetch_password_to_control` | `false` | Fetch the password file back to the Ansible control node |
| `restic_control_password_dir` | `./restic_passwords` | Directory on the control node where fetched passwords are stored |
| `restic_control_password_dir_mode` | `0700` | Mode for the control-node password directory |
| `restic_control_password_file_mode` | `0600` | Mode for fetched password files |

> **Important:** the password is generated once and never rotated by the role.
> Losing it makes the backups unrecoverable. Enable `restic_fetch_password_to_control`
> or arrange another secure off-host copy.

### Backend credentials / environment

| Variable | Default | Description |
|---|---|---|
| `restic_env` | `{}` | Dict of extra environment variables placed in `/etc/restic/env` (backend creds, `RESTIC_CACHE_DIR`, etc.) |

### Backup contents

| Variable | Default | Description |
|---|---|---|
| `restic_backup_paths` | `[/etc, /home, /root, /var/log]` | Paths to back up |
| `restic_backup_excludes` | `[]` | Exclude patterns (written to `/etc/restic/excludes`, passed via `--exclude-file`) |
| `restic_backup_exclude_files` | `[]` | Additional exclude files passed via `--exclude-file` |
| `restic_backup_exclude_caches` | `true` | Skip directories tagged with `CACHEDIR.TAG` |
| `restic_backup_one_file_system` | `false` | Don't cross filesystem boundaries |
| `restic_backup_tags` | `[scheduled]` | Tags applied to each snapshot |
| `restic_backup_extra_args` | `[]` | Additional flags appended to the `restic backup` command |

### Retention

| Variable | Default | Description |
|---|---|---|
| `restic_keep_daily` | `7` | Daily snapshots to keep |
| `restic_keep_weekly` | `4` | Weekly snapshots to keep |
| `restic_keep_monthly` | `12` | Monthly snapshots to keep |
| `restic_keep_yearly` | `3` | Yearly snapshots to keep |
| `restic_forget_extra_args` | `[]` | Additional flags appended to the `restic forget` command |

### Check

| Variable | Default | Description |
|---|---|---|
| `restic_check_extra_args` | `[]` | Additional flags for the metadata-only weekly check |
| `restic_check_data_subset` | `10%` | `--read-data-subset` value for the monthly data check |
| `restic_check_data_extra_args` | `[]` | Additional flags for the monthly data check |

### Scheduler

| Variable | Default | Description |
|---|---|---|
| `restic_scheduler` | `systemd` | `systemd` or `cron` |
| `restic_persistent_timers` | `true` | `Persistent=true` on each timer (catch up missed runs after downtime) |
| `restic_randomized_delay` | `30min` | `RandomizedDelaySec` on each timer; set `""` to disable |

#### Systemd schedules (OnCalendar)

| Variable | Default |
|---|---|
| `restic_backup_schedule` | `*-*-* 02:00:00` |
| `restic_forget_schedule` | `*-*-* 03:30:00` |
| `restic_prune_schedule` | `Sun *-*-* 04:00:00` |
| `restic_check_schedule` | `Sat *-*-* 05:00:00` |
| `restic_check_data_schedule` | `Sun *-*-01..07 06:00:00` (first Sunday of month) |

#### Cron schedules

Each is a `{minute, hour, day, month, weekday}` dict. Defaults mirror the systemd
schedules above. See `defaults/main.yml`.

| Variable |
|---|
| `restic_backup_cron` |
| `restic_forget_cron` |
| `restic_prune_cron` |
| `restic_check_cron` |
| `restic_check_data_cron` |
| `restic_cron_user` (default `root`) |
| `restic_cron_package` (default `cron`; use `cronie` on RHEL-family) |

### Failure notification

| Variable | Default | Description |
|---|---|---|
| `restic_failure_command` | `""` | Shell command run by the OnFailure unit when a scheduled job fails (systemd mode only) |

## What gets installed

- **Binary**: `restic` in `/usr/local/bin/`
- **Config**: `/etc/restic/` (mode `0750`, root-owned) containing:
  - `env` — `RESTIC_REPOSITORY`, `RESTIC_PASSWORD_FILE`, backend credentials (mode `0600`)
  - `password` — generated repository password (mode `0600`)
  - `excludes` — exclude patterns (only when `restic_backup_excludes` is non-empty)
  - `restic-backup.sh`, `restic-forget.sh`, `restic-prune.sh`, `restic-check.sh`, `restic-check-data.sh`
- **Logs (cron mode only)**: `/var/log/restic/<job>.log`
- **Repository**: initialized via `restic init` on first run
- **Schedule**:
  - systemd: `restic-{backup,forget,prune,check,check-data}.{service,timer}` in `/etc/systemd/system/`
  - cron: `/etc/cron.d/restic`

## Operator runbook (BACKUPS.md)

The role renders a host-tailored quick reference into `/root/BACKUPS.md`
(path configurable via `restic_backups_md_path`, deployment toggled by
`restic_deploy_backups_md`, default `true`). It is refreshed on every
apply so it never drifts from the live configuration.

The runbook covers the operations an operator actually needs in the
moment — with the host's real repository URL, paths, schedules, and
retention numbers baked in instead of placeholder examples:

- Sourcing `/etc/restic/env` so plain `restic` commands work
- Listing scheduled timers / cron entries and tailing recent logs
- Triggering an ad-hoc backup
- Inspecting snapshots (`snapshots`, `ls`, `find`, `diff`, `stats`)
- Verifying integrity (`check`, `check --read-data-subset`)
- **Restoring** — always starting with `restic restore ... --dry-run --verbose=2`
  for a preview, then a sandboxed restore to a fresh directory, with
  recipes for single-file restores, streaming `dump`, and fuse mount
- A one-shot end-to-end test-restore recipe so operators can routinely
  prove the backups actually work, not just that they ran
- Previewing the retention policy (`restic forget --dry-run`) before
  the scheduled forget job deletes anything
- Troubleshooting stale locks and lost-password recovery
- A "where things live" path reference for the host

Authoritative restic documentation: https://restic.readthedocs.io/

| Variable | Default | Description |
|---|---|---|
| `restic_deploy_backups_md` | `true` | Render BACKUPS.md to root's home |
| `restic_backups_md_path` | `/root/BACKUPS.md` | Where to place the runbook (mode 0600 root) |

## Check mode

The role is designed to run cleanly under `--check`:
- Read-only commands (`restic version`, `restic cat config`) use
  `check_mode: false` and `changed_when: false`.
- Tasks that depend on the binary or repository existing
  (decompress, copy binary, `restic init`, enable timers) use
  `ignore_errors: "{{ ansible_check_mode }}"` so the play can be previewed end-to-end.
- A debug task reports planned repository initialization when the binary is
  not yet present.
