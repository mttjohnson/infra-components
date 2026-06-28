# restic_restore Role

Restores one or more "state bundles" from a restic repository to their target
paths on a host, so service roles further along in the play can converge from
already-populated state instead of building everything from scratch.

This role is **state-restore plumbing**: generic, service-agnostic, idempotent,
and tolerant of fresh deployments that have no prior backup. It does not
install restic, configure the repository, or know which backend the repo
lives on. Those concerns belong to the [`restic`](../restic/) role, which must
run earlier in the play.

## Interface contract with the `restic` role

`restic_restore` consumes two things the `restic` role establishes:

1. The restic binary at `restic_restore_binary_path` (default
   `/usr/local/bin/restic`).
2. At least one source env file from `restic_restore_sources` (by default the
   primary `/etc/restic/env`, and the secondary `/etc/restic/env.secondary` when
   present), each containing at minimum `RESTIC_REPOSITORY` and
   `RESTIC_PASSWORD_FILE` plus any backend credentials.

If the binary or *every* source env file is missing the role fails with a clear
message (gate failure). There is no flag to make this permissive — if you want
to run `restic_restore`, the `restic` role must have run first.

No repo URL, password, or backend credentials appear in this role's variables.
Switching the backend is the `restic` role's concern; `restic_restore` is
unaffected.

## Orchestration pattern

```yaml
- hosts: web01
  roles:
    - role: mttjohnson.infra.restic
      # installs /usr/local/bin/restic, writes /etc/restic/env,
      # configures the repository at /data/backups

    - role: mttjohnson.infra.restic_restore
      vars:
        restic_restore_bundles:
          - name: letsencrypt
            paths:
              - /etc/letsencrypt
            snapshot_selector:
              tags: [letsencrypt]
              host: web01
              paths: [/etc]   # see "Snapshot-selector gotcha" below
            owner: root
            group: root
            dir_mode: "0755"
            file_mode: "0600"

    - role: mttjohnson.infra.certbot
      # finds /etc/letsencrypt already populated with a valid cert
      # and no-ops the ACME request
```

The design assumes service roles further down the play are tolerant of
pre-existing state on first run. Update individual service roles to no-op
when their target state is already present; `restic_restore` only puts the
bytes on disk.

## Failure modes

| Scenario                                | Default behaviour | Knob |
| --------------------------------------- | ----------------- | ---- |
| Restic binary or env file missing       | **Fail**          | (none — always fails in a real run) |
| Repository unreachable (mount/network)  | **Skip all bundles, warn** | `restic_restore_fail_on_missing_repo: true` to fail |
| No snapshot matches a non-critical bundle | **Skip that bundle, warn** | Per-bundle `restore_required: true` to make it recovery-critical |
| No snapshot for a `restore_required` bundle, **recovery expected** | **Fail; host not blessed** | `restic_force_fresh_start=<hostname>` to start fresh |
| No snapshot for a `restore_required` bundle, **first deploy** | **Skip; converge fresh** | (recovery not expected — nothing to restore) |
| Restic exits non-zero during restore    | **Fail**          | (none — always fails loudly) |
| Permission error writing target / chmod / chown | **Fail**  | (none — always fails loudly) |

The skip-by-default flags make the permissive case the default: restore if
available, continue if not. `restore_required: true` marks a bundle
**recovery-critical** — its failure is conditioned on `recovery_expected` (see
below), so it is safe to set even on a fresh install.

## Recovery enforcement & blessed marker

This role is the sole authority for the **blessed marker**
(`restic_restore_blessed_marker_path`, default `/etc/restic/blessed`) that the
`restic` role's state-changing jobs (backup / copy / forget / prune) gate on.
The full cross-role design is in the [`restic` role's RECOVERY.md](../restic/RECOVERY.md).

`recovery_expected` is computed by the `restic` role (any of: a host or
control-node password, or an existing primary/secondary repository) and shared
via a host fact. After the bundle loop this role:

- **Writes the marker** when the host is in a known-good state — a genuine first
  deploy (`recovery_expected = false`), a confirmed recovery (every
  `restore_required` bundle restored or already populated), or an explicit
  `restic_force_fresh_start=<inventory_hostname>` override.
- **Removes the marker and fails loudly** when recovery is expected but one or
  more `restore_required` bundles have no matching snapshot. The marker is
  removed *before* the failure, so backups and retention stay locked
  (fail-closed) until the situation is resolved — preventing both polluting
  backups and retention eroding the surviving snapshots.

Under `--check` the failure is downgraded to a "would fail" warning, and the
bless/remove decision is still shown.

## Check mode

`--check` is the operator's pre-run review. The role makes it report the
real action set:

- **Every task in the real-run path still appears** in the check-mode output.
  The per-bundle plan summary debug task always runs and reports the selector
  decision, idempotency state, and the action that would be taken
  (`RESTORE` / `SKIP` / `FAIL`).
- **Gate failures and probe failures are downgraded to warnings** under
  `--check`, so the inspection continues to the end. The warning is part of
  the output: the operator sees "would fail at gate" AND the full per-bundle
  plan that would follow on a healthy host.
- **Read-only restic queries** (`snapshots --json`, `snapshots --last 1`)
  are marked `check_mode: false` and `changed_when: false`. They are safe:
  they don't mutate the host or the repo.
- **State-changing tasks** (the restic `restore`, the chmod / chown commands,
  the marker write, the hooks) are not marked `check_mode: false`. They
  appear in `--check` output as skipped tasks with descriptive names like
  `[letsencrypt] Restore snapshot abc12345`, paired with the plan summary
  above them.

A check-mode pass plus your configured failure flags is a reliable basis for
approving the real run.

## Idempotency

The source of truth for "already restored" is **on-disk state**, not the
marker file. The role skips the restore for a bundle when **all** of its
effective target paths exist and are non-empty. Consequences:

- If the operator wipes a target directory, the next run re-restores it
  (the marker file is ignored).
- If a target was populated by a non-restore source (a hand bootstrap,
  another tool), the role leaves it alone and **does not** write a marker
  — we didn't restore it, so we don't claim we did.
- The marker file at `/var/lib/restic_restore/<bundle_name>.json` is
  diagnostic only: it records which snapshot, when restored, and the
  selector that picked it. Useful for post-hoc forensics; never gates
  behaviour.

For a bundle with multiple `paths`, all must be populated for the skip to
trigger — if any is empty or missing, the entire bundle is restored
atomically from one snapshot.

## Multi-source restore

Each bundle is restored from whichever **source** holds the newest matching
snapshot. `restic_restore_sources` is an ordered list of `{name, env_file}`;
the default is the primary env plus the secondary env, and any source whose env
file is absent on the host is dropped (so a host with no secondary just uses the
primary). For each bundle the role queries every reachable source, then picks
the snapshot with the newest `time` across all of them — **ties resolve to the
earlier source** (the primary). Because `restic copy` preserves a snapshot's
`time`, a steady-state host (both repos hold the same snapshots) restores from
the local primary, while a rebuilt host whose local repo is gone restores
automatically from the off-box secondary. No manual env swap is needed.

The per-source queries are skipped for a bundle whose targets are already
populated, so a converged host doesn't pay the (possibly off-box) query cost.

After an actual restore, the role re-probes the target paths and **fails if any
is still empty** — catching the case where a declared path is absent from the
chosen snapshot (`restic restore --include` silently restores nothing for a
missing path). This is a real-run guard (the restore is skipped under `--check`).

## Snapshot selection

Each bundle's `snapshot_selector` filters each source's snapshot list. Within a
source the role picks the most recent matching snapshot (`sort by time | last`);
across sources it then picks the newest (ties → primary), and restores the
bundle's `paths` from that single snapshot.

Selector subkeys:

- `tags` — list of tags every selected snapshot must carry (AND semantics).
- `host` — hostname the snapshot was taken from.
- `paths` — list of snapshot-metadata paths to filter on. Defaults to the
  bundle's `paths` if omitted (but read the gotcha below).
- `latest` — always treated as `true`; the role only restores the most
  recent matching snapshot.

### Snapshot-selector `paths` gotcha

`restic snapshots --path X` matches snapshots whose **top-level backup
paths include X exactly**, not snapshots whose contents contain X. If your
restic backup is `restic backup /etc /home /root`, the snapshot's metadata
`paths` list is `["/etc", "/home", "/root"]`.

If you want to restore `/etc/letsencrypt` from such a snapshot, you must
filter with `snapshot_selector.paths: [/etc]`, **not** `[/etc/letsencrypt]`.
The default of "use the bundle's `paths`" works only when the backup root
is the same path you're restoring (e.g. `restic backup /etc/letsencrypt`).
When in doubt, set `snapshot_selector.paths` explicitly and verify with
`restic snapshots --tag <tag> --host <host> --path <path>`.

## Variables

| Variable                              | Default                        | Description |
| ------------------------------------- | ------------------------------ | ----------- |
| `restic_restore_binary_path`          | `/usr/local/bin/restic`        | Path to the restic binary established by the `restic` role |
| `restic_restore_env_file`             | `/etc/restic/env`              | Primary source env file (also the `primary` entry in `restic_restore_sources`) |
| `restic_restore_secondary_env_file`   | `/etc/restic/env.secondary`    | Secondary source env file (the `secondary` entry); dropped automatically when absent |
| `restic_restore_sources`              | `[primary, secondary]`         | Ordered restore sources — each bundle restores from whichever holds the newest matching snapshot (see below) |
| `restic_restore_fail_on_missing_repo` | `false`                        | Fail the play (vs. warn-and-skip) when **no** source is reachable |
| `restic_restore_marker_dir`           | `/var/lib/restic_restore`      | Where per-bundle diagnostic markers are written |
| `restic_restore_blessed_marker_path`  | `/etc/restic/blessed`          | Fail-closed gate marker written/removed by this role; MUST match the `restic` role's `restic_blessed_marker_path` |
| `restic_restore_blessed_marker_mode`  | `0640`                         | Mode for the blessed marker file |
| `restic_restore_recovery_expected_default` | `false`                   | Value assumed for `recovery_expected` when this role runs standalone (no `restic` role in the play → no enforcement) |
| `restic_restore_bundles`              | `[]`                           | List of bundles to restore — see below |

### Bundle schema

```yaml
restic_restore_bundles:
  - name: letsencrypt              # required, used in task names & marker filename
    paths:                          # required, ≥1 absolute path
      - /etc/letsencrypt
    snapshot_selector:              # filter for picking the snapshot
      tags: [letsencrypt]           # optional
      host: web01                   # optional
      paths: [/etc]                 # optional, defaults to bundle `paths`
      latest: true                  # optional, default true (only true is honored)
    target: ""                      # optional, relocate restore under this prefix
    restore_required: false         # optional, default false; true = fail if no match
    owner: root                     # optional, recursive chown user
    group: root                     # optional, recursive chown group
    dir_mode: "0755"                # optional, recursive dir mode
    file_mode: "0600"               # optional, recursive file mode
    pre_restore_hook: ""            # optional, shell command run before restore
    post_restore_hook: ""           # optional, shell command run after restore
```

#### Bundle fields in detail

- **`name`** — used in Ansible task names, in plan-summary output, and in
  the marker filename. Stick to filesystem-safe identifiers.
- **`paths`** — restore these from the selected snapshot via `--include`.
  All paths in a bundle come from one snapshot, atomically.
- **`snapshot_selector`** — see "Snapshot selection" above.
- **`target`** — optional. When unset, files land at their original paths
  (`--target /`). When set, the effective target for each entry in `paths`
  is `<target><path>` — e.g. `target: /restore` + `paths: [/etc/letsencrypt]`
  writes to `/restore/etc/letsencrypt`. The idempotency check inspects the
  effective target paths, not the original paths.
- **`restore_required`** — default `false`. Marks the bundle
  **recovery-critical**. Enforcement is conditioned on `recovery_expected`: when
  recovery is expected (a prior backup exists) and no snapshot matches, the play
  fails and the host is left unblessed; on a genuine first deploy it is simply
  skipped (you cannot restore what was never backed up). Set `true` for state
  that has no sensible fresh-system default; leave `false` for state a
  downstream role can rebuild. See the [RECOVERY.md](../restic/RECOVERY.md)
  design.
- **`owner`, `group`** — recursive ownership enforcement post-restore.
  Either may be set independently; the chown form is `owner:group`,
  `owner`, or `:group` depending on which are defined.
- **`dir_mode`, `file_mode`** — separate modes for directories and regular
  files, applied recursively post-restore. Modes are enforced via two-phase
  detect-then-apply (`find -not -perm` first, then `find -exec chmod`) so
  the chmod task only runs and reports `changed` when drift exists.
- **`pre_restore_hook`, `post_restore_hook`** — shell commands run with
  `/bin/bash` before/after the restore. The restic env file is **not**
  pre-sourced; the hook handles its own environment. Non-zero exit fails
  the play.

## Plan summary output

Every bundle emits a debug block during the play. Example:

```
bundle:    letsencrypt
paths:     ['/etc/letsencrypt']
target:    (original)
selector:
  tags:    ['letsencrypt']
  host:    web01
  paths:   ['/etc']
restore_required: false
gate:      ok
repo:      reachable
snapshot:  abc12345 @ 2026-05-24T02:00:13Z (host=web01, tags=['letsencrypt', 'scheduled'])
targets_populated: false
action: RESTORE +dir_mode=0755 +file_mode=0600 +chown=root:root
```

This is the same in check mode and real runs. In check mode the lines below
it (`Restore snapshot ...`, `Enforce dir_mode ...`, etc.) appear as
skipped Ansible tasks — the plan summary makes the intent explicit.

## Marker file format

After a real restore, `/var/lib/restic_restore/<bundle_name>.json` records:

```json
{
  "bundle": "letsencrypt",
  "snapshot_id": "abc12345...full-id...",
  "snapshot_short_id": "abc12345",
  "snapshot_time": "2026-05-24T02:00:13.123456789Z",
  "snapshot_hostname": "web01",
  "snapshot_tags": ["letsencrypt", "scheduled"],
  "restored_at": "2026-05-25T14:22:01Z",
  "paths": ["/etc/letsencrypt"],
  "target": "",
  "selector": {
    "tags": ["letsencrypt"],
    "host": "web01",
    "paths": ["/etc"]
  }
}
```

Diagnostic only. Read it during incident response; do not depend on it
programmatically — the on-disk state of the target paths is the source of
truth.

## Scenarios

The role is designed to handle each of these correctly without operator
intervention:

| Scenario                                            | Behaviour |
| --------------------------------------------------- | --------- |
| Fresh rebuild, `/data` reattached with backups      | Gate ok, probe ok, restore from latest matching snapshot (local primary). |
| Rebuild, local `/data/backups` gone, NAS intact     | Primary empty/absent, secondary holds the snapshots → restore automatically from the NAS. |
| Fresh deployment, no source disk yet                | Gate ok, no source reachable, all bundles skipped (with `fail_on_missing_repo: false`). |
| Fresh deployment, `/data` mounted but no backups    | Gate ok, probe ok, no snapshots match, bundle skipped (with `restore_required: false`). |
| Re-run against converged system                     | Gate ok, probe ok, targets non-empty, bundle skipped via on-disk idempotency; host (re)blessed. |
| Check mode against host without restic installed    | Gate failure logged as warning, per-bundle plan summary still emitted. |
| Bundle the operator wants to enforce strictly       | Per-bundle `restore_required: true` causes failure when no snapshot matches **and recovery is expected**. |
| Rebuild, recovery expected, critical bundle has no snapshot | Marker removed, play fails loudly; backups/retention stay locked until resolved (or `restic_force_fresh_start=<hostname>`). |
