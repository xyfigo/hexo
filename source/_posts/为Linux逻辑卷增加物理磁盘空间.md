---
title: 为Linux逻辑卷增加物理磁盘空间
date: 2018-12-28 09:08:15
tags: Linux LVM
categories: Technology
---

测绘资质管理系统的文件附件系统后端使用的是基于Linux逻辑卷群作为文件系统的SMB服务，由于附件容量非常庞大，在虚拟化平台上每个虚拟磁盘最大只能分派2T的容量，所以必须由多个虚拟磁盘共同组成一个大的逻辑卷空间进行使用，目前已分派的8T空间已满，需要继续增加空间以维持文件附件数据的存储。

未来希望将此套NAS服务替换为[Minio NAS Gateway](https://docs.minio.io/docs/minio-gateway-for-nas.html)，后端使用华为5500系列存储直接提供的NAS服务。



1. 查看逻辑卷组成

   pvdisplay

2. 停止SMB服务：

   service smb stop

3. 扩展卷空间

   （假如新增加的物理磁盘为/dev/sdd，空间为2T）

   vgextend data /dev/sdd              

   lvextend -L +2T /dev/data/file      

4. 卸载逻辑卷，检查空间并重新扫描空间

   umount /data/file/                  

   e2fsck -f /dev/data/file            非常慢

   resize2fs /dev/data/file  

5. 查看完成情况

   df -h                               

   lvdisplay                           

6. 重新挂载逻辑卷，并重启SMB服务

   mount /dev/data/file /data/file     

   df -h                               

   service smb start             