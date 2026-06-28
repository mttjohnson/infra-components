# Restic backup / restore / recovery design

This document is the authoritative record of the **disaster-recovery design** shared
by the `restic` and `restic_restore` roles. It is intentionally implementation-agnostic:
the same design applies to any implementation that uses these roles, whether it runs a
single instance or a cluster of cooperative instances.

The roles cooperate:

- **`restic`** installs the binary, establishes the repository password, writes the env
  file(s), initializes the repository (and an optional off-box secondary), and deploys the
  scheduled backup / retention / check jobs.
- **`restic_restore`** runs *after* `restic` in the play and restores declared state
  bundles from whichever repository holds the most recent snapshot.

The whole point of a backup system is to protect against data loss, so the design's first
priority is that **the backup/restore machinery must never itself cause data loss** — not
through a silent fresh-start that masquerades as a recovery, and not through automatic
retention quietly eroding the only surviving snapshots while a rebuild is unresolved.

## The failure this design prevents

Without the safeguards below, a routine instance rebuild can silently destroy data:

1. The data volume is rebuilt, so the local repository at `/data/backups` is gone and the
   live data (e.g. `/data/prometheus`) is gone. The off-box copy on the NAS still holds
   every snapshot.
2. `restic` runs, finds no host password, **generates a brand-new password**, and
   initializes a fresh, empty primary repository.
3. `restic_restore` probes that fresh repo, finds zero snapshots, and — because the
   bundles are not required — **silently skips every restore**.
4. The host converges from scratch. Scheduled backups begin capturing the empty/fresh
   state and the copy job replicates them to the NAS.
5. Over the following days the retention (`forget` + `prune`) jobs age out the genuine
   old snapshots. **By the time anyone notices, the real data is gone from every repo.**

Every step above is individually "successful." The disaster is the *absence* of an alarm.

## Core principle: distinguish first-deploy from rebuild

The system must tell apart two situations that look identical on a fresh host:

- **Genuine first deployment** — nothing was ever backed up; empty repositories are correct.
- **Rebuild / recovery** — backups exist and *must* be restored; empty repositories are a
  disaster.

It does this with out-of-band, rebuild-surviving **signals**, then refuses to proceed into
a state that could lose data unless those signals are satisfied or an operator explicitly
overrides.

## Signals

`recovery_expected` is true if **any** of these indicate a prior backup:

| Signal | Where it lives | Survives… | Meaning |
|---|---|---|---|
| Control-node password (`<prefix>.password`) | control node | host rebuild | restic was provisioned for this host before |
| Host password (`/etc/restic/password`) | remote host | control-node rebuild | the key is present on the host |
| Primary repo exists (`<repo>/config` present) | primary backend | host rebuild | an initialized primary repository is present |
| Secondary repo exists (`<repo>/config` present) | NAS / off-box | host **and** data-volume rebuild | an initialized off-box repository is present |
| Volume sentinel (`<volume>/.restic-volume-id`) | data volume + control | — (its *absence* is the signal) | the expected data volume is mounted and is the same one |
| Repo-identity pin (`<prefix>.<repo>.repo-id`) | control node | host & data rebuild | which repository this host is bound to |

Two signals are password-independent on purpose. The **repo `config` probe** needs no
password (the repository *structure* is visible without decrypting it), so it catches the
worst case — control-node password lost *and* host rebuilt — where the password-only signal
would wrongly read "first deploy." The **volume sentinel** catches the gap the password
signal misses: a host whose root filesystem is intact (host password present) but whose
**data volume was wiped, swapped, or failed to mount** — there the local repo is gone yet
nothing else flags it.

## The recovery decision

After seeding the password (see below) and classifying each configured repository as
`present-with-snapshots` / `present-but-empty` / `unreachable`:

| Condition | Outcome |
|---|---|
| Not `recovery_expected` (true first deploy) | Generate key, init, record sentinel + repo-id to control, **bless** the host. Normal. |
| A repository exists but **no usable password** anywhere (host or control) | **FAIL early.** Recover the control-node key from its off-host copy, then re-run. No init, no fresh password. |
| `recovery_expected`, some repository has snapshots | Seed key, **restore (required)** from the source with the newest snapshot, verify the declared paths are present and the targets populated, then **bless**. |
| `recovery_expected`, all repositories reachable but empty | **FAIL by default** (overridable). Likely benign, but could be a wrong/empty mount — make the operator confirm. |
| `recovery_expected`, any repository **unreachable** / sentinel missing | **FAIL.** A mount/network fault may be hiding real data. Override to proceed. |

Under `--check`, every hard failure above is downgraded to a prominent
"a real run WOULD FAIL here" warning so the full decision is previewable without aborting.

## Password handling

The repository password is the one irreplaceable secret: lose it and the backups are
unrecoverable by design. The design keeps a copy on the control node and uses it to
**reseed** a rebuilt host:

- **Seed before generate.** If the host has no password but the control node holds one for
  this host, push the control copy to `/etc/restic/password` and reuse it. Only when
  neither exists does the role generate a new password.
- **Write-once to the control node.** The fetch that copies the password back to the
  control node runs **only when no control copy exists yet**, so it can never clobber the
  good key. A deliberate re-key (`restic_allow_new_password`) archives the existing control
  copy (`<prefix>.password.<timestamp>.bak`) before writing the new one.

This closes the chicken-and-egg problem: the on-host password is backed up inside `/etc`,
but you cannot decrypt that backup to recover the password without already having it — so
the control-node copy (and your off-host replication of it) is the only thing that breaks
the cycle.

## Fail-closed job gate (protects backups *and* retention)

A single **blessed marker** on the data volume records "as of the last apply, this host is
in a known-good backup state." Every **state-changing** job script —
`backup`, `copy`, `forget`, `prune` (primary and secondary) — checks the marker and
**refuses to run when it is absent**. Read-only `check` / `check-data` jobs are not gated.

The marker is written **only** when an apply blesses the host (first deploy, or a confirmed
recovery, or an explicit override) and is **actively removed** whenever the role detects an
unconfirmed-recovery state. Consequences:

- A polluting backup of empty/fresh data cannot run, so nothing bad propagates to the NAS.
- **Retention cannot erode snapshots while a rebuild is unresolved.** If a rebuild fails and
  is left for days or weeks, `forget`/`prune` on both repositories stay parked, so no
  snapshot ages out until an apply blesses the host again. Snapshots only ever accumulate
  in the meantime — the safe direction.
- A gated job exits non-zero, tripping the existing `restic_failure_command` / systemd
  `OnFailure` handler, so the stuck state actively alerts instead of failing silently.

## Mount-failure protection

If the repository's backing volume fails to mount, an unguarded `restic backup` would write
the repository to the instance's local root disk at the mountpoint path — silently
defeating the off-box guarantee. Two configurable layers guard against this:

- **Volume sentinel (robust, default on).** A `<volume>/.restic-volume-id` file (host +
  UUID) written at first init and copied to the control node. If the control side expects a
  sentinel but the on-disk one is missing, the volume is fresh/unmounted → recovery path /
  fail-closed. Works reliably inside system containers where the volume is a device, not an
  fstab mount, so `mountpoint` semantics are unreliable.
- **`restic_require_mountpoint` (opt-in, default `false`).** A strict `mountpoint -q` check
  on the primary, mirroring the secondary's guard. Default off so a deliberately
  root-disk-only "light local backup" (no attached volume) works out of the box; set `true`
  on hosts where the repository lives on an attached volume *and* `mountpoint` reports it
  reliably.

## Restore correctness

`restic restore --include <path>` against a path that is **not** present in the snapshot
does not error — it restores nothing and exits 0. Combined with population-based idempotency
(a non-empty target is treated as "already restored"), a typo'd or absent path, or a
half-written target, can masquerade as a successful restore. Guards:

- **Path-presence check.** Before restoring a bundle, verify each declared path actually
  exists in the chosen snapshot (`restic ls <snapshot> <path>`). For a recovery-critical
  bundle, a missing path fails loudly instead of restoring nothing.
- **Post-restore population verification.** After restoring a required bundle, confirm the
  effective targets are populated; an empty target after a "successful" restore is an error.

## `restore_required` semantics

`restore_required: true` on a bundle marks it **recovery-critical**. Its enforcement is
**conditioned on `recovery_expected`**:

- Fail when `recovery_expected AND restore_required AND` (no matching snapshot / a declared
  path is absent / the restore could not be verified).
- On a genuine first deploy (`recovery_expected = false`), a `restore_required: true` bundle
  with no snapshot is **skipped**, not failed — you cannot restore what was never backed up,
  and the host should converge fresh.

This makes `restore_required` the single, per-bundle place to declare which pieces are
recovery-critical, while keeping it safe to set `true` on a fresh install. It is preferred
over blanket-forcing every bundle when `recovery_expected`, because it preserves the
operator's ability to mark some bundles non-critical even during a recovery.

## Multi-source restore (local vs off-box)

`restic_restore` queries **every** configured source (primary env, secondary env), picks the
snapshot with the newest `time` across all of them (ties resolve to the earlier source —
the primary), and restores from that source. Because `restic copy` preserves each snapshot's
`time`, identical snapshots tie and the local primary wins; after a data-volume rebuild the
primary is empty/absent and the off-box NAS holds the newest snapshot, so the role naturally
restores from the NAS. This is the headline rebuild scenario and is a first-class path.

## Asymmetric retention

Local and off-box retention are independent. The off-box (NAS) copy is the last-resort
safety net and keeps a **longer tail**; the local copy may keep **less** to save disk.
Because `restic copy` only ever *adds* snapshots and never deletes, a longer off-box tail
survives even a polluting fresh-start far longer than the local copy would — buying recovery
time even if every other guard somehow failed.

## Overrides

Narrow and separate, so no single flag disables all safety:

- **`restic_force_fresh_start`** — bypass the recovery gate and accept starting fresh. To
  prevent fat-fingering it onto the wrong host, it takes effect **only when its value equals
  the target `inventory_hostname`**; any other value is ignored with a warning.
- **`restic_allow_new_password`** — permit generating a new password even though a control
  copy exists (a deliberate re-key/reset). Independent of `restic_force_fresh_start`, and it
  archives the existing control-node key rather than overwriting it.

## Control-node state directory

All per-host control-node state lives in one flat directory (`restic_control_dir`, default
`restic-state`) with the host as a filename **prefix** (no nested subdirectories), so a
clustered implementation can keep several instances' state side by side:

| File | Secret? | Purpose |
|---|---|---|
| `<prefix>.password` | **yes** | repository password (git-ignored; replicate off-host) |
| `<prefix>.primary.repo-id` | no | primary repository identity pin |
| `<prefix>.secondary.repo-id` | no | secondary repository identity pin |
| `<prefix>.volume.sentinel-id` | no | data-volume sentinel UUID |

`<prefix>` defaults to `inventory_hostname` (`restic_control_state_prefix`). Only
`*.password` is git-ignored; the non-secret pins are safe to commit alongside the
implementation configuration so they survive a control-node rebuild via the repo.

## Implementation status

This document is the **target design**. It is being implemented in stages, each validated
under `--check`:

1. ✅ **Done.** Password seed from control + write-once (anti-clobber) fetch + control-dir
   rename (`restic-passwords/` → flat `restic-state/`).
2. ✅ **Done.** Password-independent repository `config` detection + recovery decision gate
   (fail-early when a repo exists but no key) + the fail-closed blessed marker across the
   state-changing jobs + the primary mountpoint guard + `restic_force_fresh_start`. The
   blessed marker is owned by `restic_restore`, which writes it on a confirmed recovery /
   first deploy and removes it (locking out backups + retention) on a recovery miss, with
   `restore_required` enforcement conditioned on `recovery_expected`.
3. ✅ **Done.** Multi-source restore (`restic_restore_sources`: local primary + off-box
   secondary, newest snapshot wins, ties → primary) so a rebuild whose local repo is gone
   restores automatically from the off-box copy + post-restore population verification
   (fails when a restore left a declared target empty because the path was absent from the
   chosen snapshot).
4. ⏳ Repo-identity pinning + volume sentinel write/verify + asymmetric retention +
   `restic_allow_new_password` re-key (with control-key archival).

> Until stage 4 lands, the **volume sentinel** and **repo-identity pin** signals described
> above are part of the target design but not yet written, so the "data volume swapped
> while the root filesystem is intact" gap is not yet closed. Stages 1–2 close the main
> rebuild path (host and/or control-node rebuild) and the retention-erosion window.

See each role's `README.md` for the per-variable reference.
