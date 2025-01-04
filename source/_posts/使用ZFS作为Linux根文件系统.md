---
title: 使用ZFS作为Linux根文件系统
toc: true
original: true
date: 2023-03-12 16:49:43
tags:
- Linux
categories:
- 随笔
---

## 背景
现代文件系统一般支持CoW、快照等特性，如果将它作为操作系统的根文件系统，那么可靠性能更胜一筹。

本文主要记录将现有的Arch Linux转成使用ZFS文件系统的过程，注意系统需要提前装上zfs，这可以通过安装`zfs-dkms`包。

## ISO加载最小系统
重启系统进入UEFI，选择安装镜像iso进入，该最小系统作为操作环境。官方提供的 `archlinux-x86_64.iso` 默认不支持zfs，目前有两种方式：
- 使用 [https://github.com/eoli3n/archiso-zfs](https://github.com/eoli3n/archiso-zfs) 补丁
- 下载支持ZFS的安装镜像 [https://archzfs.leibelt.de](https://archzfs.leibelt.de)

## 备份已有系统
进入最小系统后，将rootfs挂载到`/mnt/`下，同时挂载备份`/mnt2`：

```
# mount /dev/vda2 /mnt
# mount /dev/vdb /mnt2
# rsync -aAXHPv /mnt /mnt2
# umount /mnt
```

## 重建文件系统
目前系统存在两个分区`boot`和`rootfs`：

```
# fdisk -l
Disk /dev/vda: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 88BCFD5A-DBB8-4B45-93C4-87C4DC4C5861

Device       Start      End  Sectors  Size Type
/dev/vda1     2048  1026047  1024000  500M EFI System
/dev/vda2  1026048 41940991 40914944 19.5G Linux filesystem
```

使用`fdisk`调整`rootfs`的类型为`Solaris root`：

```
# fdisk /dev/vda

Welcome to fdisk (util-linux 2.38.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): t
Partition number (1,2, default 2): 2
Partition type or alias (type L to list all): 156

Changed type of partition 'Linux filesystem' to 'Solaris root'.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

查看`rootfs`的PARTID，并利用zpool创建池：
```
# blkid /dev/vda2
/dev/vda2: PARTUUID="fb4e1dbd-bf9a-3e41-bef3-f862d0eb699d"

# zpool create -f -o ashift=12 -O atime=off -O compression=lz4 -O canmount=off -R /mnt zroot fb4e1dbd-bf9a-3e41-bef3-f862d0eb699d
```

创建两个dataset，主要分别存放系统文件和用户数据文件：
```
# zfs create -o mountpoint=/ -o canmount=noauto zroot/ROOT
# zfs create -o mountpoint=none zroot/DATA
```

系统分区：
```
# zfs create -o mountpoint=/var -o acltype=posixacl -o xattr=sa zroot/ROOT/var
```

数据分区：
```
# zfs create -o mountpoint=/home zroot/DATA/home
# zfs create -o mountpoint=/root zroot/DATA/root
```

检查配置：
```
# zfs list -r zroot
NAME              USED  AVAIL     REFER  MOUNTPOINT
zroot            1.10M  18.9G       96K  /mnt/zroot
zroot/DATA        288K  18.9G       96K  none
zroot/DATA/home    96K  18.9G       96K  /mnt/home
zroot/DATA/root    96K  18.9G       96K  /mnt/root
zroot/ROOT        192K  18.9G       96K  /mnt
zroot/ROOT/var     96K  18.9G       96K  /mnt/var
```

恢复数据：

```
# zpool export zroot
# zpool import -d /dev/disk/by-partuuid/fb4e1dbd-bf9a-3e41-bef3-f862d0eb699d  -R /mnt zroot -N
# zfs mount zroot/ROOT
# zfs mount -a
# rsync -aAXHPv /mnt2/ /mnt/
```

挂载boot分区：
```
# mount /dev/vda1 /mnt/boot
```

设置启动标记：
```
# zpool set bootfs=zroot/ROOT zroot
```

## 更新GRUB
使用 `arch-chroot` 进入系统：
```
# arch-chroot /mnt
```

删掉 `/etc/fstab` 多余的项，仅留下 `/boot'：

```
# cat /etc/fstab
# <file system> <dir> <type> <options> <dump> <pass>
UUID=482D-024B          /boot           vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro 0 2
```

编辑`/etc/mkinitcpio.conf`，将`zfs`模块加到`HOOK`项里，注意顺序放到`keymap`后边：

```
HOOKS=(base udev autodetect modconf kms keyboard keymap zfs consolefont block filesystems fsck)
```

生成initramfs：
```
# mkinitcpio -p linux
```

更新grub配置：
```
# grub-mkconfig -o /boot/grub/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-linux
Found initrd image: /boot/initramfs-linux.img
Found fallback initrd image(s) in /boot:  initramfs-linux-fallback.img
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done
```

最后重启进入系统。

```
[root@archlinux ~]# neofetch
                   -`                    root@archlinux
                  .o+`                   --------------
                 `ooo/                   OS: Arch Linux x86_64
                `+oooo:                  Host: KVM/QEMU (Standard PC (Q35 + ICH9, 2009) pc-q35-7.2)
               `+oooooo:                 Kernel: 6.2.2-arch2-1
               -+oooooo+:                Uptime: 7 mins
             `/:-:++oooo+:               Packages: 191 (pacman)
            `/++++/+++++++:              Shell: bash 5.1.16
           `/++++++++++++++:             Resolution: 1280x800
          `/+++ooooooooooooo/`           Terminal: /dev/pts/0
         ./ooosssso++osssssso+`          CPU: Intel N100 (1) @ 806MHz
        .oossssso-````/ossssss+`         GPU: 00:01.0 Red Hat, Inc. Virtio GPU
       -osssssso.      :ssssssso.        Memory: 210MiB / 946MiB
      :osssssss/        osssso+++.
     /ossssssss/        +ssssooo/-
   `/ossssso+/:-        -:/+osssso+-
  `+sso+:-`                 `.-/+oso:
 `++:.                           `-/+/
 .`                                 `/

[root@archlinux ~]# df -h
Filesystem       Size  Used Avail Use% Mounted on
dev              463M     0  463M   0% /dev
run              474M  732K  473M   1% /run
zroot/ROOT        19G  3.2G   16G  18% /
zroot/ROOT/var    16G  639M   16G   4% /var
tmpfs            474M     0  474M   0% /dev/shm
tmpfs            474M     0  474M   0% /tmp
zroot/DATA/home   16G   44M   16G   1% /home
zroot/DATA/root   16G  256K   16G   1% /root
/dev/vda1        499M   90M  410M  18% /boot
tmpfs             95M     0   95M   0% /run/user/0
```

## 注意事项
考虑`/var`作为dataset存在，在挂载之前journald已经往/var写日志，因此zfs将无法卸载`/var`，考虑使用`zfs-mount-generator`方案：https://wiki.archlinux.org/title/ZFS#Using_zfs-mount-generator

## 参考链接
- [https://wiki.archlinux.org/title/Install_Arch_Linux_on_ZFS](https://wiki.archlinux.org/title/Install_Arch_Linux_on_ZFS)
- [https://zhiyb.wordpress.com/2022/11/28/change-rootfs-from-ext4-to-zfs/](https://zhiyb.wordpress.com/2022/11/28/change-rootfs-from-ext4-to-zfs/)
- [https://wiki.archlinux.org/title/ZFS](https://wiki.archlinux.org/title/ZFS)