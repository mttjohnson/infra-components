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
