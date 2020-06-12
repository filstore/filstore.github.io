---
title: "IPFS 源码阅读 -- add 命令详解"
date: 2019-03-10 20:46:33+0800
summary: 本文将通过源码的学习，带领大家深入了解 IFPS 的 add 命令的执行逻辑。
tags: [ipfs]
---

ipfs add 命令的入口命令是 core/commands/add.go 中的 AddCmd ，其中 PreRun 负责参数解析，Run 是真正的 add 操作的执行体，PostRun 是 add 操作的结果返回输出。

## 前言

add 命令属于 CoreAPI ，而 CoreAPI 的定义（core/coreapi/coreapi.go）如下：

```go
type CoreAPI struct {
   node *core.IpfsNode
   dag  ipld.DAGService
}
```

可以看到整个 CoreAPI 主要包含 IPFSNode 对象和 DAGService，它们分别表示 IPFS  节点状态以及 IPFS Merkle DAG 状态维护的服务。

因为所有核心命令其实是 CoreAPI 一个具体的实现，比如 add 命令就是 core/coreapi/unixfs.go 中的 UnixfsAPI 来负责的。

## UnixfsAPI 详解

UnixfsAPI 接口定义可以查看 core/coreapi/interface/unixfs.go，其中：

- Add 方法：将一个 I/O reader 对象转化为 merkledag 节点，并且保存到 blockstore中，并返回 merkle 节点的 key。
- Get 方法： 返回指向一个特定的 path 的只读的 fd。
- Ls 方法： 返回一个目录下的所有 links。

## Add 方法详解

当启动 daemon 的时候，本地执行 ipfs add 其实是通过 http 代理，将请求发送到 daemon 服务，此时真正执行的是 daemon 命令中启动的 IFPS Node, 当然我们也可以在服务没有启动的时候执行 ipfs add , 此时相当于使用的是 offlineExchange。

实际执行 Add 方法首先会声明一个 Adder 对象，定义如下：

```go
// Adder holds the switches passed to the `add` command.
type Adder struct {
   ctx        context.Context
   pinning    pin.Pinner
   blockstore bstore.GCBlockstore
   dagService ipld.DAGService
   bufferedDS *ipld.BufferedDAG
   Out        chan<- interface{}
   Progress   bool
   Hidden     bool
   Pin        bool
   Trickle    bool
   RawLeaves  bool
   Silent     bool
   Wrap       bool
   Name       string
   NoCopy     bool
   Chunker    string
   root       ipld.Node
   mroot      *mfs.Root
   unlocker   bstore.Unlocker
   tempRoot   cid.Cid
   CidBuilder cid.Builder
   liveNodes  uint64
}
```

然后再调用 adder的 AddAllAndPin 方法。

## 数据的存储

### 创建 merkle DAG

coreunix/add.go add首先构造chunker（数据流切分的splitter），然后根据数据流自底向上构造merkle DAG（代码位于balanced/builder.go Layout方法），详细如下：

![add-command.png](/images/add-command.png)

步骤如下：

- 首先创建根节点
- 如果有数据剩余则继续添加节点，同时把根节点挂入左子树
- 填充未满的节点

```bash
//      `Layout` creates a root and hands it off to be filled:
//
//             +-------------+
//             |   Root 1    |
//             +-------------+
//                    |
//       ( fillNodeRec fills in the )
//       ( chunks on the root.      )
//                    |
//             +------+------+
//             |             |
//        + - - - - +   + - - - - +
//        | Chunk 1 |   | Chunk 2 |
//        + - - - - +   + - - - - +
//
//                           ↓
//      When the root is full but there's more data...
//                           ↓
//
//             +-------------+
//             |   Root 1    |
//             +-------------+
//                    |
//             +------+------+
//             |             |
//        +=========+   +=========+   + - - - - +
//        | Chunk 1 |   | Chunk 2 |   | Chunk 3 |
//        +=========+   +=========+   + - - - - +
//
//                           ↓
//      ...Layout's job is to create a new root.
//                           ↓
//
//                            +-------------+
//                            |   Root 2    |
//                            +-------------+
//                                  |
//                    +-------------+ - - - - - - - - +
//                    |                               |
//             +-------------+            ( fillNodeRec creates the )
//             |   Node 1    |            ( branch that connects    )
//             +-------------+            ( "Root 2" to "Chunk 3."  )
//                    |                               |
//             +------+------+             + - - - - -+
//             |             |             |
//        +=========+   +=========+   + - - - - +
//        | Chunk 1 |   | Chunk 2 |   | Chunk 3 |
//        +=========+   +=========+   + - - - - +
```

### 数据持久化

coreunix/add.go Finalized() 调用file.go Flush方法进行数据持久化到磁盘，其中经过一些列的转化例如：shard文件到对应的目录，cid到key的转化等，最终调用flatfs.go Put方法进行落盘（首先把数据生成到临时文件，之后进行mv操作）

![add-command-02.png](/images/add-command-02.png)

![add-command-03.png](/images/add-command-03.png)

flatfs 文件系统来自于配置文件的datastore->Spec->mounts。