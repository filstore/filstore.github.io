---
title: "IPFS GC研究"
date: 2018-09-25 16:26:01+0800
summary: 本文主要通过实现，实现 IPFS 存储空间的 GC，从而避免配置的存储空间被耗尽。
tags: [ipfs]
---

众所周知 IPFS 节点（peer）在初始化以后，默认使用的存储空间是 10GB（ipfs config show 可以查看），但随着 peer 节点加入网络的时间越来越久，存储空间使用率会逐步增加，直到达到上限，此时该节点将不再提供存储功能（但还能提供老数据的网络请求服务）。

## GC 猜想

对于这种情况，其实 IPFS 是提供了 GC 机制的，这个可以从 ipfs config 的  `StorageGCWatermark` 参数得出。只是 GC 默认被关闭了，需要使用 `ipfs daemon --enable-gc` 来启动，我们还可以使用 `ipfs repo gc` 来手动 GC 。 

猜想： 存储空间主要使用在两个方面，主动添加的内容（pined content）和节点访问而 Cache 的内容，当执行 GC 的时候，只删除 Cache 部分的内容，而被 pin 的内容应该被保留。

## 猜想验证

实验环境：

- IPFS 私网搭建，参考[链接](https://zhuanlan.zhihu.com/p/35141862) 。
- 两台虚拟机。

此时两台虚拟机构建一个 IPFS 私网，它们分别为 ipfs01 和 ipfs02，数据只会在它们之间共享。

### 实验过程

- step 1. 在 ipfs01 添加一张照片，然后执行 `ipfs pin | grep $HASH` 查看是否已经 pined。

![gc01](/images/ipfs_gc01.png)

*Tips:  通过 ipfs add 命令的时候，内容会直接被 pined。*

- step 2. 在 ipfs02 中通过命令 `ipfs cat $HASH > cat.jpg`  来下载该文件，然后执行 `ipfs pin | grep $HASH` 查看是否 pined。

![gc02](/images/ipfs_gc02.png)

通过 ipfs cat 把 ipfs01 中的 cat.jpg 下载到了 ipfs02 ，我们可以通过浏览器查看其内容：

![gc03](/images/ipfs_gc03.png)

*Tips:  内容只是被缓存，并没有 pined。*

- step3 : 将 ipfs01 中的 $HASH unpin, 并在本地执行 `ipfs repo gc` ，可以看到 该 $HASH 被清除：

![gc04](/images/ipfs_gc04.png)

- step 4: 在 ipfs01 中重复执行 step 2 步骤：

![gc05](/images/ipfs_gc05.png)

*Tips:  可以看到 ipfs01 可以从 ipfs02 缓存节点中读取内容并保存。*

- step 5:  在两台机器中分别执行 `ipfs repo gc` 后，再执行 `ipfs cat $HASH`：

![gc06](/images/ipfs_gc06.png)

*Tips: 可以看到分别执行 gc 后，所有的缓存备份都被删除，故在全网中都无法搜索到内容，直到最后超时。*

## 结论：

通过实验，验证了我们的猜想：

- 首先 IPFS 存在 GC ，其次它只 GC  Cache 内容(即 unpin)。
- 因为默认 GC 不开启，此时 cached 和 pined 内容等效。