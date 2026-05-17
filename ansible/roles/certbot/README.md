# Certbot

Installs Certbot (Let's Encrypt)
https://certbot.eff.org/

https://eff-certbot.readthedocs.io/en/latest/using.html#manual

Auto-renew is handled by a snap timer. View with `systemctl list-timers` and test with `certbot renew --dry-run`.

This will actually update the magic acme records on acme-dns
```bash
sudo certbot renew --dry-run
```

## Requirements

* Certbot requires snapd be installed

## View snap services

```bash
snap services certbot.renew
snap logs certbot
```

## Storing certs on a separate data volume

To keep Let's Encrypt keys and certs on a separately attached disk (so they
survive rebuilds of the system volume), set:

```yaml
certbot_relocate_data_dir: true
certbot_custom_data_dir: /data/letsencrypt   # default
```

When enabled, the role will:

1. Create `certbot_custom_data_dir` if it does not exist.
2. If `/etc/letsencrypt` is an existing real directory, `rsync` its
   contents into `certbot_custom_data_dir` (rsync is required rather than
   `copy` because `/etc/letsencrypt/live/<domain>/*.pem` are symlinks
   into `archive/` that must be preserved), then remove the source.
3. Replace `/etc/letsencrypt` with a symlink to `certbot_custom_data_dir`.

The parent directory (e.g. `/data`) must already exist and be mounted —
the role asserts this and will not create or mount it for you. If both
`/etc/letsencrypt` and `certbot_custom_data_dir` already hold data, the
role fails rather than silently merging them.

The role installs `rsync` on the target when the relocation actually
needs to run, and uses `ansible.posix.synchronize` with
`delegate_to: "{{ inventory_hostname }}"` so the copy runs locally on
the target between two local paths.

## TODO
- [ ] backup to S3 compatible storage via hook and ansible check
- [ ] auto restore from S3 compatible storage if not existing locally
