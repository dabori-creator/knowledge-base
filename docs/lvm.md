# Adding a New Disk, LVM

## 1. Basic Terms

**Volume Group (VG)** — a high-level container that holds one or more logical and physical volumes.

**Physical Volume (PV)** — a storage device (hard disk or other data medium).

**Logical Volume (LV)** — equivalent to a disk partition and, like a partition, can contain a file system.

**Physical Extent (PE)** — each physical volume (PV) is divided into fixed-size blocks known as physical extents.

## 2. Creating a New Partition

First, check which disks the system sees:

```bash
fdisk -l
```

If the disk is not visible but exists in VMware, you need to tell the controller to "rescan" the device list:

```bash
echo "- - -" > /sys/class/scsi_host/host2/scan
```

Create a new PV:

```bash
pvcreate /dev/sdd
```

If you get an error like 'Device /dev/sdd excluded by a filter' or 'Cannot use /dev/sdd: device is partitioned' — you need to wipe the disk with:

```
wipefs --all /dev/sdd
```

You see:

```bash
Physical volume "/dev/sdd" successfully created.
```

Verify that the PV was created:

```bash
pvdisplay
```

Create a VG:

```bash
vgcreate vg_pgbackup /dev/sdd
```

Check the list of VGs:

```bash
vgdisplay
```

Create a logical volume named `lv_pgbackup` in the group `vg_pgbackup` using 100% of free space:

```bash
lvcreate -l 100%FREE -n lv_pgbackup vg_pgbackup
```

Create a logical volume of 100 GB named `lv_pgbackup` in the group `vg_pgbackup`:

```bash
lvcreate -L 100G -n lv_pgbackup vg_pgbackup
```

View all LVs:

```bash
lvdisplay
```

Create an ext4 file system:

```bash
mkfs.ext4 /dev/vg_pgbackup/lv_pgbackup
```

Create a mount point in the root directory:

```bash
cd /
mkdir pgbackup
```

Mount the partition:

```bash
mount /dev/vg_pgbackup/lv_pgbackup /pgbackup
```

To mount permanently, open the file:

```bash
vi /etc/fstab
```

And add the following line:

```
/dev/vg_pgbackup/lv_pgbackup  /pgbackup    ext4    defaults        1 2
```

Mount all entries:

```bash
mount -a
```

Verify that the disk is mounted:

```bash
df -hT
lsblk -f
```

## 3. Extending an Existing Partition

As an example, we will extend the `var` partition. Follow step 2 to create a PV and check which VGs exist:

```bash
vgdisplay
```

Add the PV to the VG:

```bash
vgextend VG_SystemData /dev/sdd
```

Extend the LV:

1. By a specific amount (this command adds 30 GB to the volume group `VG_SystemData` for the `var` partition):
   
   ```bash
   lvextend -L+30G /dev/VG_SystemData/var
   ```

2. To a target size:
   
   ```bash
   lvextend -L500G /dev/VG_SystemData/var
   ```

3. As a percentage:
   
   ```bash
   lvextend -l +100%Free /dev/VG_SystemData/var
   ```

Check the partitions:

```bash
lsblk -f
```

Resize the file system on the target partition:

1. For ext2/ext3/ext4:
   
   ```bash
   resize2fs /dev/VG_SystemData/var
   ```

2. For XFS:
   
   ```bash
   xfs_growfs /dev/VG_SystemData/var
   ```

## 4. Reducing Volumes

LVM allows you to reduce the size of a volume by unmounting it:

```bash
umount /pgbackup
```

Perform a disk check:

```bash
e2fsck -fy /dev/vg_pgbackup/lv_pgbackup
```

Reduce the file system size:

```bash
resize2fs /dev/vg_pgbackup/lv_pgbackup 500M
```

Reduce the volume size:

```bash
lvreduce -L-500 /dev/vg_pgbackup/lv_pgbackup
```

Confirm the reduction.

## 5. Removing Volumes

If you need to completely dismantle LVM volumes, perform the following steps.

Unmount the partitions:

```bash
umount /pgbackup
```

You must remove the corresponding entry from fstab (otherwise your system may fail to boot after reboot):

```bash
vi /etc/fstab
```

Delete the line:

```
/dev/vg_pgbackup/lv_pgbackup /pgbackup ext4 defaults 1 2
```

View information about logical volumes:

```bash
lvdisplay
```

Now remove the logical volume:

```bash
lvremove /dev/vg_pgbackup/lv_pgbackup
```

Confirm when prompted.

> If the system returns "Logical volume contains a filesystem in use", make sure you have unmounted the volume.

View information about volume groups:

```bash
vgdisplay
```

Remove the volume group:

```bash
vgremove vg01
```

Remove the LVM label from the disks:

```bash
pvremove /dev/sd{b,c,d}
```

> In this example, we de-initialize disks `/dev/sdb`, `/dev/sdc`, `/dev/sdd`.

You should see:

```
Labels on physical volume "/dev/sdb" successfully wiped.
Labels on physical volume "/dev/sdc" successfully wiped.
Labels on physical volume "/dev/sdd" successfully wiped.
```