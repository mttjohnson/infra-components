To see the partition details:

```bash
parted /dev/sda print
```

```
Model: Argon Forty (scsi)
Disk /dev/sda: 1000GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
 1      1049kB  269MB   268MB   primary  fat32        boot, lba
 2      269MB   100GB   99.7GB  primary  ext4
 3      100GB   1000GB  900GB   primary
```

```bash
parted /dev/nvme0n1 print
```

```
Model: CT1000P3PSSD8 (nvme)
Disk /dev/nvme0n1: 1000GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
 1      1049kB  538MB   537MB   primary  fat32        boot, lba
 2      538MB   100GB   99.5GB  primary  ext4
 3      100GB   1000GB  900GB   primary
```

```
Model: Micron MTFDKBA1T0TFH (nvme)
Disk /dev/nvme0n1: 1024GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  4296MB  4295MB  fat32              boot, esp
 2      4296MB  112GB   107GB   ext4
 3      112GB   1024GB  913GB   zfs          data
```

```
Model: KINGSTON OM8PGP41024Q-A0 (nvme)
Disk /dev/nvme0n1: 1024GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  1128MB  1127MB  fat32              boot, esp
 2      1128MB  109GB   107GB   ext4
 3      109GB   1024GB  916GB                data
```