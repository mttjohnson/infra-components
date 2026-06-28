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

Strongly recommended — without the password, the backups are unrecoverable, and
the control-node copy is what lets the role reseed (and therefore recover) a
rebuilt host. See [RECOVERY.md](RECOVERY.md) for the full design.

```yaml
restic_fetch_password_to_control: true
restic_control_dir: "~/restic-state"
```

The first run generates the password on the remote host and fetches it to
`~/restic-state/<inventory_hostname>.password` on the control node with mode
`0600`. The fetch is **write-once**: once a control-node copy exists it is never
overwritten, so a good key cannot be clobbered by a later run.

**Password establishment order** (see [RECOVERY.md](RECOVERY.md)):

1. If the host already has `/etc/restic/password`, keep it.
2. Else if the control node holds a copy for this host and
   `restic_seed_password_from_control` is true (default), **reseed the host from
   it** — this is the rebuild / disaster-recovery path, reusing the existing key
   so existing backups remain readable.
3. Else generate a fresh password (genuine first deployment).

The control-node state directory is flat, with the host name as a filename
**prefix** (no nested subdirectories) so a clustered implementation can keep
several instances' state side by side. Only `<prefix>.password` is secret; the
other state files written by the recovery design (repo-id pins, volume sentinel
id) are non-secret and safe to commit alongside the implementation config.

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

### Secondary repository (off-box copy)

Maintain a second, independent repository kept in sync with the primary via
`restic copy` — e.g. replicate a local-disk primary to an NFS-backed NAS so a
copy survives loss of the host's disk. Disabled by default; set
`restic_secondary_repository` to enable.

```yaml
restic_repository: /data/backups          # primary (local disk)
restic_secondary_repository: /backups      # secondary (e.g. NFS mount)
```

When enabled the role:

- Initializes the secondary on first detection with
  `restic init --copy-chunker-params --from-repo <primary>` so deduplication
  parameters match and copies stay efficient. Existing deployments pick the
  secondary up automatically the next time the role runs.
- Deploys a `restic-copy` job (primary → secondary) plus
  `restic-{forget,prune,check,check-data}-secondary` jobs, so the copy gets the
  same retention policy and integrity verification as the primary. The
  secondary jobs reuse the primary job templates against a second env file
  (`env.secondary`), so there is no second source of truth for retention/check
  settings.
- By default **shares the primary password** (`restic_secondary_password_file`
  defaults to `restic_password_file`); the copy job passes it explicitly as
  `--from-password-file` so a shared password never triggers an interactive
  prompt.

The **primary stays the canonical `RESTIC_REPOSITORY`** for ad-hoc operator
commands and the `restic_restore` role. To operate on the secondary, source
`env.secondary` (documented in the rendered BACKUPS.md). Both repos are
independent and restorable on their own.

#### NFS all_squash and the mountpoint guard

If the secondary is an NFS export that squashes all users to one identity,
every repository file on it will be owned by that identity and `chown` is
rejected. This is fine: restic does not rely on repository file ownership, and
the data is encrypted, so confidentiality comes from the password (which never
leaves the host's local disk), not filesystem permissions. Accordingly the role
does **not** enforce owner/group/mode on the secondary directory
(`restic_manage_secondary_repository_dir` defaults to `false`; the ownership
vars default to unset).

Because an unmounted NFS path would silently leave the "off-box" copy on the
instance's local disk, the role refuses to initialize — and the scheduled
secondary jobs refuse to run — unless the secondary path is a mountpoint. Set
`restic_secondary_require_mountpoint: false` for secondaries that are not local
mounts (e.g. an `s3:`/`b2:`/`rclone:` URL).

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
| `restic_require_mountpoint` | `false` | Refuse to init / run primary jobs unless the local repository path is a mountpoint (guards against writing backups to the root disk if the volume failed to mount). See [RECOVERY.md](RECOVERY.md). |
| `restic_manage_local_repository_dir` | `true` | Create the local repository directory on the remote host (only meaningful for filesystem paths) |
| `restic_local_repository_owner` | `root` | Owner for the local repository directory |
| `restic_local_repository_group` | `root` | Group for the local repository directory |
| `restic_local_repository_mode` | `0700` | Mode for the local repository directory |

### Password

| Variable | Default | Description |
|---|---|---|
| `restic_password_file` | `/etc/restic/password` | Path to the repository password file on the remote host |
| `restic_password_length` | `64` | Length of the generated password (only used when generating a fresh password) |
| `restic_seed_password_from_control` | `true` | Reseed the host password from the control-node copy when the host has none (rebuild/recovery) instead of generating a new one |
| `restic_fetch_password_to_control` | `false` | Fetch the password file back to the Ansible control node (write-once — never overwrites an existing copy) |
| `restic_control_dir` | `./restic-state` | Flat directory on the control node for all per-host restic state (host name is a filename prefix) |
| `restic_control_state_prefix` | `{{ inventory_hostname }}` | Filename prefix for this host's control-node state files |
| `restic_control_password_file` | `{{ restic_control_dir }}/{{ restic_control_state_prefix }}.password` | Derived path to this host's control-node password copy |
| `restic_control_dir_mode` | `0700` | Mode for the control-node state directory |
| `restic_control_password_file_mode` | `0600` | Mode for the (secret) fetched password file |
| `restic_control_state_file_mode` | `0644` | Mode for non-secret state files (repo-id pins, sentinel ids) |

> **Important:** the password is generated once and never rotated by the role.
> Losing it makes the backups unrecoverable. Enable `restic_fetch_password_to_control`
> (and replicate the control-node copy off-host) — it is also what lets the role
> reseed and recover a rebuilt host. See [RECOVERY.md](RECOVERY.md).

### Recovery safety

The role and its companion `restic_restore` cooperate to ensure a rebuild can
never silently turn into a fresh start that loses data. The full design is in
[RECOVERY.md](RECOVERY.md); the short version:

- **Fail-early when a repository exists but no password is available.** A restic
  repository's `config` is detectable without the password, so the role refuses
  to generate a new key or initialize over an existing repository when no key
  (host or control node) can read it — recover the control-node key and re-run.
- **Fail-closed blessed marker.** The state-changing jobs (backup / copy /
  forget / prune) refuse to run unless `restic_blessed_marker_path` exists. The
  `restic_restore` role writes it only when the host is in a known-good state
  and removes it otherwise, so a failed recovery locks out both new backups and
  retention until resolved.
- **`restic_force_fresh_start`** — operator override to proceed as a fresh
  deployment despite recovery signals. Takes effect only when it equals the
  target `inventory_hostname`; does not bypass the "repository exists but no
  key" stop and does not re-key.

| Variable | Default | Description |
|---|---|---|
| `restic_blessed_marker_path` | `{{ restic_config_dir }}/blessed` | Fail-closed gate marker the state-changing jobs require; written/removed by `restic_restore` (its `restic_restore_blessed_marker_path` must match) |
| `restic_force_fresh_start` | `""` | Set to the target `inventory_hostname` to proceed fresh despite recovery signals |

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

### Secondary repository

| Variable | Default | Description |
|---|---|---|
| `restic_secondary_repository` | `""` | Secondary repository URL/path. Empty disables the whole feature. |
| `restic_secondary_password_file` | `{{ restic_password_file }}` | Password file for the secondary (defaults to sharing the primary password) |
| `restic_manage_secondary_repository_dir` | `false` | Create the secondary directory on the host (rarely needed — the mount usually exists and `restic init` populates it) |
| `restic_secondary_repository_owner` | _(unset)_ | Owner for the secondary directory; unset so an all_squash export isn't fought |
| `restic_secondary_repository_group` | _(unset)_ | Group for the secondary directory; unset by default |
| `restic_secondary_repository_mode` | _(unset)_ | Mode for the secondary directory; unset by default |
| `restic_secondary_require_mountpoint` | `true` | Refuse to init / run secondary jobs unless the secondary path is a mountpoint (guards against writing to local disk) |
| `restic_secondary_copy_extra_args` | `[]` | Extra args appended to `restic copy` (e.g. `--tag`/`--host` filters) |

#### Secondary systemd schedules (OnCalendar)

| Variable | Default |
|---|---|
| `restic_secondary_copy_schedule` | `*-*-* 03:00:00` |
| `restic_secondary_forget_schedule` | `*-*-* 03:45:00` |
| `restic_secondary_prune_schedule` | `Sun *-*-* 04:30:00` |
| `restic_secondary_check_schedule` | `Sat *-*-* 05:30:00` |
| `restic_secondary_check_data_schedule` | `Sun *-*-01..07 06:30:00` |

#### Secondary cron schedules

Each is a `{minute, hour, day, month, weekday}` dict mirroring the systemd
schedules above: `restic_secondary_copy_cron`, `restic_secondary_forget_cron`,
`restic_secondary_prune_cron`, `restic_secondary_check_cron`,
`restic_secondary_check_data_cron`. See `defaults/main.yml`.

## What gets installed

- **Binary**: `restic` in `/usr/local/bin/`
- **Config**: `/etc/restic/` (mode `0750`, root-owned) containing:
  - `env` — `RESTIC_REPOSITORY`, `RESTIC_PASSWORD_FILE`, backend credentials (mode `0600`)
  - `password` — generated repository password (mode `0600`)
  - `excludes` — exclude patterns (only when `restic_backup_excludes` is non-empty)
  - `restic-backup.sh`, `restic-forget.sh`, `restic-prune.sh`, `restic-check.sh`, `restic-check-data.sh`
  - when a secondary repository is configured: `env.secondary`, `restic-copy.sh`,
    and `restic-{forget,prune,check,check-data}-secondary.sh`
- **Logs (cron mode only)**: `/var/log/restic/<job>.log`
- **Repository**: initialized via `restic init` on first run (secondary, when
  configured, via `restic init --copy-chunker-params --from-repo <primary>`)
- **Schedule**:
  - systemd: `restic-{backup,forget,prune,check,check-data}.{service,timer}` in `/etc/systemd/system/`
    (plus `restic-{copy,forget-secondary,prune-secondary,check-secondary,check-data-secondary}.{service,timer}`
    when a secondary repository is configured)
  - cron: `/etc/cron.d/restic`

## Operator runbook (BACKUPS.md)

The role renders a host-tailored quick reference into `/root/BACKUPS.md`
(path configurable via `restic_backups_md_path`, deployment toggled by
`restic_deploy_backups_md`, default `true`). It is refreshed on every
apply so it never drifts from the live configuration.

The on-host file is intentionally a **terse copy-paste reference** — it
assumes whoever opens it in a terminal is mid-incident and wants a command,
not a tutorial. It carries only the host's real repository URL, paths,
schedules, and retention numbers plus the one-liners to check status, verify,
and restore, and it links back to this README (`restic_readme_url`) for the
long-form material below.

This section is that long form: the same operations with the explanation the
on-host file omits. Every command assumes `/etc/restic/env` is sourced first
(`set -a; . /etc/restic/env; set +a`) so plain `restic` works.

### Inspect what got backed up

```bash
restic snapshots                       # all snapshots
restic snapshots --latest 3            # three most recent
restic snapshots --host <hostname>     # filter to one host (multi-host repos)
restic stats latest                    # size & file count of the latest snapshot
restic stats --mode raw-data           # raw repo size

restic ls latest                       # full file tree in latest snapshot
restic ls --long latest /etc           # one subtree, with sizes / mtimes
restic find PATTERN                    # search snapshots by filename
restic diff <old-id> <new-id>          # what changed between two snapshots
```

Snapshot IDs are the short hex strings from `restic snapshots`; the literal
word `latest` always aliases the newest snapshot.

### Verify integrity

```bash
restic check                              # fast — metadata only
restic check --read-data-subset=10%       # slower — catches on-disk bit-rot
```

The role already runs the metadata check on the `restic-check` schedule and the
data-subset check on `restic-check-data`. Ad-hoc runs matter most **after** a
suspected hardware issue or **before** relying on a snapshot to recover
something important.

### Restore

Always preview first — it prints exactly which files would be written, and
where, with no changes to disk:

```bash
restic restore latest --target /tmp/restore-preview --dry-run --verbose=2
```

Narrow with `--include /path/under/snapshot` or `--exclude PATTERN`. Then
restore into a **fresh** directory (never on top of live data), inspect, and
copy out only what you need:

```bash
restic restore latest --target /var/restore                          # whole snapshot
restic restore latest --target /var/restore --include /etc/file.conf # lands at /var/restore/etc/file.conf
restic restore <snapshot-id> --target /var/restore                   # a specific, non-latest snapshot
restic dump latest /etc/file.conf > /tmp/file.conf                   # stream one file, no temp tree
```

Browse the repo like a filesystem (read-only):

```bash
apt-get install -y fuse                            # if not already present
mkdir -p /mnt/restic && restic mount /mnt/restic & # mounts /mnt/restic/snapshots/
ls /mnt/restic/snapshots/latest/
fusermount -u /mnt/restic                          # when done
```

### Prove a restore actually works (run periodically)

A backup that has never been restored might not work. This round-trip touches
no live files; vary the file each time to exercise more of the restore path:

```bash
TMP=$(mktemp -d /tmp/restic-test.XXXXXX)
restic restore latest --target "$TMP" --include /etc/hostname
diff /etc/hostname "$TMP/etc/hostname" && echo "RESTORE OK"
rm -rf "$TMP"
```

### Preview retention before it deletes anything

The scheduled `restic-forget` job applies the retention policy. To see what it
*would* remove without doing it:

```bash
restic forget --dry-run \
  --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --keep-yearly 3
```

(Substitute the host's actual `restic_keep_*` values — the rendered BACKUPS.md
bakes them in.)

### Operating the secondary repository

When a secondary is configured, every command above works against it once you
source its env file instead of the primary's:

```bash
set -a; . /etc/restic/env.secondary; set +a   # now `restic snapshots` lists the SECONDARY
```

The two repos are fully independent — each has its own index and is restorable
on its own. The `restic-copy` job replicates primary → secondary on schedule,
and the secondary gets matching forget / prune / check jobs. See
[Secondary repository](#secondary-repository-off-box-copy) for the NFS
all-squash ownership behaviour and the mountpoint guard.

### Troubleshooting

**`repository is already locked`** — a previous restic process left a stale
lock (reboot mid-run, SIGKILL). Confirm nothing live is holding it, then clear:

```bash
systemctl list-units 'restic-*' --state=running    # systemd mode: should be empty
restic list locks
restic unlock                                       # only if no live process
```

**`Fatal: wrong password ...`** — confirm the password file is intact
(`sha256sum /etc/restic/password`). If the password is lost the repository is
**unrecoverable** — there is no maintainer backdoor by design. Enable
`restic_fetch_password_to_control` (above) so a copy lands on the control node.

**See exactly what the scheduled jobs run** — the job scripts are plain bash and
safe to read: `cat /etc/restic/restic-backup.sh`.

Authoritative restic documentation: https://restic.readthedocs.io/

| Variable | Default | Description |
|---|---|---|
| `restic_deploy_backups_md` | `true` | Render BACKUPS.md to root's home |
| `restic_backups_md_path` | `/root/BACKUPS.md` | Where to place the runbook (mode 0600 root) |
| `restic_readme_url` | _(this README on GitHub)_ | URL the rendered BACKUPS.md links to for the full guide |

## Check mode

The role is designed to run cleanly under `--check`:
- Read-only commands (`restic version`, `restic cat config`) use
  `check_mode: false` and `changed_when: false`.
- Tasks that depend on the binary or repository existing
  (decompress, copy binary, `restic init`, enable timers) use
  `ignore_errors: "{{ ansible_check_mode }}"` so the play can be previewed end-to-end.
- A debug task reports planned repository initialization when the binary is
  not yet present.
