---
title: 'Storage system at BTB and related issues'
date: 2020-10-08
permalink: /posts/2020/10/BTB_Storage/
tags:
  - Dell Complement
  - LVM
---
### Finding documentation online

These entries below are not user tutorials to the Dell Compellent system whatsoever. Online source
for this kind of storage is at:
 
- [SAN Storage Works](https://www.sanstorageworks.com/datasheets/Compellent/Compellent-DataProg-DS.pdf)
- [Dell / RHEL 7x](https://downloads.dell.com/solutions/storage-solution-resources/SC-Series-with-RHEL-7x-Dell-EMC-2018-(CML1071).pdf)
- [Dell Storage Admin Guide ](https://dl.dell.com/topicspdf/storage-sc2000_administrator-guide_en-us.pdf)

### Checking the system
First you have to connect via SSH and have an X11 connection as well. The most straightforward is to connect by
an RDP client using port 3389, like Remmina, using the Xvnc session. Once logged in, open a terminal, become root,
and start virt-manager to access the Dell management software running Windows:
```
$ sudo bash
[root@munin /home/username]# export XAUTHORITY=/home/username/.Xauthority
[root@munin /home/username]# virt-manager 
```  
Choose 'mgmt0' and login to Windows. Instead of pressing Ctr-Alt-Del, use the "Send Key" menu. Once logged in,
start the Dell Storage Management software. You also have to log in into this system, once there, you will 
have a view of the Storage Manager. Go to the "Storage" tab first. Select the storage center you are going 
to play with, and you can have a view of many items. The most important now are the volumes and the disks. 

When looking around the volumes, you will see their status. It is going to be OK when they are green or
orange. When they are red - well, there is something fishy going on. Ditto, looking at the disks you can
see disk pools assigned to different volumes. Note, every disk pool has two components: the Read-intensive 
SSDs are usually full at their 90% - it is normal, SSDs are used to buffer data for processing. We are more
concerned about the 7K disks, as these are the real storage media. 

### Adding new disks and expanding an array
You have to ask CMM people to physically add the disks first. Once they are at their place, you can go to Storage
Manager to assign the new disks to an existing disk pool. Go to Storage / Disks, and there should be a set of disks
marked as "Unassigned". These should be first assigned to the desired disk pool - we are considering that
the disk pool is already linked to a volume.

Once the new disks are assigned, we have to expand the volume by clicking first on the "Volumes" item and the
selected volume, finally _right_ click on the selected volume to access the "Expand volume" menu. Type the new
size (in terabytes) and press OK. Note, you can expand a volume, but there is no way to shrink it. Furthermore,
this expansion is virtual now, you can assign 123456T space on a single disk, there will be little effect in the
moment. Even when looking at the storage on the server, you will see nothing. I have already changed the size 
for *mpathf* from 32T to 34T in the Storage Manager, but it can not be seen yet:
```
# multipath -l| egrep "Compel|size"
mpathf (36000d31003e30400000000000000000b) dm-7 COMPELNT,Compellent Vol  
size=32T features='1 queue_if_no_path' hwhandler='1 alua' wp=rw  
```
To add this extra space - especially if we added new disks we have 
 - to rescan the buses
 - restart multipathd
 - resize the partition
 - resize PV and extend LV ([here](https://opensource.com/business/16/9/linux-users-guide-lvm) is an introduction to LVM )
 - grow the filesystem 

These steps are in details below, first, have to find out the names for devices:
```
multipath -l| awk '/[0-9]:[0-9]:[0-9]:[0-9]/{print}'| cut -b 6-12
1:0:7:4
1:0:6:4
...
```  

If we have something like that above, we can reset them like:

```
for device in `multipath -l| awk '/[0-9]:[0-9]:[0-9]:[0-9]/{print}'| cut -b 6-12`; do \
    echo 1 > /sys/class/scsi_device/${device}/device/rescan ; \
done
```
and restart multipathd with `systemctl restart multipathd.service` . After restart we should see the expanded volume
in the list of multipath:
```
 # multipath -l mpathf
mpathf (36000d31003e30400000000000000000b) dm-7 COMPELNT,Compellent Vol  
size=34T features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
...
```
Next step is to resize the partition. First print out the partition, use `parted`, and type 'Fix' for the questions:
```
# parted /dev/mapper/mpathf print
Error: The backup GPT table is not at the end of the disk, as it should be.  This might mean that another operating system believes the disk is smaller.  Fix, by moving the backup to the end (and removing the old backup)?
Fix/Ignore/Cancel? Fix                                                    
Warning: Not all of the space available to /dev/mapper/mpathf appears to be used, you can fix the GPT to use all of the space (an extra 4294967296 blocks) or continue with the current setting? 
Fix/Ignore? Fix                                                           
Model: Linux device-mapper (multipath) (dm)
Disk /dev/mapper/mpathf: 37,4TB
Sector size (logical/physical): 512B/4096B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name       Flags
 1      2097kB  35,2TB  35,2TB               Linux LVM  lvm
```
Now we can do the resize for real:
```
# parted /dev/mapper/mpathf resizepart 1 100%
Information: You may need to update /etc/fstab.

# partprobe /dev/mapper/mpathf                   
# parted /dev/mapper/mpathf print
Model: Linux device-mapper (multipath) (dm)
Disk /dev/mapper/mpathf: 37,4TB
Sector size (logical/physical): 512B/4096B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name       Flags
 1      2097kB  37,4TB  37,4TB               Linux LVM  lvm
```

Resize PVs by:
```
# pvs /dev/mapper/mpathf1
  PV                  VG       Fmt  Attr PSize    PFree
  /dev/mapper/mpathf1 storage2 lvm2 a--   <32,00t    0 
# pvresize /dev/mapper/mpathf1
  Physical volume "/dev/mapper/mpathf1" changed
  1 physical volume(s) resized or updated / 0 physical volume(s) not resized
# pvs /dev/mapper/mpathf1
  PV                  VG       Fmt  Attr PSize    PFree
  /dev/mapper/mpathf1 storage2 lvm2 a--   <34,00t 2,00t
```

Ditto LVs:
```
# lvs /dev/storage2/data2
  LV    VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  data2 storage2 -wi-ao---- <32,00t
# lvextend -l+100%FREE  /dev/storage2/data2
  Size of logical volume storage2/data2 changed from <32,00 TiB (8388606 extents) to <34,00 TiB (8912894 extents).
  Logical volume storage2/data2 successfully resized.
# lvs /dev/storage2/data2
  LV    VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  data2 storage2 -wi-ao---- <34,00t
```

Finally, grow the filesystem:

```
# df -h /data2
Filesystem                  Size  Used Avail Use% Mounted on
/dev/mapper/storage2-data2   32T   18T   15T  56% /data2
# xfs_growfs /data2
meta-data=/dev/mapper/storage2-data2 isize=512    agcount=33, agsize=268434944 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=8589932544, imaxpct=5
         =                       sunit=512    swidth=512 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=521728, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 8589932544 to 9126803456
# df -h /data2
Filesystem                  Size  Used Avail Use% Mounted on
/dev/mapper/storage2-data2   34T   18T   17T  53% /data2
```

### Creating a new volume / new disk array
It is recommended not to make an array larger than ~70T though I know there is one that is already quite bigger. 
The main point is not make all the disks as a single huge array, as RAID rebalance takes ages and if there is
a failure at least some of the arrays still can be used. 

If you do not have any volumes yet, it is relatively straightforward to make one: create a new volume (right-click on 
Volumes and Create Volume), after that assign the new 
disks to the volume. Generally, it is difficult to estimate the capacity of the system, as the tiered architecture 
makes it impossible to calculate it, as you would do for RAID-10. Therefore, the foolproof approach is to make a small
volume first, set its filesystem up, and increase the size in the next step instead of making one huge volume and 
realizing it is overcommited. OTOH, as I have mentioned earlier, it is possible to make a volume that is larger than the
available disk space, but we will fail with file system creation when we are trying to use the stuff.    
      
If there is only one volume in the storage center (i.e. if you have a new server with only one volume set up) new disks 
will be assigned to this volume automatically. I do not know whether it is a feature or a bug, but if this happens, 
you have to unassign the yet unusued disks, make a new volume, and assign them to the new one. I had no chance to try
it out, but my feeling is that it is better to make a new volume *before* adding the new disks to avoid this feature.

When createing a brand-new volume, the steps are more or less the same as previously when adding new disks to an 
existing volume. Things to consider:

- once created a new volume, that would appear in the multipath list (`multipath -l`) - note its id like **mpathg**
- find the SCSI IDs with in this list, they look like `1:0:6:2 sde 8:64 active undef running` and rescan them as above
- using `parted` create a new primary partition (partition 1) for the new volume - though you can use the whole 
device as a single partition, it will be awkward to continue with the rest of the procedure: 
```
# parted /dev/mapper/mpathg
# pvcreate /dev/mapper/mpathg1
# pvscan
# pvdisplay /dev/mapper/mpathg1
# vgscan
# vgdisplay storageX
# vgs
# vgextend -Ay storageX /dev/mapper/mpathg1
# lvextend -l 100%ORIGIN /dev/storageX/dataX
# vgdisplay
# mkfs.xfs /dev/mapper/mpathg1
# mkdir /dataX
```
Finally extend `/etc/fstab` with line using TAB delimiters:
```
...
/dev/mapper/storageX-dataX      /dataX  xfs     defaults        0 0
```
          

### Changing default snapshot policy
The default policy is not  

### - to remove a multipath device:
umount and lvremove, vgremove, pvremove as needed
multipath -l # to read device names
echo 1 >  /sys/block/sdX/device/delete
multipath -f /dev/mapper/mpathX

