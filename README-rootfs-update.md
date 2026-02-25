- [OpenWrt RootFS 更新指南](#openwrt-rootfs-更新指南)
  - [概述](#概述)
  - [1. 基于 uboot 的系统更新 - 把 openwrt 系统刷入 SD 卡](#1-基于-uboot-的系统更新---把-openwrt-系统刷入-sd-卡)
    - [1.1 将编译出来的 openwrt.img 中的 boot 和 root mount 到 host](#11-将编译出来的-openwrtimg-中的-boot-和-root-mount-到-host)
    - [1.2 将 openwrt root 中的 kernel 和 dtb 拷贝到 SD (sda1) 卡中](#12-将-openwrt-root-中的-kernel-和-dtb-拷贝到-sd-sda1-卡中)
    - [1.3 将 openwrt 中的 rootfs 拷贝到 SD (sda2) 卡中](#13-将-openwrt-中的-rootfs-拷贝到-sd-sda2-卡中)
    - [1.4 通过 uboot 刷入 openwrt linux+rootfs](#14-通过-uboot-刷入-openwrt-linuxrootfs)


# OpenWrt RootFS 更新指南

## 概述

当需要更新 rootfs 分区时，OpenWrt 提供了多种方法。根据设备类型和存储介质的不同，更新 rootfs 的方法也有所不同。本指南将详细说明如何将新的 rootfs 覆盖原来的 rootfs。

## 1. 基于 uboot 的系统更新 - 把 openwrt 系统刷入 SD 卡

### 1.1 将编译出来的 openwrt.img 中的 boot 和 root mount 到 host
```
fdisk -l openwrt-bcm27xx-bcm2712-rpi-5-ext4-factory.img

    Disk openwrt-bcm27xx-bcm2712-rpi-5-ext4-factory.img: 176 MiB, 184549376 bytes, 360448 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0x9b67bfa8

    Device                                          Boot  Start    End Sectors  Size Id Type
    openwrt-bcm27xx-bcm2712-rpi-5-ext4-factory.img1 *      8192 139263  131072   64M  c W95 FAT32 (LBA)
    openwrt-bcm27xx-bcm2712-rpi-5-ext4-factory.img2      147456 360447  212992  104M 83 Linux


sudo mount -o loop,offset=4194304,sizelimit=67108864 openwrt-bcm27xx-bcm2712-rpi-5-ext4-factory.img /mnt/openwrt_boot

sudo mount -o loop,offset=75497472,sizelimit=109051904 openwrt-bcm27xx-bcm2712-rpi-5-ext4-factory.img /mnt/openwrt_root
```

### 1.2 将 openwrt root 中的 kernel 和 dtb 拷贝到 SD (sda1) 卡中
```
lsblk

    NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    loop0         7:0    0    64M  0 loop /mnt/openwrt_boot
    loop1         7:1    0   104M  0 loop /mnt/openwrt_root
    sda           8:0    1  29.7G  0 disk 
    ├─sda1        8:1    1   512M  0 part /run/media/wyou/bootfs
    └─sda2        8:2    1  29.2G  0 part /run/media/wyou/rootfs

mv /run/media/wyou/bootfs/kernel_2712.img /run/media/wyou/bootfs/kernel_2712.img.old_
mv /run/media/wyou/bootfs/bcm2712-rpi-5-b.dtb /run/media/wyou/bootfs/bcm2712-rpi-5-b.dtb.old_
cp /mnt/openwrt_boot/kernel_2712.img /run/media/wyou/bootfs/kernel_2712.img
cp /mnt/openwrt_boot/bcm2712-rpi-5-b.dtb /run/media/wyou/bootfs/bcm2712-rpi-5-b.dtb
```

### 1.3 将 openwrt 中的 rootfs 拷贝到 SD (sda2) 卡中
```
sudo rsync -avh /run/media/wyou/rootfs/ rootfs.old_/
sudo rm -rf /run/media/wyou/rootfs/*
sudo rsync -avh /mnt/openwrt_root/ /run/media/wyou/rootfs/

sudo umount /run/media/wyou/bootfs
sudo umount /run/media/wyou/rootfs
```

### 1.4 通过 uboot 刷入 openwrt linux+rootfs
```
git@github.com:wt2017/u-boot-raspberrypi-fork.git
READMErpi.md # Boot Linux
```

