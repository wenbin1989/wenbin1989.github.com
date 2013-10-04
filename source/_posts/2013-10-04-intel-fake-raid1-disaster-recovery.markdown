---
layout: post
title: "intel主板raid1灾难恢复小计"
date: 2013-10-04 10:32
comments: true
categories: linux
keywords: intel raid1 恢复 fsck resize2fs superblock
description: 重建intel raid1后，mount出现wrong fs type, bad option, bad superblock错误；fsck出现Either the superblock or the partition table is likely to be corrupt错误。resize2fs将filesystem size重设为physical size大小。
---

本人电脑用intel主板raid1来备份重要数据，昨天发现其中一块盘挂了。
二话不说换上一块新盘，开机Ctrl+I进入intel raid管理界面，设置好之后进入操作系统进行raid重建。

一是偷懒，二是想打会游戏，所以重建工作选择进入windows进行。
打开最新版Intel Matrix Storage Manager程序，会自动检测并重建raid1阵列。
也可以在linux下使用mdadm进行重建。

重建完成后，重启进入linux，运行`mount /dev/md126 /mnt`进行挂载，却出现以下错误：

> mount: wrong fs type, bad option, bad superblock on /dev/md126,
missing codepage or helper program, or other error
In some cases useful info is found in syslog - try
dmesg | tail or so

用testdisk查看分区表：

    testdisk /list /dev/md126

与原始分区相符，目测分区表没有问题，应该是文件系统受损。

用fsck进行检测：

    fsck /dev/md126

出现以下提示：

> The filesystem size (according to the superblock) is 244189980 blocks
The physical size of the device is 244189952 blocks
Either the superblock or the partition table is likely to be corrupt!
Abort &lt;y>?

猜测是新换的硬盘跟原来的型号不一样（一个希捷一个西数），造成的上述问题。

输入y退出fsck检测，google一番后，找到如下解决方案：

    resize2fs -f /dev/md126 244189952

运行该命令将filesystem size重设为244189952（fsck检测时提示的physical size）。
重新运行`fsck /dev/md126`进行检测，无任何问题。挂载后查看数据，完好如初。

谨以此记。
