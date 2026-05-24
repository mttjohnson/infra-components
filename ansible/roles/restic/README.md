# Restic Role

Installs [restic](https://restic.net/) from upstream GitHub releases and configures
scheduled backup, retention (forget), prune, and integrity-check jobs against a
restic repository.

## Overview

- Downloads the `restic` binary archive from [restic/restic](https://github.com/restic/restic)
  releases, verifies the SHA256 checksum against the release's `SHA256SUMS` file,
  decompresses the bzip2 archive, and installs it to `/usr/local/bin/restic`.
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

## Check mode

The role is designed to run cleanly under `--check`:
- Read-only commands (`restic version`, `restic cat config`) use
  `check_mode: false` and `changed_when: false`.
- Tasks that depend on the binary or repository existing
  (decompress, copy binary, `restic init`, enable timers) use
  `ignore_errors: "{{ ansible_check_mode }}"` so the play can be previewed end-to-end.
- A debug task reports planned repository initialization when the binary is
  not yet present.
