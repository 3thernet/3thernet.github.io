---
layout: post
title: "Linux系统迁移"
date: 2023-09-19
author: "3thernet"
header-img: "img/bg-touhou-7.jpg"
tags:

- Linux

---

此前已在电脑上安装了双系统，最近又加装了一块固态硬盘，因此自然想着把本来在同一块硬盘上的系统分开分别放在两块硬盘上。对 Windows 进行迁移并不容易，而 Linux 下“一切皆文件”，自然就会简单很多。

### 0x01 分区

首先使用 `gnome-disks`查看分区情况，主要记住带有星号和三角符号（正在挂载）分区的`UUID`，之后要用到。然后使用`sudo gprepared`分配硬盘：

- fat32：EFI（300MB）  `/dev/nvme1n1p1`

- swap：交换分区（1.5\~2倍内存）  `/dev/nvme1n1p2`

- ext4：挂载根目录  `/dev/nvme1n1p3`

- ext4：挂载home目录  `/dev/nvme1n1p4`

如果 fat32 分区显示红色感叹号，请安装 `mtools` 工具：`sudo apt install mtools`

### 0x02 同步

将新的硬盘分区挂载到目录，并进行同步

（如果不是在同一台机器迁移，过程也是类似的）

```bash
sudo su
cd /mnt
# 挂载
mkdir root home efi
mount /dev/nvme1n1p1 /mnt/efi
mount /dev/nvme1n1p3 /mnt/root
mount /dev/nvme1n1p4 /mnt/home
# 同步
rsync -avx /boot/efi/ /mnt/efi
rsync -avx / /mnt/root
rsync -avx /home /mnt/home
# 卸载
umount root
umount home
umount efi
rm -rf *
```

接下来需要重新挂载目录，并修改`/mnt`为主目录，然后修改`/etc/fstab`和`/boot/grub/grub.cfg`，由于修改了主目录，因此这里修改的是新硬盘分区的文件

```bash
# 重新按照目录挂载
mount /dev/nvme1n1p3 /mnt  # 先挂载根目录
mount /dev/nvme1n1p1 /mnt/boot
mount /dev/nvme1n1p4 /mnt/home
# chroot 必要操作
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
chroot \mnt
nano /etc/fstab  # Ctrl+Shift+P 粘贴新设备 UUID
nano /boot/grub/grub.cfg  # Ctrl+\ 替换所有的根分区 UUID
```

更新 grub 引导，重启电脑：

```bash
grub-install /dev/nvme1n1p1  # EFI，这里有警告，忽略
update-grub
sync
exit  # 退出 chroot
umount /mnt/dev
umount /mnt/sys
umount /mnt/prc
umount /mnt/boot/efi
umount /mnt/home
exit
reboot
```

进入新系统后再次`update-grub`

### 0x03 清除多余启动项

此时虽然已经完成迁移，但原 Ubuntu 分区还存在，且 BIOS 和 UEFI 有一些重复选项

- 重启进入 Windows，进入硬盘管理，删除原 Ubuntu 的主分区，注意不要删除 EFI，因为大概率安装双系统时 Windows 和 Ubuntu 共享了 EFI，如果误删，请进入迁移后的 Ubuntu 挂载该分区然后同步回去

- 下载`EasyUEFI`，`管理EFI系统分区->EFI系统分区资源管理器`，打开 Windows 所在的 EFI，删除`\EFI\ubuntu`文件夹，同理打开 Ubuntu 所在的 EFI，删除`\EFI\Microsoft文件夹`，然后进入`管理UEFI启动项目`，删除对应启动选项，重启

- 如果删错了 Windows 启动项导致重启进入高级启动选项，则进入`疑难杂症->命令行`，输入`bcdboot c:\windows /l cn-zh`修复 EFI 分区

此外 Lenovo 的 BIOS 页面需要完全关机后再启动才会刷新

### 0x04 参考

- [Ubuntu18.04系统备份迁移手册 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/126228018)

- [迁移Ubuntu18.04至新硬盘 - Pabebe's Blog (pabebezz.github.io)](https://pabebezz.github.io/article/2f7fc36c/)

- [uefi - Ubuntu 21.04. Gparted warning symbol. How to fix FAT32 EFI partition? Or can I ignore it? - Ask Ubuntu](https://askubuntu.com/questions/1353148/ubuntu-21-04-gparted-warning-symbol-how-to-fix-fat32-efi-partition-or-can-i-i)

- [UEFI分区的重建办法，不需要额外软件_怎么用命令分区uefl_天已青色等烟雨来的博客-CSDN博客](https://blog.csdn.net/x356982611/article/details/81460876)
