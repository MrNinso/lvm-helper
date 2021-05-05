# How I configure proxmox

- Using xfs
- Using raid5 over lvm2

# Copy Partition Table
Get the /dev/sdX partition table to a file

````bash
$ sfdisk -d /dev/sdX > table
````

Restore the partition table to /dev/sdY and /dev/sdZ

````bash
$ sfdisk /dev/sdY < table
$ sfdisk /dev/sdZ < table
````
## Install Grub
````bash
$ grub-install /dev/sdY
$ grup-install /dev/sdZ
````

Clone BIOS and UEFI partitions
````bash
$ dd if=/dev/sdX1 of=/dev/sdY1
$ dd if=/dev/sdX1 of=/dev/sdZ1
$ dd if=/dev/sdX2 of=/dev/sdY2
$ dd if=/dev/sdX2 of=/dev/sdZ2
````

# Convert Proxmox Logical volumes to raid5

Extend Volume group

````bash
$ vgextend pve /dev/sdY3 /dev/sdZ3
````

## Recreate data Logical Volume
[Based](https://listman.redhat.com/archives/linux-lvm/2014-January/msg00049.html)

Remove data lvm
````bash
$ lvremove pve/data
````

Create metadata logical volume
````bash
$ lvcreate -n meta_data --type raid5 -L 256M pve
````

Create data logical volume 
````bash
$ lvcreate -n data --type raid5 --nosync -L 500G pve
````

Now let's convert to a Thinpool
```bash
$ lvconvert --thinpool pve/data --poolmetadata meta_data
```

# Convert root and swap Logical volumes

First you need to convert from linear to raid1

````bash
$ lvconvert --type raid1 pve/root
$ lvconvert --type raid1 pve/swap
````

Wait the raid1 sync and convert to a raid5
````bash
$ lvconvert --type raid5 pve/root
$ lvconvert --stripes 2 pve/root

$ lvconvert --type raid5 pve/swap
$ lvconvert --stripes 2 pve/swap
````

Wait for sync and enjoy your raid5 in proxmox :)

# (Optional) Grow root Logical Volume
````bash
$ xfs_growfs /dev/pve/root
````
