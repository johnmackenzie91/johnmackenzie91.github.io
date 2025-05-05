+++
title = 'Format an HDD for use as a Longhorn Drive'
date = 2024-01-22T10:14:05Z
+++

I needed to upgrade a drive in one of my kubernetes nodes. Here are the steps that I followed.

1. Identify the Disk

Run the following command to list all disks:

```bash
lsblk
```

Identify the target HDD you want to format (e.g., /dev/sdb).

2. Partition the Disk

Launch fdisk to partition the disk:

```bash
sudo fdisk /dev/sdb
```
Inside fdisk run:

* Press d to delete any existing partitions (if any).
* Press n to create a new partition.
* Choose p for a primary partition.
* Press Enter to accept the default partition size (use the entire disk).
* Press t to set the partition type, and choose 8e for Linux LVM (recommended for Longhorn).
* Press w to write changes and exit.


3. Create a Filesystem

Longhorn typically works best with a filesystem like ext4 or xfs. Here, we'll use ext4:
```bash
sudo mkfs.ext4 /dev/sda
```

4. Label the Partition

Assign a meaningful label to the disk to make it easier to manage:

```bash
sudo e2label /dev/sda longhorn
```

5. Mount the Disk

```bash
sudo mkdir /mnt/longhorn
sudo mount /dev/sda /mnt/longhorn
df -h | grep longhorn
```

6. Add the Disk to Longhorn

* Ensure Longhorn is running in your Kubernetes cluster.
* Navigate to the Longhorn UI.
* Add the disk under the Settings → Node & Disk section:
  * Specify the disk path (e.g., /mnt/longhorn). 
  * Set it as Schedulable.

7. Ensure Proper Permissions

Set the necessary permissions for Longhorn to access the disk:

```bash
sudo chmod -R 777 /mnt/longhorn
```

## Finally. Ensure the drive is mounted automatically on startup.

1. Identify the Drive's UUID

The UUID (Universally Unique Identifier) ensures consistent identification of the drive.
Run the following command to find the UUID of the partition:

```bash
sudo blkid /dev/sda
```

2. Edit /etc/fstab

Open  /etc/fstab and add the drive configuration.

```bash
sudo nano /etc/fstab
```
Something like 

```bash
    UUID=123e4567-e89b-12d3-a456-426614174000 /mnt/longhorn ext4 defaults 0 2
```

3. Test the Configuration

It is very important to test the configuration. Why? If there is a problem you will not know until the next time your machine reboots!

```bash
sudo mount -a
```

Any errors will be output to stout.