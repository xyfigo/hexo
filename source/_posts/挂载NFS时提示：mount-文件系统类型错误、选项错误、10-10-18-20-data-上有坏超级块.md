---
title: '挂载NFS时提示：mount: 文件系统类型错误、选项错误、10.10.18.20:/data 上有坏超级块'
date: 2019-01-28 09:27:12
tags: Linux
categories: Technology
---

最近购买力华为OceanStor系列存储，获得了存储级NAS能力。可以由存储直接提供NFS服务。

在将NFS挂载到Linux时，提示了如下错误：

```shell
#  mount -t nfs 10.10.18.20:/data /mnt/nfs
mount: 文件系统类型错误、选项错误、10.10.18.20:/data 上有坏超级块、
       缺少代码页或助手程序，或其他错误
       (对某些文件系统(如 nfs、cifs) 您可能需要
       一款 /sbin/mount.<类型> 助手程序)

       有些情况下在 syslog 中可以找到一些有用信息- 请尝试
       dmesg | tail  这样的命令看看。
```

主要原因是由于没有安装nfs的客户端，所以需要使用yum进行安装

```shell
yum -y install nfs-utils
systemctl start nfs-utils
systemctl enable nfs-utils
rpcinfo -p
mount -t nfs 10.10.18.20:/data /home/ddsc/data/
```

挂载成功。

