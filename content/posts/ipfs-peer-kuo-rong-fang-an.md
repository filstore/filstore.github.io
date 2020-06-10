---
title: "IPFS peer 扩容方案"
date: 2018-10-10 19:44:33+0800
summary: 本文主要讨论使用 LVM 的技术实现 IPFS 单节点磁盘扩容的需求。
tags: [ipfs]
---

## 背景

我们都知道 IPFS 默认安装在 ~/.ipfs 目录，存储空间(StorageMax) 为 10GB， 这显然无法满足生产环境的需求，毕竟现在一个机械硬盘动则就是几个TB。那我们该如何对 IPFS 单一节点的容量进行扩充，做到同时使用多个硬盘？

## 环境

- Linux 系统
- 三个物理硬盘盘分别为 /dev/vda1(系统盘)，/dev/vdb1（100GB），/dev/vdc1(100GB)

## 方案

步骤1:  使用 LVM 将两个物理硬盘 /dev/vdb1 和 /dev/vdc1 组成同一物理卷组，最后再再物理卷组中创建逻辑卷组 （LV），具体命令如下：

```bash
# 创建物理卷（PV）
pvcreate /dev/sdb1
pvcreate /dev/sdc1
  
# 查看已经创建好的物理卷
pvdisplay
  
# 创建物理卷组
vgcreate IPFS /dev/sdb1 /dev/sdc1
  
#创建逻辑卷（LV）
lvcreate -L 195G IPFS -n IPFS
```

此时，现在你的逻辑卷应该已经在 /dev/mapper/ 和 */dev/YourVolumeGroupName* 中了。

步骤2:  挂载到目录 /IPFS ：

```bash
mkdir /IPFS
mkfs.ext4 /dev/mapper/VolGroup00-lvolhome
mount /dev/mapper/VolGroup00-lvolhome /IPFS
```

步骤3: 设置环境变量 IPFS_PATH, 指向挂在的目录：

```bash
export IPFS_PATH=/IPFS
```

步骤4:  ipfs 初始化、配置、启动应用

```bash
ipfs init
vi /IPFS/config # 修改 StorageMax 参数，设置大小为 190GB，具体按照自己的磁盘大小设置。
ipfs daemon
```

此时你将看到 ipfs 运行在了 /IPFS 目录，并且所有数据 blocks 存储在了 /IPFS/blocks ， 整个 ipfs 可以用的数据空间为 190 GB。

## 总结

我们通过 LVM 将多个物理硬盘连成物理硬盘组，再将磁盘组划分为一个大容量的逻辑卷，并将该卷挂载到我们的 IPFS_PATH 目录，从而使 IPFS 节点可以同时使用多个硬盘的存储空间。

 