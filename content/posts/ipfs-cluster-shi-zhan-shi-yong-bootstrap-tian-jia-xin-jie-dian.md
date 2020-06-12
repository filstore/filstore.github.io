---
title: "IPFS Cluster 实战 -- 使用 bootstrap 添加新节点"
date: 2019-03-23 22:08:42+0800
summary: 本文主要讨论 ipfs cluster 中添加节点的其中一种方式，即采用 bootstrap 动态添加。
tags: [ipf, cluster]
---

## 准备工作

-  安装并运行 IPFS
-  根据操作系统下载预编译好的 ipfs ipfs-cluster-service 和 ipfs-cluster-ctl，本文以 ubuntu 为例
-  将可执行文件 ipfs-cluster-service 和 ipfs-cluster-ctl 拷贝到 /usr/local/bin 目录下，使其可以当做全局工具使用

以上是前置准备工作，每一个节点都需要做好这些前置步骤。接下来要进行集群配置。


## Step 0: 运行第一个集群节点

运行 ` ipfs-cluster-service init ` 初始化第一个集群节点配置。

> 你也可以在运行上述命令之前手动生成和配置 CLUSTER_SECRET 变量，如下所示(假设当前是在 node0 机器上)：
```
node0 $ export CLUSTER_SECRET=$(od  -vN 32 -An -tx1 /dev/urandom | tr -d ' \n')
node0 $ echo $CLUSTER_SECRET
9a420ec947512b8836d8eb46e1c56fdb746ab8a78015b9821e6b46b38344038f
```

初始化完成后，在 ~/.ipfs-cluster/service.json 文件中有  secret 字段就是该集群的秘钥。其他节点加入集群需要有相同的秘钥。

接着把这个节点的 cluster daemon 服务启动：

```
node0> ipfs-cluster-service daemon
```

## Step 1: 添加新节点到集群

必须确保秘钥和 Step 0 中的秘钥一致：

```
node1> export CLUSTER_SECRET=<copy from node0>
node1> ipfs-cluster-service init
```

执行上述命令后，就完成了集群配置。接着使用 bootstrap 命令加入集群。

```
node1> ipfs-cluster-service daemon --bootstrap /ip4/<ip-of-node0>/tcp/9096/ipfs/<cluster-peerid-of-node0>
```

接着运行 ipfs-cluster-ctl peers ls 就可以看到集群中其他节点，示例如下：

```
node1 > ipfs-cluster-ctl peers ls
QmYFYwnFUkjFhJcSJJGN72wwedZnpQQ4aNpAtPZt8g5fCd | Sees 1 other peers
  > Addresses:
    - /ip4/127.0.0.1/tcp/10096/ipfs/QmYFYwnFUkjFhJcSJJGN72wwedZnpQQ4aNpAtPZt8g5fCd
    - /ip4/192.168.1.3/tcp/10096/ipfs/QmYFYwnFUkjFhJcSJJGN72wwedZnpQQ4aNpAtPZt8g5fCd
  > IPFS: Qmaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
    - /ip4/127.0.0.1/tcp/4001/ipfs/Qmaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
    - /ip4/192.168.1.3/tcp/4001/ipfs/Qmaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
QmZjSoXUQgJ9tutP1rXjjNYwTrRM9QPhmD9GHVjbtgWxEn | Sees 1 other peers
  > Addresses:
    - /ip4/127.0.0.1/tcp/9096/ipfs/QmZjSoXUQgJ9tutP1rXjjNYwTrRM9QPhmD9GHVjbtgWxEn
    - /ip4/192.168.1.2/tcp/9096/ipfs/QmZjSoXUQgJ9tutP1rXjjNYwTrRM9QPhmD9GHVjbtgWxEn
  > IPFS: Qmbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
    - /ip4/127.0.0.1/tcp/4001/ipfs/Qmbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
    - /ip4/192.168.1.2/tcp/4001/ipfs/Qmbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
```

## Step 2: 从集群移除节点

命令 `ipfs-cluster-ctl peers rm <peer_id>` 可以将节点移除，如：

```
node0> ipfs-cluster-ctl peers rm QmYFYwnFUkjFhJcSJJGN72wwedZnpQQ4aNpAtPZt8g5fCd
```

运行之后就会断开 node1 的连接并且自动关闭 node1 的 ipfs-cluster-daemon 服务。

## 两种集群启动方式比较

| 指标 | 方式一：预设置集群节点 | 方式二：动态添加新节点 |
| ------ | ------ | ------ |
| 适用场景 | 第一次启动集群，并且清楚知道每个集群节点信息，例如节点的 cluster id 等 | 需要添加节点到正在运行的集群之中 |
| 灵活性 | 较为笨拙，需要配置 init_peerset, peerstore 等繁琐的信息 | 较灵活，可以动态的加入新节点 | 


## 参考文献

https://cluster.ipfs.io/guides/quickstart/

