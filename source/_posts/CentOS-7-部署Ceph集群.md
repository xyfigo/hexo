---
title: CentOS 7 部署Ceph集群
date: 2019-04-29 09:38:00
tags: Ceph
categories: Technology
---

#三、部署集群

#### 1. 安装准备，创建文件夹

在管理节点上创建一个目录，用于保存 ceph-deploy 生成的配置文件和密钥对。

```
$ cd ~
$ mkdir my-cluster
$ cd my-cluster
```

**注：若安装ceph后遇到麻烦可以使用以下命令进行清除包和配置：**

```
// 删除安装包
$ ceph-deploy purge admin-node node1 node2 node3

// 清除配置
$ ceph-deploy purgedata admin-node node1 node2 node3
$ ceph-deploy forgetkeys
```

#### 2. 创建集群和监控节点

创建集群并初始化**监控节点**：

```
$ ceph-deploy new {initial-monitor-node(s)}
```

这里node1是monitor节点，所以执行：

```
$ ceph-deploy new ceph_node1
```

完成后，my-clster 下多了3个文件：`ceph.conf`、`ceph-deploy-ceph.log` 和 `ceph.mon.keyring`。

> - 问题：如果出现 "[ceph_deploy][ERROR ] RuntimeError: remote connection got closed, ensure `requiretty` is disabled for node1"，执行 sudo visudo 将 Defaults requiretty 注释掉。

#### 3. 修改配置文件

```
$ cat ceph.conf
```

内容如下：

```
[global]
fsid = 2539e16a-2b19-476d-8005-3d749e288583
mon_initial_members = ceph_node1
mon_host = 10.10.20.61
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
osd pool default size = 2
rbd_default_features = 1
```

把 Ceph 配置文件里的默认副本数从 3 改成 2 ，这样只有两个 OSD 也可以达到 active + clean 状态。把 osd pool default size = 2 加入 [global] 段：

```
$ sed -i '$a\osd pool default size = 2' ceph.conf
```

如果有多个网卡，
可以把 public network 写入 Ceph 配置文件的 [global] 段：

```
public network = {ip-address}/{netmask}
```

如果Ceph存储需要提供kubernetes使用的OSD，注意要设置

```sehll
rbd_default_features = 1
```

#### 4. 安装Ceph

在所有节点上安装ceph：

```
$ ceph-deploy install ceph_admin_node ceph_node1 ceph_node2 ceph_node3 ceph_node4
```

> - 问题：[ceph_deploy][ERROR ] RuntimeError: Failed to execute command: yum -y install epel-release
>
> 解决方法：
>
> ```
> sudo yum -y remove epel-release
> ```

#### 5. 配置初始 monitor(s)、并收集所有密钥

```
$ ceph-deploy mon create-initial
```

完成上述操作后，当前目录里应该会出现这些密钥环：

```
{cluster-name}.client.admin.keyring
{cluster-name}.bootstrap-osd.keyring
{cluster-name}.bootstrap-mds.keyring
{cluster-name}.bootstrap-rgw.keyring
```

#### 6. 添加OSD

1) 登录到 Ceph 节点、并给 OSD 守护进程创建一个目录，并添加权限。

```
$ ssh node1
$ sudo mkdir /var/local/osd1
$ sudo chmod 777 /var/local/osd1/
$ exit

$ ssh node2
$ sudo mkdir /var/local/osd2
$ sudo chmod 777 /var/local/osd2/
$ exit
```

注意，从最好将osd单独做一个分区，log单独做一个分区。其中osd分区使用xfs格式，这是ceph最佳推荐设置。

我是用了linux lvm，LV名称为/dev/data/osd

```shell
[root@localhost ceph]# mkfs.xfs -f /dev/data/osd
meta-data=/dev/data/osd          isize=256    agcount=4, agsize=32761600 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=0        finobt=0
data     =                       bsize=4096   blocks=131046400, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
log      =internal log           bsize=4096   blocks=63987, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@localhost ceph]# mount -t xfs /dev/data/osd /var/local/osd4
[root@localhost ceph]# df -Th
文件系统                类型      容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root xfs        50G  1.7G   49G    4% /
devtmpfs                devtmpfs  7.8G     0  7.8G    0% /dev
tmpfs                   tmpfs     7.8G     0  7.8G    0% /dev/shm
tmpfs                   tmpfs     7.8G  8.6M  7.8G    1% /run
tmpfs                   tmpfs     7.8G     0  7.8G    0% /sys/fs/cgroup
/dev/mapper/centos-home xfs        42G   33M   42G    1% /home
/dev/sda1               xfs       497M  124M  373M   25% /boot
tmpfs                   tmpfs     1.6G     0  1.6G    0% /run/user/0
/dev/mapper/data-osd    xfs       500G   33M  500G    1% /var/local/osd4
```



2) 然后，从管理节点执行 ceph-deploy 来准备 OSD 。

```
$ ceph-deploy osd prepare node2:/var/local/osd0 node3:/var/local/osd1
```

3) 最后，激活 OSD 。

```
$ ceph-deploy osd activate node2:/var/local/osd0 node3:/var/local/osd1
```