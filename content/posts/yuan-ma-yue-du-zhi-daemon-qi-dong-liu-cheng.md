---
title: IPFS 源码阅读 -- daemon 启动流程
date: 2018-11-14 16:22:29+0800
summary: 本文将带着大家一起通过源码来学习 IFPS daemon 的启动流程，加深对 IPFS 的认识。
tags: [ipfs]
---

## daemon 的定义

ipfs daemon 命令的定义在 cmd/ipfs/daemon.go 中的 daemonCmd 变量，其中 Run 方法是 daemonFunc，daemonFunc 主要包含如下几个步骤：

- fsrepo.Open(): 初始化数据持久层，包括 blocks 和 DAG
- core.NewNode(ncfg): 节点初始化
- serveHTTPApi(): API HTTP 服务监听
- mountFuse(): mount 目录 with FUSE
- maybeRunGC(): 启动 Repo GC 任务监听
- serveHTTPGateway(): HTTP Gateway 代理服务坚听

## fsrepo.Open 详解

参考 [《IPFS 源码阅读 -- 命令执行流程》](http://localhost:1313/posts/ipfs-ming-ling-zhi-xing-liu-cheng/) 文章中的 FSRepo 初始化逻辑。

## core.NewNode 详解

在 daemonCmd 中 node BuildCfg 被覆盖了，其值为：

```go
// Start assembling node config
ncfg := &core.BuildCfg{
   Repo:                        repo,
   Permanent:                   true, // It is temporary way to signify that node is permanent
   Online:                      !offline,
   DisableEncryptedConnections: unencrypted,
   ExtraOpts: map[string]bool{
      "pubsub": pubsub,
      "ipnsps": ipnsps,
      "mplex":  mplex,
   },
   //TODO(Kubuxu): refactor Online vs Offline by adding Permanent vs Ephemeral
}
  
// with default dht config
ncfg.Routing = core.DHTOption
```

core.NewNode(ctx, ncfg) 具体过程可以参考 [《IPFS 源码阅读 -- 命令执行流程》](http://localhost:1313/posts/ipfs-ming-ling-zhi-xing-liu-cheng/) 中的 Node 初始化设置，其中 cfg.Online  = true, 所以会分别执行：

- n.startOnlineServices(): 通过 bitswap 构建 Exchange
- n.startLateOnlineServices(): 通过 routing 创建 Reprovider

## serveHTTPApi 详解

API  HTTP 服务监听，根据配置文件中的 Addresses.API 以及运行命令参数 --api 决定其监听地址。

## mountFuse 详解

当启动添加参数  --mount 的时候，会使用到 mount Fuse 的功能，最终调用的方式是 fuse/node/mount_unix.go 中的 doMount 方法，而该方法中分别执行 fuse/readonly/mount_unix.go 和 fuse/ipns/mount_unix.go 中的 Mount 方法。

## maybeRunGC 详解

当启动添加 --enable-gc 参数的时候，可以让 repo 启动执行 GC，对内容进行回收，本质执行的代码是 pin/gc/gc.go 的 GC 方法。具体步骤大致分为：

1. 标记不需要删除的 cid，即所有递归和直接 pinned , bestEffortRoots（白名单，例如filesRoot）, 以及 pinner 内部使用的 blocks。 
2. 扫描所有 blocks 中的所有key, 依次检测是否需要删除。
3. 如果需要删除，直接删除本地 /blocks/下的文件即可。

## serveHTTPGateway 详解

Gateway  HTTP 服务监听，根据配置文件中的 Addresses.Gateway 决定其监听地址，可以通过 gateway 服务，采用 http 方式访问 ipfs 网络中的内容。


