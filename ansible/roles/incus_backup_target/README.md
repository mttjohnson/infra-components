# incus_backup_target

Mount a [Synology](https://www.synology.com/) NFS share at a standard path on every Incus host, so that [Incus](https://linuxcontainers.org/incus/) disk devices can bind subdirectories of the share into containers and VMs. The mount path is identical fleet-wide, which lets a profile or instance reference `source={{ incus_backup_target_mount_path }}/<subdir>` without per-host special casing.

This role manages **only the host-side mount**. Per-instance subdirectories are created by Incus at instance-creation time — the role never creates them.

## Requirements

- Ansible 2.15+
- `ansible.posix` collection (provides `ansible.posix.mount`)
- Debian or Ubuntu target host
- A Synology NAS (or any NFS server) exporting the backup share, reachable from every Incus host
- The export configured to permit the Incus hosts in the Synology NFS permissions (host/subnet, squash, security)
- Outbound access to the NAS on the NFS ports (TCP 2049 for NFSv4)

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `incus_backup_target_state` | `"mounted"` | `mounted` mounts the share and persists it in `/etc/fstab`; `absent` unmounts and removes the fstab entry. |
| `incus_backup_target_nfs_host` | `"127.0.0.1"` | Synology NFS server hostname or IP. Inert placeholder by default; set per host/group. |
| `incus_backup_target_nfs_export` | `"/volume/backup"` | Export path as published by the NAS, e.g. `/volume1/LabBackup`. |
| `incus_backup_target_mount_path` | `"/srv/backup"` | Standard local mount path. Must be identical across hosts so Incus disk devices can reference it uniformly. |
| `incus_backup_target_mount_opts` | `"vers=4.1,rw,hard,timeo=600,_netdev,noatime,nofail"` | NFS mount options. `hard` + `_netdev` so the mount waits for networking and blocks (rather than erroring) on a NAS reboot; `nofail` so a host still boots if the NAS is unreachable. |
| `incus_backup_target_mount_dir_mode` | `"0755"` | Mode of the mount point directory. Set it to match the mode the mounted share presents (see idempotency note). |

The mount `src` is composed as `{{ incus_backup_target_nfs_host }}:{{ incus_backup_target_nfs_export }}` and the filesystem type is `nfs4`. The mount point directory is created `root:root`.

### Mount point mode and idempotency

The directory-mode task enforces `incus_backup_target_mount_dir_mode` against `incus_backup_target_mount_path`. Before the share is mounted that path is the local mount point; **once mounted, that same path is the share root as the NFS server presents it**. If the configured mode does not match what the server exports, every converge reports a change and a real run re-`chmod`s the mounted share root.

So set `incus_backup_target_mount_dir_mode` to whatever the server presents. For an `all_squash` export (e.g. a Synology share mapping all users to one identity), that is commonly `0777`:

```yaml
incus_backup_target_mount_dir_mode: "0777"
```

The `0755` default suits a generic/local mount point; override per implementation to keep the role idempotent against the specific share.

## Example Playbooks

Mount a Synology export at the default path on all Incus hosts:

```yaml
- hosts: incus_hosts
  roles:
    - role: mttjohnson.infra.incus_backup_target
      vars:
        incus_backup_target_nfs_host: "10.10.30.77"
        incus_backup_target_nfs_export: "/volume1/LabBackup"
        incus_backup_target_mount_path: "/srv/nas_lab_backup"
```

Typically the host/export/path live in `group_vars` so the role can be applied bare:

```yaml
# group_vars/system.yml
incus_backup_target_nfs_host: "10.10.30.77"
incus_backup_target_nfs_export: "/volume1/LabBackup"
incus_backup_target_mount_path: "/srv/nas_lab_backup"
```

```yaml
- hosts: incus_hosts
  roles:
    - role: mttjohnson.infra.incus_backup_target
```

Unmount and remove the fstab entry:

```yaml
- hosts: incus_hosts
  roles:
    - role: mttjohnson.infra.incus_backup_target
      vars:
        incus_backup_target_state: absent
```

## Verification

The role ends with a built-in smoke test (tagged `smoketest`) that writes a
host-unique probe file under the mount, reads it back, and removes it — failing the
play with a clear message if any step fails. Run it on its own against an
already-mounted host:

```bash
ansible-playbook playbooks/system.yml --tags smoketest
```

Manual checks on an Incus host after applying the role:

1. Confirm the share is mounted at the expected path with the expected type:

   ```bash
   findmnt --target /srv/nas_lab_backup
   findmnt -t nfs4 /srv/nas_lab_backup
   ```

2. Confirm it will come back on reboot (an fstab entry exists):

   ```bash
   grep nas_lab_backup /etc/fstab
   ```

3. Confirm an Incus instance can bind a subdirectory of the share. Incus creates the
   subdirectory at instance-creation time; add a `disk` device pointing at it and read
   it back from inside the guest:

   ```bash
   incus config device add test-instance backup disk \
     source=/srv/nas_lab_backup/test-instance path=/backup
   incus exec test-instance -- ls -la /backup
   incus config device remove test-instance backup
   ```

A clean result is the mount present in `findmnt` and `/etc/fstab` and the `smoketest`
play reporting `SMOKE TEST OK`.

## License

MIT
