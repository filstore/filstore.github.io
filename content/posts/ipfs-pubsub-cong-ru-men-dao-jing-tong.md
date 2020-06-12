---
title: "IPFS PubSub 从入门到精通"
date: 2019-02-28 17:29:43+0800
summary: 本篇主要介绍 IPFS PubSub 的基本用法，原理，以及使用注意事项。
tags: [ipfs]
---

IPFS 有个叫 PubSub 的功能模块，能够实现 IPFS 网络中任意消息的订阅和发布，我们先来看个简单的例子。

### PubSub 示例

Step 1: 开启 pubsub 功能

因为 pubsub 属于实验功能，所以我们需要以 `--enable-pubsub-experiment` 参数来运行，我们在本地启动三个IPFS节点，并相互连接：

```
$ cd ~/ipfstest/node1
$ ipfs daemon --enable-pubsub-experiment
Initializing daemon...

$ cd ~/ipfstest/node2
$ ipfs daemon --enable-pubsub-experiment
Initializing daemon...

$ cd ~/ipfstest/node3
$ ipfs daemon --enable-pubsub-experiment
Initializing daemon...
```

三个节点的 ID 分别为：
- node1: QmaEhAQ3DHC9NyEFTQfd9q4V3FVp9hUt5NCbnFq6bWX4Dk
- node2: QmRYgkFHhMLhDXKd33n178MNuPHSr8DbBE4UPDHjyoz3BZ
- node3: QmXsSe9HsTGrmotAwZ3YHPotPgFFpRHwjwfGLpzCcxsEPx

此时通过任意一个节点可以看到另外两个节点，例如 node1：

```
$ ipfs swarm peers

/ip4/10.12.220.142/tcp/4002/ipfs/QmXsSe9HsTGrmotAwZ3YHPotPgFFpRHwjwfGLpzCcxsEPx
/ip4/10.12.220.142/tcp/4011/ipfs/QmRYgkFHhMLhDXKd33n178MNuPHSr8DbBE4UPDHjyoz3BZ
```

Step 2: 订阅主题：

分别让 node1 和 node2 订阅主题 `my-topic`

```
$ cd ~/ipfstest/node1
ipfs pubsub sub my-topic

$ cd ~/ipfstest/node2
ipfs pubsub sub my-topic
```
此时，node1 和 node2 的终端会卡住，等到接受新消息:

![pubsub-01](/images/pubsub/01.png)

Step 3: 发送和接收消息：

使用 node3 向主题 `my-topic` 发送消息：

```
$ cd ~/ipfstest/node3
$ ipfs pubsub pub my-topic 'hello world'
```

此时，node1 和 node2  将收到 `hello world` 消息:
![pubsub-02](/images/pubsub/02.png)

### PubSub 命令详解

我们可以使用 `ipfs pubsub --help` 来查看 PubSub 的所有命令和参数。

### ls 命令

`ls` 命令可以列出本节点订阅的所有主题。

```
$ cd ~/ipfstest/node1
$ ipfs pubsub ls
my-topic

$ cd ~/ipfstest/node2
$ ipfs pubsub ls
my-topic
```

### peers 命令

`peers` 命令可以列出与本节点 PubSub 相连的节点，默认列出所有节点，可以通过参数过滤定阅特定主题节点。

```
$ cd ~/ipfstest/node1
$ ipfs pubsub peers
QmRYgkFHhMLhDXKd33n178MNuPHSr8DbBE4UPDHjyoz3BZ
QmXsSe9HsTGrmotAwZ3YHPotPgFFpRHwjwfGLpzCcxsEPx

$ ipfs pubsub peers my-topic
QmRYgkFHhMLhDXKd33n178MNuPHSr8DbBE4UPDHjyoz3BZ

$ ipfs pubsub peers temp
(空)
```

### sub 命令

可以通过 `sub` 命令订阅一个主题，该命令支持 `--discover` 参数，表示允许通过 IPFS 网络搜素该主题的订阅节点，并与其建立连接。

```
$ ipfs pubsub sub my-topic
$ ipfs pubsub sub my-topic --discover
```

### pub 命令

可以通过 `pub` 向一个主题发布消息，该节点可以将消息通知到所有连接并订阅该主题的节点。

```
$ ipfs pubsub pub my-topic 'some message'
```

### PubSub 实现原理

这里我们主要了解 PubSub 的 sub 和 pub 命令的原理。首先我们来看看 PubSub 结构体的定义：

```golang
// PubSub is the implementation of the pubsub system.
type PubSub struct {
    // atomic counter for seqnos
    // NOTE: Must be declared at the top of the struct as we perform atomic
    // operations on this field.
    //
    // See: https://golang.org/pkg/sync/atomic/#pkg-note-BUG
    // The set of topics we are subscribed to
    myTopics map[string]map[*Subscription]struct{}
    // topics tracks which topics each of our peers are subscribed to
    topics map[string]map[peer.ID]struct{}
    peers  map[peer.ID]chan *RPC
```

它主要涉及三个关键字段，其它字段我已忽略，含义是：

- myTopics: 表示该节点订阅详情。
- topics: 收到的主题节点订阅情况。
- peers: 所有相连的节点。

下面我们来看看几个核心流程，是怎样通过该三个字段实现消息订阅和发布的：

状态 1: 所有三个节点两两相连

![node-01.png](/images/pubsub/node-01.png)

此时三个节点的 myTopics 和 topics 都为空，peers 是其它相连的两个节点。

状态 2: Node1 节点执行 `ipfs pubsub sub my-topic`：

![node-02.png](/images/pubsub/node-02.png)

- Node1 的 myTopics 会添加 `my-topic` 记录。
- Node1 会向相连的节点广播自己新增的订阅信息，所以 Node2 和 Node3 节点的 topics 会新增 `my-topic` 记录。

状态 3: Node2 节点执行 `ipfs pubsub pub my-topic 'some message'`：

![node-03.png](/images/pubsub/node-03.png)

- Node2 节点从本地的 topics["my-topic"] 中查到到订阅的节点列表为 Node1 节点。
- 发送消息 'some message' 给 Node1 节点。
- Node1 收到通知消息会匹配本地的 myTopics["my-topic"] 记录，输出到对应的 `*Subscription`中。

状态 4: 网络中存在隔离节点（没有直接相连，DHT Table 可以检索到）：

![node-04.png](/images/pubsub/node-04.png)

如果 Node4 也想发布内容到 `my-topic`，并且能够让 Node1 收到消息，需要使用到 `--discover`。

状态 5: 使用 `--discover` 参数，让 Node4 与 Node1 相连

![node-05.png](/images/pubsub/node-05.png)

- 使用 --discover 参数让 Node1 重新 sub `my-topic`。
- Node1 会在本地 blocks 文件夹下创建一个内容为 'floodsub:my-topic' 的 Block 文件，然后 IPFS 会自动将内容 Hash 发布到距离最近(XOR)的节点。
- Node4 也采用 --discover 参数 sub `my-topic`，此时它会根据 'floodsub:my-topic' 的内容 Hash，利用 `ipfs dht findprovs` 命令找到 Hash 的对应提供者。
- 尝试与所有提供者进行连接，然后将它们的节点添加到本地的 topics["my-topic"] 列表中。

状态 6: Node4 节点执行 `ipfs pubsub pub my-topic 'some message again'`：

![node-06.png](/images/pubsub/node-06.png)

- Node4 节点从本地的 topics["my-topic"] 中查到到订阅的节点列表为 Node1 节点。
- 发送消息 'some message again' 给 Node1 节点。
- Node1 收到通知消息会匹配本地的 myTopics["my-topic"] 记录，输出到对应的 `*Subscription`中。

## 总结

好了，到目前为止，我们学习到了 IPFS PubSub 的基本用法，参数详解以及具体实现逻辑。

如果想要在复杂网络中（一开始并不直接相连的节点）使用 PubSub 功能，需要通过 `--discover` 参数，将节点之间按照 topic 形成互联，从而实现任意地方消息订阅和发布。