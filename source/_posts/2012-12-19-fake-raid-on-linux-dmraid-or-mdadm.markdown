---
layout: post
title: "linux之主板raid(fake raid): dmraid or mdadm"
date: 2012-12-19 16:40
comments: true
categories: linux
keywords: linux 主板 fake raid dmraid mdadm
description: linux下最重要的raid管理程序为dmraid与。鉴于mdadm的各种优点与不断的更新完善，以及dmraid似乎停止开发，管理fake raid与software raid的首选均为mdadm。另外，mdadm也得到了intel的极力推荐。
---

当系统有多块硬盘时，通过组建[raid][1]以提升磁盘性能或提供磁盘冗余，往往成为人们的首选考量。
当今主流raid实现方案大致可分为三种：

* 硬件raid(hardware raid)，通过购买昂贵的raid卡实现，我等平民不考虑。
* 软件raid(software raid)，通过操作系统内软件创建阵列，raid处理开销由CPU负责。
* 主板raid(fake raid)，通过主板内建raid控制器创建阵列，由操作系统驱动识别。

需要注意的是，fake raid仅提供廉价的控制器，raid处理开销仍由CPU负责，因此性能与CPU占用基本与software raid持平。
如果只有单个linux系统，使用software raid一般比fake raid更健壮，但是，在多启动环境中(例如windows与linux双系统)，为了使各个系统都能正确操作相同的raid分区，就必须使用fake raid了。
可参考[FakeRaidHowto @ Community Ubuntu Documentation][2]以获得更多相关信息。

linux下最重要的raid管理程序为[dmraid][3]与[mdadm][4]。
本人印象中，前者用于识别rake raid，后者用于管理software raid，但最近更新系统时突然发现，openSUSE及rhel等发行版都改用mdadm替代dmraid来处理rake raid。
放狗搜索一番后，发现大家都有相似的疑问，便想到简单考察总结一番，对linux下rake raid应该使用dmraid还是mdadm进行管理做一个评判。
<!--more-->

*说明：本人操作系统为openSUSE 12.2 x86_64，主板南桥为ICH10R，两块硬盘通过Intel Matrix Storage Manager Option Rom工具程序(开机自检时按Ctrl + I进入)设定为raid 0。*

##起源与发展

dmraid从linux kernel version 2.6.18起引入，由Red Hat维护，主要目的为管理fake raid，通过libdevmapper与device-mapper kernel runtime实现。

mdadm从linux kernel version 2.6.27起引入，由SUSE维护，主要目的为管理software raid，通过md (Multiple Devices) device driver实现。

``` bash
$ /sbin/dmraid --version
dmraid version:         1.0.0.rc16-3 (2010.01.12)
dmraid library version: 1.0.0.rc16-3 (2010.01.12)
device-mapper version:  unknown
```
``` bash
$ /sbin/mdadm --version
mdadm - v3.2.5 - 18th May 2012
```
由以上对比可以看出，dmraid自2010年初就没有更新过了，mdadm则一直更新频繁。
特别是，mdadm 3.0版本加入对[external metadata][5]的支持；mdadm 3.2.1版本的release说明中有如下内容：

> Secondly, the support for Intel Matrix Storage Manager (IMSM) arrays has been substantially enhanced. Spare migration is now possible as is level migration and OLCE (OnLine Capacity Expansion). This support is not quite complete yet and requires MDADM_EXPERIMENTAL=1 in the environment to ensure people only use it with care. In particular if you start a reshape in Linux and then shutdown and boot into Window, the Windows driver may not correctly restart the reshape. And vice-versa.

自此，mdadm正式踏入管理fake raid的舞台。

##功能比较

下图为[Intel文档][6]中对dmraid与mdadm的简单比较，以供参考：

![comparison of dmraid and mdadm](https://lh5.googleusercontent.com/-ZgSuXbQSn5E/UNK-0CcKq-I/AAAAAAAAAJE/uZAQg1Epow0/s1229/comparison_of_dm_and_md.png)

另外已知的区别有：

* mdadm可以使用其他电脑的磁盘进行恢复。
* dmraid有时无法正确识别大硬盘(3TB imsm raid1被识别为746GB)。
* *不断添加中...*

##实战比较

使用openSUSE 12.2 x86_64 DVD安装系统，dmraid与mdadm均能正确识别已经设置好的raid分区，正常安装、引导启动系统。唯一不同的是，dmraid需要设置单独的/boot分区，mdadm则无此限制。

安装完成后，用dd测试硬盘读写速度并监控CPU使用率，二者无任何差别。

##结论

鉴于mdadm的各种优点与不断的更新完善，以及dmraid似乎停止开发，管理fake raid与software raid的首选均为mdadm。
另外，mdadm也得到了intel的极力推荐。

有关mdadm的具体使用，本文不再讨论，可参考[Intel® Rapid Storage Technology in Linux][6]。

***欢迎大家帮助补充更多相关资料。***

[1]: http://en.wikipedia.org/wiki/RAID
[2]: https://help.ubuntu.com/community/FakeRaidHowto
[3]: http://linux.die.net/man/8/dmraid
[4]: http://linux.die.net/man/8/mdadm
[5]: https://raid.wiki.kernel.org/index.php/RAID_setup#External_Metadata
[6]: http://download.intel.com/design/intarch/papers/326024.pdf
