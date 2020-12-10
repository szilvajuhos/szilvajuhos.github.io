---
title: 'Storage system at BTB and related issues - part 2'
date: 2020-12-18
permalink: /posts/2020/12/BTB_Storage_02/
tags:
  - Dell Complement
  - LVM
---
``

### Changing default snapshot policy
The default policy is not  

### - to remove a multipath device:
umount and lvremove, vgremove, pvremove as needed
multipath -l # to read device names
echo 1 >  /sys/block/sdX/device/delete
multipath -f /dev/mapper/mpathX

