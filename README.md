# LVM2 (Logical Volume Manager)

## What is LVM?

Logical Volume Manager (LVM) is a **device mapper framework** that provides logical volume management for the Linux kernel. Most modern Linux distributions are LVM-aware, enabling their root file systems to reside on logical volumes.  

LVM offers tools to create **virtual block devices** from physical devices, making them easier to manage and providing additional capabilities beyond what physical devices offer.  

- **Volume Group (VG):** A collection of one or more physical devices, each called a Physical Volume (PV).  
- **Logical Volume (LV):** A virtual block device that the system or applications can use.  

Each block of data in an LV is stored on one or more PVs in the VG, with storage algorithms managed by the kernel's **Device Mapper (DM)**.

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
---
# 3.2- LVM - VG LV Extend Operation

## Extend VG amd LV to have more space

The difference is that lvextend can only increase the size of a volume, whereas lvresize can increase or reduce it. This makes lvresize more powerful but more dangerous.

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
