# LVM2 (Logical Volume Manager)

## What is LVM?

Logical Volume Manager (LVM) is a **device mapper framework** that provides logical volume management for the Linux kernel. Most modern Linux distributions are LVM-aware, enabling their root file systems to reside on logical volumes.  

LVM offers tools to create **virtual block devices** from physical devices, making them easier to manage and providing additional capabilities beyond what physical devices offer.  

- **Volume Group (VG):** A collection of one or more physical devices, each called a Physical Volume (PV).  
- **Logical Volume (LV):** A virtual block device that the system or applications can use.  

Each block of data in an LV is stored on one or more PVs in the VG, with storage algorithms managed by the kernel's **Device Mapper (DM)**.
![Alt Text](image/lvm.png)
---

## Why Do We Need LVM?

LVM provides flexibility and scalability for storage management.  

### Key Benefits:
1. **Flexibility:**  
   - Logical and physical volumes can be created, resized, and deleted **without requiring a system restart**.  
   - Logical volumes can be resized dynamically, allowing partitions to expand as needed.  

2. **Scalability:**  
   - Easily add more physical disks to extend storage.  
   - Merge multiple disks and partitions into a single logical volume.  

3. **Easy Migration:**  
   - Supports migration to different storage devices with simple commands.  

4. **Advanced Features:**  
   - Instant snapshots of logical volumes while the OS is running.  
   - Advanced encryption support for secure storage.

---

## LVM Operations

### 1. LVM - PV, VG, LV Creation

#### Commands:
```bash
# List Disks
lsblk | grep -i 'sd[a-z]'

# List available disks
lvmdiskscan | grep -v loop

# Check existing physical volumes
pvs

# Format the Disks
fdisk /dev/sdb  # Create partitions like sdb1, sdb2
fdisk /dev/sdc  # Create sdc1

# Create Physical Volumes (PV)
pvcreate /dev/sdb1 /dev/sdc1

# Check Volume Groups (VG)
vgs

# Create a Volume Group (VG)
vgcreate data_vg /dev/sdb1 /dev/sdc1

# Create Logical Volumes (LV)
lvcreate data_vg -n data_lv -L 2G
#if the disk is not os disk you can utlize 100% of it by the following command
lvcreate data_vg -n data_lv -l 100%FREE
#to delete logical volume if you create it with wrong size , you can use the follwing command, but if the logical volume is mounted you have to umount it before removing the logical volume 
lvremove /dev/data_vg/data_lv

# Format the Logical Volume
mkfs.ext4 /dev/mapper/data_vg-data_lv

# Create another Logical Volume using remaining space
lvcreate data_vg -n lv_log -l 100%FREE

# Format the new Logical Volume
mkfs.ext4 /dev/mapper/data_vg-lv_log

# Create directories and mount volumes
mkdir /data /log
mount /dev/mapper/data_vg-data_lv /data
mount /dev/mapper/data_vg-lv_log /log
```
# Update /etc/fstab (Optional) , this is to make mount persistent on reboot
To ensure the logical volume mounts automatically on reboot, add an entry to /etc/fstab. Open the file in an editor:

```
nano /etc/fstab
```
Add the following line:
```
/dev/data_vg/data_lv  /mnt/data  ext4  defaults  0 0
```
---
# 3.2- LVM - VG LV Extend Operation

## Extend VG amd LV to have more space

The difference is that lvextend can only increase the size of a volume, whereas lvresize can increase or reduce it. This makes lvresize more powerful but more dangerous.
```
# Check pv
pvs

# check the vg
vgs 

# check the lvs
lvs 


# check if resize_inode options is active 

tune2fs -l /dev/data_vg/data_lv | grep resize_inode

# check available disks 
lvmdiskscan | grep -v loop
lsblk | grep -v loop

# create PV 
pvcreate /dev/sdb2

# Check pv
pvs

# add pv to vg
vgextend data_vg /dev/sdb2

# Check pv
pvs

# check the vg
vgs 


# check the mount
mount | grep data_lv

# fsck
 fsck -N /dev/mapper/data_vg-data_lv

# check current lv size 
lvs 
df -h | grep -v "tmpfs\|loop" | grep data_lv

# extend lv using automatic way 
lvresize --resizefs /dev/mapper/data_vg-data_lv -L +1G

# check the vg
vgs 

# check the lvs
lvs 

# extend lv using manual way 
lvextend /dev/mapper/data_vg-data_lv -L 4G

# check current lv size 
lvs 
df -h | grep -v "tmpfs\|loop" | grep data_lv


# resize to fs
resize2fs /dev/data_vg/data_lv

# check lvs 
lvs 

# vgs lvs 
vgs 
lvs 

# extend and resize with full size of the VG 
lvextend /dev/mapper/data_vg-data_lv -l +100%FREE

# check 
vgs 
lvs 
df -h | grep -v "tmpfs\|loop" | grep data_lv

# resize to fs
resize2fs /dev/data_vg/data_lv

# check 
vgs 
lvs 
df -h | grep -v "tmpfs\|loop" | grep data_lv
```
---
# 3.3- LVM- LV Shrink Operation
In the last Section we have learned how to add more space to VG and LV, so this video will be about, how we can Shrink the LV, the shrinking process is risky task need preparation before you performing it on production env, and the main goal of lvreduce is to free some space to VG. Please to Pay attention for two things, first one is, the LV used space and available space, you need to shrink the freespace amount, don't try to shrink more than the freespace, the second one you need to check the filesystem whether it ext4, xfs or any other filesystems this is important to see how you can perform the shrink operation.

to perform shrink operation you need to understand one think you have two layers first one is the Logical Volume (LV) belongs to lvm, and the second one is os filesystem (FS), that means you need first to shrink the filesystem then you need to reduce or resize the LV, note that if you reduce the LV without do it first on filesystem, the logical volume (LV) (FileSystem) FS will be currepted.

EXT4 filesystem not support online shrink you need to umount the LV first then you can resize the FS then reduce or shrink the LV

XFS filesystem is not supporting shrink operation till now as per the xfs offical site published on 2019.
```
# Before proceed to shrink task 
# 1- what the FS you have ext4,xfs ..etc.
# 2- check free space for LV.
# 3- the size you can give for LV

# ## Shrink the ext4 file system and the LVM LV

# check the free space 
vgs 

# check 
lvs 


# check free space 
df -h | grep -v 'loop\|tmpfs'

# check mount 
mount | grep -i data 




# umount 
umount /data

#  lvresize provides an automatic resize underlaying 
filesystem
# --size option takes the intended new LV size 
# --resizefs resize the underlaying filesystem
lvresize --resizefs --size 2G /dev/data_vg/data_lv

#mount 
mount /dev/mapper/data_vg-data_lv /data

# check size
df -h | grep -v 'loop\|tmpfs'

# write dump data 
dd if=/dev/zero of=/data/test1 bs=3M count=100

# check space again 
df -h | grep -v 'loop\|tmpfs'

# lvreduce Manual way 

# check mount 
mount | grep -v loop 

# check free space 
# check free space
df -h | grep -v 'loop\|tmpfs'1

# umount 
umount /logs 

# shrink fs 
e2fsck -f /dev/mapper/data_vg-log_lv
resize2fs /dev/mapper/data_vg-log_lv 2G

# reduce lv 
lvreduce -L 2G /dev/mapper/data_vg-log_lv

# check lvs 
lvs data_vg

# check fs 
e2fsck -f /dev/mapper/data_vg-data_lv

# mount 
mount /dev/mapper/data_vg-log_lv /logs

# check size
df -h | grep -v 'loop\|tmpfs'

# write dump data 
dd if=/dev/zero of=/logs/test1 bs=3M count=100

# check space again 
df -h | grep -v 'loop\|tmpfs'
```
---
# 3.4- LVM- VG Migration
This section will cover the volume group migration. Well, we have two ways for Volume group migration , the first one is using LVM mirroring and the second one is using LVM pvmove command. This video will cover the second way which is pvmove command, also in this video we have two scenarios will be covered, the first one will be re-claim the PV and migrate VG to one partition within the Volume Group (VG) and the second one will be migrate Volume Group (VG) to new hard drive :
```
# there is many reasons sometime force us to migrate the Volume group on of those reasons we need to reclaim some partitions or to replace a faulty disk, or replace an existing smaller size disk with a large one.

# list all available disks 
lsblk | grep -v loop 

# vgs gather some stats 
vgs -o+devices | grep data_vg


# lvs check the lv belong to this VG
lvs -o+devices data_vg 

# check the used space help us to see how log pvmove will take
df -h | grep -v 'loop\|tmpfs'
pvs -o+pv_used
pvs -o+pv_used| grep -i data_vg

# pvmove to the desired partition 
pvmove /dev/sdb1 /dev/sdc1
pvmove /dev/sdb2 /dev/sdc1

# check pvs 
pvs -o+pv_used

# check the lsblk
lsblk | grep -v loop 

###########
# Migrate to new added disk 
###########

# list all available disks 
lsblk | grep -v loop 

# fdisk the new drive 
fdisk /dev/sdd
# n p 1 enter enter t 8e p w

#pvs and vgs 
vgs 
pvs 

# create pv 
pvcreate /dev/sdd1 

# pvs and vgs  
pvs 
vgs 

# add PV to VG
vgextend data_vg /dev/sdd1

# pvs and vgs  
pvs 
vgs 

# check 
pvs -o+pv_used

pvmove /dev/sd[bc][12]

# reduce the vg
vgreduce data_vg /dev/sd[bc][12]

# check 
pvs -o+pv_used
```
---
# 3.5- LVM- PV VG LV Removal Operations
This secition will cover the removing of PV VG LV, but before we get started, please to be careful, this operation will wipe out all the data from the disks, so please don't perform it on systems that have critical production data or sensitive data, and if its so, please make sure to take a backup of those data before perform the LVM removal operation. With all that being said, lets jump in to the demonstration.
```
#Check the mount points 
df -h | grep -v 'loop\|tmpfs' | grep data

# check the lv belongs to vg 
 lvs data_vg
 lsblk | grep -v loop 

# check if the disk mounted 
mount | grep data_vg

# umount the disk 
umount /data /logs
mount | grep data_vg

# remove lv
lvs data_vg
lvremove -f  /dev/data_vg/data_lv /dev/data_vg/log_lv
lvs data_vg

# check the lvs belong to the vg
lsblk | grep -v loop

# check the vg 
vgs data_vg
pvs 

# remove vg
vgremove data_vg

# check vgs 
vgs 

# check the pvs
pvs

# remove the pvs 
pvremove /dev/sdb1 /dev/sdb2 /dev/sdc1 /dev/sdd1
pvremove /dev/sd[bcd][12]

# check again pvs 
pvs
lsblk | grep -v loop

# delete partitions 

fdisk /dev/sdb
fdisk /dev/sdc
fdisk /dev/sdd

# check the disks 
lsblk | grep -v loop
```
