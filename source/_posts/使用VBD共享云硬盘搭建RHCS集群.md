---
title: 使用VBD共享云硬盘搭建RHCS集群
date: 2020-02-21 14:35:49
tags:
---

# 简介
以操作系统为CentOS 6.5的云服务器为例，搭建RHCS（Red Hat Cluster Suite）集群系统，使用GFS2分布式文件系统，实现一个共享盘在多个云服务器之间可共享文件。

将共享云硬盘挂载给多台云服务器后，需要安装共享文件系统或类似的集群管理系统，才能实现在多台云服务器之间共享文件。
# 环境准备
本次搭建一共三个节点，一个管理节点(ecs-share-003)，两个业务节点(ecs-share-001、ecs-share-002)，都是Centos6.5的云服务器。
- 一块VBD型的共享云硬盘
- ecs-share-003:10.110.31.166
- ecs-share-001:10.110.31.29
- ecs-share-002:10.110.31.206

将共享云硬盘挂载到ecs-share-001和ecs-share-002两台云服务器上。

# 配置

- 1核CPU
- 2G内存
- Centos6.5
- 内网互通

# 搭建流程

1. 配置云服务器网络
2. 安装RHCS集群
3. 创建集群
4. 配置磁盘
5. 验证磁盘共享功能

# 配置云服务器网络

本操作在三个节点都要执行，现以ecs-share-001为例
```
[root@ecs-share-001 ~]# vi /etc/hosts

#在最后添加以下内容，保存后退出
10.110.31.29 ecs-share-001
10.110.31.206 ecs-share-002
10.110.31.166 ecs-share-003
```
```
[root@ecs-share-001 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.110.31.29 ecs-share-001
10.110.31.206 ecs-share-002
10.110.31.166 ecs-share-003

```
# 安装RHCS集群

为业务节点安装ricci软件，为管理节点安装luci软件。
- luci。RHCS集群管理工具的主控端，提供了管理RHCS集群的web页面，管理集群主要是通过跟集群中其他节点上的ricci通信来完成的。
- ricci。RHCS集群管理工具的受控端，安装在集群中的其他节点上，luci就是通过每一个节点上的ricci管理后端。

## 安装ricci

本操作在ecs-share-001和ecs-share-002上来完成，现以ecs-share-001为例。

下载并安装ricci软件包
```
[root@ecs-share-001 ~]# yum install ricci openais cman rgmanager lvm2-cluster gfs2-utils
```

关闭防火墙
```
[root@ecs-share-001 ~]# /etc/init.d/iptables stop
[root@ecs-share-001 ~]# chkconfig iptables off
```

暂停并关闭ACPI（Advanced Configuration and Power Interface）服务
```
[root@ecs-share-001 ~]# /etc/init.d/acpid stop
Stopping acpi daemon:                                      [  OK  ]
[root@ecs-share-001 ~]# chkconfig acpid off

```

设置ricci密码
```
[root@ecs-share-001 ~]# passwd ricci
Changing password for user ricci.
New password: 
BAD PASSWORD: it is too short
BAD PASSWORD: is too simple
Retype new password: 
passwd: all authentication tokens updated successfully.
```

启动ricci
```
[root@ecs-share-001 ~]# /etc/init.d/ricci start
Starting system message bus:                               [  OK  ]
Starting oddjobd:                                          [  OK  ]
generating SSL certificates...  done
Generating NSS database...  done
Starting ricci:                                            [  OK  ]
[root@ecs-share-001 ~]# chkconfig ricci on

```

查看云服务器挂载的磁盘
```
[root@ecs-share-001 ~]# fdisk -l

Disk /dev/vda: 42.9 GB, 42949672960 bytes
16 heads, 63 sectors/track, 83220 cylinders
Units = cylinders of 1008 * 512 = 516096 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000342fb

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *           3        1043      524288   83  Linux
Partition 1 does not end on cylinder boundary.
/dev/vda2            1043        5204     2097152   82  Linux swap / Solaris
Partition 2 does not end on cylinder boundary.
/dev/vda3            5204       83221    39320576   83  Linux
Partition 3 does not end on cylinder boundary.

Disk /dev/vdb: 67 MB, 67108864 bytes
16 heads, 63 sectors/track, 130 cylinders
Units = cylinders of 1008 * 512 = 516096 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

   Device Boot      Start         End      Blocks   Id  System

Disk /dev/vdc: 21.5 GB, 21474836480 bytes
16 heads, 63 sectors/track, 41610 cylinders
Units = cylinders of 1008 * 512 = 516096 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

```

新建磁盘挂载目录
```
[root@ecs-share-001 ~]# mkdir /mnt/gfs_vbd
```

为磁盘创建GFS2分布式文件系统
```
[root@ecs-share-001 ~]# mkfs.gfs2 -p lock_dlm -t mycluster:gfs_vbd -j 4 /dev/vdc
This will destroy any data on /dev/vdc.
It appears to contain: data

Are you sure you want to proceed? [y/n] y

Device:                    /dev/vdc
Blocksize:                 4096
Device Size                20.00 GB (5242880 blocks)
Filesystem Size:           20.00 GB (5242878 blocks)
Journals:                  4
Resource Groups:           80
Locking Protocol:          "lock_dlm"
Lock Table:                "mycluster:gfs_vbd"
UUID:                      868002c6-ede7-fc8f-d8b6-b2df3d42bae1
```

## 安装luci
本操作在管理节点ecs-share-003上完成。

下载并安装luci集群软件包
```
[root@ecs-share-003 ~]# yum install luci
```

关闭防火墙
```
[root@ecs-share-003 ~]# /etc/init.d/iptables stop
[root@ecs-share-003 ~]# chkconfig iptables off
```

启动luci
```
[root@ecs-share-003 ~]# /etc/init.d/luci start
Adding following auto-detected host IDs (IP addresses/domain names), corresponding to `ecs-share-003' address, to the configuration of self-managed certificate `/var/lib/luci/etc/cacert.config' (you can change them by editing `/var/lib/luci/etc/cacert.config', removing the generated certificate `/var/lib/luci/certs/host.pem' and restarting luci):
	(none suitable found, you can still do it manually as mentioned above)

Generating a 2048 bit RSA private key
writing new private key to '/var/lib/luci/certs/host.pem'
Starting saslauthd:                                        [  OK  ]
Start luci...                                              [  OK  ]
Point your web browser to https://ecs-share-003:8084 (or equivalent) to access luci

[root@ecs-share-003 ~]# chkconfig luci on
```

luci启动成功后，即可以通过管理RHCS集群的web页面来进行集群的相关配置。

# 创建集群
打开并登录管理web页面
- 地址：https://10.110.31.166:8084 
- 账号密码：登录云服务的root账号和密码

选择"Manage Cluster"，点击"Create"创建集群
- Cluser Name：自定义集群名称，例如，mycluster。
- Node Name：分别设置业务节点的名称，例如，ecs-share-0001。
- Paasword：安装ricci时设置的ricci密码。

添加完第一个节点的信息后，单击"Add another Node"添加第二个节点。具体参数如 图1所示。

图1 创建集群

![VBD_RHCS_01_Create_cluster](https://blog-1300855606.cos.ap-chengdu.myqcloud.com/VBD_RHCS/VBD_RHCS_01_Create_cluster.png)


# 配置磁盘
创建成功后，选择"Reources"，单击"Add"为集群新建磁盘资源。
- Name：自定义磁盘资源名称，例如，GFS_VBD。
- Mount Point：安装ricci中设置的磁盘挂载目录，例如，/mnt/gfs_vbd。
- Device，FS Label，or UUID：此处以填写安装ricci中磁盘设备名称为例，例如，/dev/vdc。
- Filesystem Type：此处选择安装ricci中设置的分布式文件系统，例如，GFS2。

具体参数如 图2所示。

图2 创建磁盘资源

![VBD_RHCS_02_Add_Resources](https://blog-1300855606.cos.ap-chengdu.myqcloud.com/VBD_RHCS/VBD_RHCS_02_Add_Resources.png)

创建成功后，如 图3所示。

图3 创建磁盘资源成功

![VBD_RHCS_03_Add_Resources_success](https://blog-1300855606.cos.ap-chengdu.myqcloud.com/VBD_RHCS/VBD_RHCS_03_Add_Resources_success.png)

然后，选择"Failover Domains"，单击"Add"为集群服务新建故障切换域。

本例添加了名称为gfs_failover的故障切换域，具体参数如 图4所示。

图4 创建故障切换域

![VBD_RHCS_04_Add_Failover_Domains](https://blog-1300855606.cos.ap-chengdu.myqcloud.com/VBD_RHCS/VBD_RHCS_04_Add_Failover_Domains.png)

创建成功后再选择"Service Groups"，单击"Add"为集群中的节点创建服务组。
- Service Name：自定义服务组的名称，例如，GFS_SG_VBD_1。
- Failover Domain：选择创建的故障切换域，例如，gfs_failover。
- Add Resource：选择创建的资源，例如，GFS_VBD。

本示例添加了名称为GFS_SG_VBD_1的服务组，并添加了GFS_VBD磁盘资源，具体参数如 图5和 图6所示。

图5 创建服务组

![VBD_RHCS_05_Create_SG](https://blog-1300855606.cos.ap-chengdu.myqcloud.com/VBD_RHCS/VBD_RHCS_05_Create_SG.png)

图6 添加GFS_VBD磁盘资源

![VBD_RHCS_06_Add_GFS_VBD](https://blog-1300855606.cos.ap-chengdu.myqcloud.com/VBD_RHCS/VBD_RHCS_06_Add_GFS_VBD.png)

单击"submit"后，选择start on ecs-share-001节点。按同样的操作也为ecs-share-002创建服务组。两个服务组都创建成功后，如 图7 所示。

图7 创建两个服务组成功

![VBD_RHCS_07_Create_SG_success](https://blog-1300855606.cos.ap-chengdu.myqcloud.com/VBD_RHCS/VBD_RHCS_07_Create_SG_success.png)

服务组创建完成后，分别查看云服务器ecs-share-001和ecs-share-002的集群状况。以ecs-share-001为例。

```
[root@ecs-share-001 ~]# clustat
Cluster Status for mycluster @ Wed Feb 19 04:19:41 2020
Member Status: Quorate

 Member Name                                            ID   Status
 ------ ----                                            ---- ------
 ecs-share-001                                              1 Online, Local, rgmanager
 ecs-share-002                                              2 Online, rgmanager

 Service Name                                  Owner (Last)                                  State         
 ------- ----                                  ----- ------                                  -----         
 service:GFS_SG_VBD_1                          ecs-share-001                                 started       
 service:GFS_SG_VBD_2                          ecs-share-002                                 started
```

查看磁盘分区及挂载信息。
```
[root@ecs-share-001 ~]# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda3        37G  1.3G   34G   4% /
tmpfs           939M   32M  908M   4% /dev/shm
/dev/vda1       504M   62M  417M  13% /boot
/dev/vdc         20G  518M   20G   3% /mnt/gfs_vbd
```
其中，“/dev/vdc”即为共享盘设备名，分区成功并已挂载至“/mnt/gfs_vbd”目录下。
> 注意：如果执行完df -h命令发现分区表里没有挂载的共享盘，请执行“mount 磁盘设备名称 挂载目录” 命令重新挂载磁盘，例如，mount /dev/vdc /mnt/gfs_vbd，挂载后便可与其他云服务器同步使用该共享盘。

# 验证磁盘共享功能
## ecs-share-001写入内容
使用root用户登录云服务器ecs-share-001，进入“/mnt/gfs_vbd/”目录。
```
[root@ecs-share-001 ~]# cd /mnt/gfs_vbd/
```
新建“testshare”文件，并写入以下内容。
```
[root@ecs-share-001 gfs_vbd]# vi testshare
001 write

[root@ecs-share-001 gfs_vbd]# cat testshare 
001 write
```
## ecs-share-002查看内容，然后修改内容
使用root用户登录云服务器ecs-share-002。验证在ecs-share-002的磁盘挂载目录“/mnt/gfs_vbd”能否看到“testshare”文件中的内容
```
[root@ecs-share-002 ~]# cd /mnt/gfs_vbd/
[root@ecs-share-002 gfs_vbd]# ls
testshare
[root@ecs-share-002 gfs_vbd]# cat testshare 
001 write
```
从显示结果中，可以看到内容同步成功。

然后修改testshare文件，写入以下内容
```
[root@ecs-share-002 gfs_vbd]# vi testshare
#在最后添加以下内容，保存后退出
002 write

[root@ecs-share-002 gfs_vbd]# cat testshare 
001 write
002 write
```
## ecs-share-001查看内容改动
使用root用户登录云服务器ecs-share-001，查看testshare文件中的内容
```
[root@ecs-share-001 gfs_vbd]# cat testshare 
001 write
002 write
```
可以看到，内容修改同步成功。