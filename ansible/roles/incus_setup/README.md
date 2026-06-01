# Externally managed certificates

The role can symlink externally managed certs (e.g. Let's Encrypt, issued via
`certbot`) into Incus, replacing the self-signed cert/key it generates on install.
The certs to manage are listed in `incus_externally_managed_certs`, whose default
single entry points the server cert/key at
`/etc/letsencrypt/live/{{ acme_dns_reg_cert_domains | first }}/`.

| Variable | Default | Description |
| --- | --- | --- |
| `incus_setup_manage_external_certs` | `{{ (acme_dns_reg_cert_domains \| default([])) \| length > 0 }}` | Whether to symlink externally managed certs into Incus. Defaults to enabled only when there are cert domains to manage. |
| `incus_externally_managed_certs` | server cert/key entry (see `defaults/main.yml`) | List of certs to symlink; only consulted when `incus_setup_manage_external_certs` is true. |
| `incus_use_certbot_deploy_hook_incus_reload` | `false` | Install a certbot deploy hook that reloads Incus on cert renewal. |
| `incus_use_certbot_deploy_hook_update_cluster_cert` | `false` | Install a certbot deploy hook that updates the Incus cluster cert on renewal. |

Hosts with **no** external certs must leave `acme_dns_reg_cert_domains` empty (or
unset). In that case `incus_setup_manage_external_certs` resolves to `false`, the
cert-symlink loop is skipped, and the `acme_dns_reg_cert_domains | first` lookup in
the cert paths is never evaluated. Set `incus_setup_manage_external_certs` explicitly
to override the domain-presence default in either direction.

# Troubleshooting issues with LVM seeing guest VM volumes

https://serverfault.com/questions/639152/how-can-i-tell-linux-to-ignore-disk-partitions-its-already-discovered

https://forum.proxmox.com/threads/lvm-filter-configured-but-multipath-underlying-device-still-in-use.140098/
https://forum.proxmox.com/threads/wired-behaviour-about-the-global_filter-configuration-in-lvm-conf.71095/

https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/logical_volume_manager_administration/lvm_filters#lvm_filters
https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_and_managing_logical_volumes/limiting-lvm-device-visibility-and-usage_configuring-and-managing-logical-volumes#applying-an-lvm-device-filter-configuration_the-lvm-device-filter

Make sure LVM is ignoring ZFS volumes on the host so it doesn't try to use guest LVM volumes

This should result in /etc/lvm/lvm.conf containing something like
```
devices {
	# global_filter = [ "a|.*|" ]
  global_filter = [ "r|/dev/zd.*|" ] 
}
```
