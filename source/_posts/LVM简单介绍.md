---
title: LVM简单介绍
date: 2018-10-22 13:22:23
tags: Linux LVM
categories: Technology
---

原文见：http://www.debian-administration.org/articles/410

LVM允许你用一种有效地方法创建和管理服务器上的存储，按照意愿增加、移除、改变分区大小。刚开始接触LVM会使初学者感到疑惑，而这篇文字就用简单的方式讲述LVM的基本知识。

以下术语对你学习使用LVM很重要，是你必须知道的：

- 物理卷（physical volumes）

  这是你的物理硬盘或磁盘分区，例如/dev/hda或/dev/hdb1.这是你在mounting/unmounting时经常使用的。利用LVM能够合并多个物理卷成为卷群（volume groups）

- 卷群（volume groups）

  卷群由真实的物理卷组成，它能够用来创建逻辑卷（logical volumes）。你能够创建、改变大小、移除、使用逻辑卷。可以认为卷群是一个由任意数量的物理卷组成的虚拟分区。

- 逻辑卷（logical volumes）

  逻辑卷是你最终在系统上挂载的，它们能够快速的添加、移除、改变大小。因为逻辑卷由卷群组成，所有它的空间能大于组成它的单一物理卷的空间。例如由4个5G大小的硬盘（逻辑卷）能够组合成20G的卷群，所有你能在此基础上创建两个10G的逻辑卷。

在逻辑层面上，他们的概念层次自上而下如图所示。

{% asset_img  LVM_hierarchy.png LVM的概念层次%}

# 创建卷群（Volume Group）
使用LVM你至少需要一个分区，并使用LVM进行初始化。然后把它加入一个卷群。怎样操作呢？

在我的例子中，我的笔记本有如下设置：

###  Name        Flags    Part Type   FS Type          [Label]        Size (MB)
```shell
hda1        Boot        Primary   		Linux ext3       [/]             8000.01 
hda2                    Primary   		Linux swap 		/ Solaris        1000.20
hda3                    Primary   		Linux                            31007.57
```

这里我有一个7GB的根分区，用来存储我的Debian GNU/Linux安装。我有一个28GB的分区用来给LVM使用。按照我的设置，我能用LVM创建一个专用的/home分区，而且我能够在空间不足时扩展/home分区增大空间。

在这个例子中hda1、hda2、hda3是物理卷。我对hda3进行物理卷初始化：

```sh
root@lappy:~# pvcreate /dev/hda3
```

如果你想合并硬盘或者分区，你可以像这样操作：

```shell
root@lappy:~# pvcreate /dev/hdb
root@lappy:~# pvcreate /dev/hdc
```

一旦你初始化好了分区或磁盘驱动器，我们用这些物理卷来创建卷群：
```shell
root@lappy:~# vgcreate skx-vol /dev/hda3
```
这里skx-vol是卷群的名称。如果你想跨两个磁盘创建卷群，你可以使用“vgcreate skx-vol /dev/hdb /dev/hdc”的方式。

如果你操作正确，你可以通过vgscan命令查看：

```shell
root@lappy:~# vgscan
  Reading all physical volumes.  This may take a while...
  Found volume group "skx-vol" using metadata type lvm2
```
这样我们就创建好了一个名为skx-vol的卷群，我们下面开始使用他。

# 创建和使用逻辑卷（Working with logical volumes）


我们真正想要的是创建一个能够挂载和使用的逻辑卷。如果以后我们的逻辑卷空间不足，可以对其空间进行重新分配，增加新的存储空间。当然这取决于你算则的文件系统类型。

我们将创建一个名为test的卷：

```shell
root@lappy:~# lvcreate -n test --size 1g skx-vol
Logical volume "test" created
```

这个命令创建了一个大小为1G名为test的逻辑卷，它在LVM卷群skx-vol之上。

现在这个逻辑卷可以通过/dev/skx-vol/test进行操作，它能像其他分区一样进行格式化和挂载：

```shell
root@lappy:~# mkfs.ext3 /dev/skx-vol/test
root@lappy:~# mkdir /home/test
root@lappy:~# mount /dev/skx-vol/test  /home/test
```

很酷是吧？

下面的操作将更有意思！我们假设这个test分区空间已满，我们要对这个分区进行扩大。首先，我们可以通过lvdisplay命令查看当前硬盘的状态：

{% codeblock lang:shell %}
root@lappy:~# lvdisplay 
  --- Logical volume ---
  LV Name                /dev/skx-vol/test
  VG Name                skx-vol
  LV UUID                J5XlaT-e0Zj-4mHz-wtET-P6MQ-wsDV-Lk2o5A
  LV Write Access        read/write
  LV Status              available
  # open                 0
  LV Size                1.00 GB
  Current LE             256
  Segments               1
  Allocation             inherit
  Read ahead sectors     0
  Block device           254:0
{% endcodeblock %}
我们能看到test分区大小为1G。在我们对它重新分配空间时记得要先卸载这个卷（unmount）
```shell
root@lappy:~# umount  /home/test/
root@lappy:~# lvextend -L+1g /dev/skx-vol/test 
Extending logical volume test to 2.00 GB
Logical volume test successfully resized
```
(注意：ext3文件系统可以在挂载状态重划大小，但是我建议还是先把它卸载掉，这样可以免除不必要的担心)

在此通过lvdisplay命令查看，test卷已经重新分配了大小：
```shell
root@lappy:~# lvdisplay 
  --- Logical volume ---
  LV Name                /dev/skx-vol/test
  VG Name                skx-vg
  LV UUID                uh7umg-7DqT-G2Ve-nNSX-03rs-KzFA-4fEwPX
  LV Write Access        read/write
  LV Status              available
  # open                 0
  LV Size                2.00 GB
  Current LE             512
  Segments               1
  Allocation             inherit
  Read ahead sectors     0
  Block device           254:0
```
需要着重提醒的是，即使这个卷的空间重新分配，但是它上面的ext3文件系统却没有改变，我们需要对文件系统重划大小满足扩大了的卷：
```shell
root@lappy:~# e2fsck -f /dev/skx-vol/test 
root@lappy:~# resize2fs /dev/skx-vol/test
```
然后对test卷重新挂载后你会发现它的空间变成了2G。

当然你也可以通过lvremove操纵移除这个卷：
```shell
root@lappy:~# lvremove /dev/skx-vol/test
Do you really want to remove active logical volume "test"? [y/n]: y
Logical volume "test" successfully removed
```

# 挂载逻辑卷（Mounting Logical Volumes）


在之前的章节，我们使用如下的命令来挂载一个逻辑卷：
```shell
mount /dev/skx-vol/test  /home/test
```

如果你想在开机引导的时候就挂载你的分区，你可以编辑/etc/fstab文件，增加如下内容：
```shell
/dev/skx-vol/home    /home       ext3  noatime  0 2
/dev/skx-vol/backups /backups    ext3  noatime  0 2
```