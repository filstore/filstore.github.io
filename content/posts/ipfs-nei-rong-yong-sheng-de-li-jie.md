---
title: "IPFS 内容永生的理解？"
date: 2019-01-18 18:18:50+0800
summary: 本文主要讨论 IPFS 内容永生的话题，到底存在 IPFS 网络中的内容是否可以永远存在，那我们又该注意什么呢？
tags: [ipfs]
---

## 前言

IPFS [白皮书](https://github.com/ipfs/papers/raw/master/ipfs-cap2pfs/ipfs-p2p-file-system.pdf)中提到， IPFS 甚至是一个 `Permanent` 的网络，那是否存储在 IPFS 网络中的内容也可以永生，永远不丢失呢？

显然不是，因为内容最终都需要存储在物理磁盘上，如果全网中能够提供该文件下载的节点都下线或者磁盘损坏（冷门数据，内容提供节点较少），那在整个网络中就再也无法下载完该文件的完整内容了。

今天我就遇到这样的问题，下面我会来具体讲讲整个过程，以及我对 IPFS 内容“永生”新的理解。

## 背景

我们[边缘存储](https://docs.jdcloud.com/cn/object-storage-service/edgestorage) 产品本周新上了一个功能，在进行任务迁移的时候可以显示进度百分比。

我在[IPFS 看片指南：几部IPFS网络中的电影](https://zhuanlan.zhihu.com/p/35342854)中找了一个叫做**勇往直前**的文件，它的  CID 是 `QmZRJevYhADpXmCGGF6eCcP1afNEYFahDW5jxje3iyyCJS`，大小为 1582221058 个字节（约 1.48GB）。

在迁移到京东对象存储的过程中，发现其进度一直停留在20%，重试多次依然如此，如下图：

![ipfs/content/01.jpg](/images/ipfs/content/01.jpg)

这到底是怎么回事呢？我决定弄个明白。

## 定位问题

#### Step1: 确定已迁移字节数

因为数据库记录了已迁移的字节数，根据任务 ID 很容易拿到结果是 `322699264` 字节。

#### Step2: 确定拉取失败的 CID

众所周知 IPFS 对象是一棵树状结构，它具有以下特点：

- 每个节点下面最多包含174个节点。
- 文件的数据都存在叶子节点上，每个叶子节点默认存储 256KB 字节。

所以**勇往直前**这个电影 IPFS 的组成是一棵三层的树，结构如下图：

![03.png](/images/ipfs/content/03.png)

其中根节点的 CID 是 `QmZRJe...CJS`；第二层除最后一个节点都存储了43.5MB内容、包含174个存储256KB内容的叶子节点。

因为 IPFS 内容的读取是顺序的，所以根据已经迁移的字节数很容易计算出失败的块在第二层和第三层的索引位置：

```
第二层节点位置： (322699264 / 45613056) + 1 = 8
第三层节点位置：(322699264 % 45613056) / 262144 + 1 = 14
```
备注： 数组下标 = 位置-1 

下面我们通过 IPFS 的命令找到对应的 CID。

a. 首先根据文件 Root CID 查询第二层的所有节点（Links）：

```
ipfs object get QmZRJevYhADpXmCGGF6eCcP1afNEYFahDW5jxje3iyyCJS
```
命令输出内容如下：

![ipfs/content/02.png](/images/ipfs/content/02.png)

所得结果中 Links[7] 的 CID 是 `QmcfWDphyxf92EUv8DrHxvPDRzejeWg192cADyjyfiq33b`

b. 继续查询第三层的所有节点（Links）:

```
ipfs object get QmcfWDphyxf92EUv8DrHxvPDRzejeWg192cADyjyfiq33b
```

命令输出内容如下：

![ipfs/content/04.png](/images/ipfs/content/04.png)

所得结果中 Links[13] 的 CID 是 `QmNMjt6k7qXLtiDBHzsbBzmUJ6WJ6Aqx6VP3Uwje6Crnta`，这个就是无法拉取到内容的起始 Block CID。 使用 `ipfs dht findprovs` 命令确认全网确实没有能够提供内容的节点。

它前一个块和后一个块的 CID 分别是 `QmfMcounLQZCXaaUESkCWxZXLnoSyXfurpqxayMJALdK9w` 和 `QmW48B3yM7KdB3HxDGWFy3XNY1Fi9rEETZLSrPtkMStEDw`，前一个块能够查询到内容提供者：

```
ipfs dht findprovs QmfMcounLQZCXaaUESkCWxZXLnoSyXfurpqxayMJALdK9w
```

命令输出内容如下：

![05.png](/images/ipfs/content/05.png)

后一个块也无法查询到内容提供者。为什么会出现这样的结果呢？我的猜测是这样：

- 该内容是分享在 IPFS 网络中的大文件（电影），大家在尝试播放观看该文件的时候，并没有将电影看完（缓存完全），最大缓存的位置就是拉取失败的块的前一个块的位置。
- 首次提供该内容的节点已下线或者内容清理，GC将原始内容清除，所以全网中就再也找不到该文件的全部内容了。

## 内容“永生”的理解

- IPFS 网络中的内容通常并不会永久能访问到。
- 使用 `ipfs dht findprovs` 命令查询到某个 CID 存在的节点，并不一定能够提供该文件的完整内容，它很有可能只是记录了该文件的元信息，真实的数据块（叶子节点）有损坏。
-  如果想要使自己的内容在 IPFS 网络中“永生”，一定永远至少存在一个能够提供该文件完整内容的节点，就是我们的产品，[京东云边缘存储](https://docs.jdcloud.com/cn/object-storage-service/edgestorage)存在的价值，有能力帮助你在 IPFS 网络中永久 Pin 住某个特定的文件。