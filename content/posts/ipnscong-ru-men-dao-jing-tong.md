---
title: "IPNS 从入门到精通"
date: 2019-02-20 17:17:12+0800
summary: 本篇主要介绍 IPFS 中一个重要模块 IPNS 的基本使用，原理，以及回答大家所关心的问题。
tags: [ipfs]
---

IPNS 是构建在 IPFS 之上的一套依赖 PKI的命名空间系统(namespace sytem， KV 存储空间)，其中 Key 是公钥，值是经过私钥签名的新值，所以网络中的其它节点可以根据公钥就能进行内容验证（防伪造）。 

### 为什么需要 IPNS

众所周知，我们可以通过 `/ipfs/QmS4ust...` 方式来访问一个对象，但由于 IPFS Hash 与内容一一对应（如果内容发生改变，Hash 值也会随之改变），所以我们访问新内容的时候需要知道新的 Hash， 这显然不够友好，那么我们是否可以通过一个固定的 Hash 来访问不同版本的内容呢？

要实现上述功能，我们可以使用 IPNS，下面我们就来看看实际的例子。

### IPNS 示例

- 添加内容

```bash
$ echo "hello, this IPNS example" > ipns.txt

$ ipfs add ipns.txt
added QmeVuuv7disk9Ri6n3QtTUCEW462UPZe3iWYFVNWNQwYxr ipns.txt
```

- 发布到 IPNS 中

```bash
$ ipfs name publish /ipfs/QmeVuuv7disk9Ri6n3QtTUCEW462UPZe3iWYFVNWNQwYxr
Published to QmbhgXW9LBpTcpEUerXTuoNZNd4AeHuEkb6iKx67KwSd9p: /ipfs/QmeVuuv7disk9Ri6n3QtTUCEW462UPZe3iWYFVNWNQwYxr
```

- 通过 IPNS 查询访问内容

```bash
$ ipfs cat /ipns/QmbhgXW9LBpTcpEUerXTuoNZNd4AeHuEkb6iKx67KwSd9p
hello, this IPNS example
```


- 修改内容，重复上述步骤

```bash
# 在 ipns.text 文件追加内容
$ echo "hello, this IPNS example" >> ipns.txt

# 将新文件添加到 IPFS 网络
$ ipfs add ipns.txt
added Qmak4GvXptsAwXLFnnSfLLhpv4PLNHPRgruueKeVTUFnXP ipns.txt

# 向网络发布最新的值
$ ipfs name publish Qmak4GvXptsAwXLFnnSfLLhpv4PLNHPRgruueKeVTUFnXP
Published to QmbhgXW9LBpTcpEUerXTuoNZNd4AeHuEkb6iKx67KwSd9p: /ipfs/Qmak4GvXptsAwXLFnnSfLLhpv4PLNHPRgruueKeVTUFnXP

#通过固定的 IPNS Key 查询最新的值
$ ipfs cat /ipns/QmbhgXW9LBpTcpEUerXTuoNZNd4AeHuEkb6iKx67KwSd9p
hello, this IPNS example
hello, this IPNS example
```

在上面的例子中，我们使用相同的 Key 先后映射 ipns.txt 文件的两个不同版本，而且使用 /ipns/KEY 的方式来获取对应内容。

> 它就像一个指针，固定的 Key 指向了一个具体的内容。

### IPNS 高级指南

- 本地 KPI 列表

```bash
$ ipfs key list 
self
```

默认使用 节点的 ID 作为 Key，并且标记为 self。

- 新增KPI密钥对

```
$ ipfs key gen ipns-example --type=rsa --size=2048
QmVsHo1xo32utEr8rsj3sjfx78WikZZ6sWGGNJTRFZu3jV

# 再次查看可以查看到新建的 `ipns-example` 
$ ipfs key list 
self
ipns-example
```

- 选择特定的 Key 来发布内容

```
$ ipfs name publish --key=ipns-example Qmak4GvXptsAwXLFnnSfLLhpv4PLNHPRgruueKeVTUFnXP
Published to QmVsHo1xo32utEr8rsj3sjfx78WikZZ6sWGGNJTRFZu3jV: /ipfs/Qmak4GvXptsAwXLFnnSfLLhpv4PLNHPRgruueKeVTUFnXP

# 使用新 Key 查询内容
$ ipfs cat /ipns/QmVsHo1xo32utEr8rsj3sjfx78WikZZ6sWGGNJTRFZu3jV
hello, this IPNS example
hello, this IPNS example
```

- 查询某个 Key 的映射内容（版本）

```
$ ipfs name resolve QmbhgXW9LBpTcpEUerXTuoNZNd4AeHuEkb6iKx67KwSd9p
/ipfs/Qmak4GvXptsAwXLFnnSfLLhpv4PLNHPRgruueKeVTUFnXP
```

> 我们可以通过创建多个 Key，每个 Key 映射一个目录，从而实现分组需求，可以用这个功能来实现单个节点托管多个静态网站。

### IPNS 实现原理

IPNS 本质是一个KV存储，当我们知道 Key 的时候，能够在 IPFS 网络中快速查询到对应的 Value，要实现这一步， IPFS 网络中就要存储这个 KV 映射信息，那 IPFS 是如何来实现的呢？

实现原理大致如图：

![ipns.jpg](/images/ipfs/ipns.jpg)

具体如下：

1. 通过 `ipfs name publish` 发布内容时候，IPFS 节点会先将 KV 数据存储到本地的 LevelDB 中，其中 Key 的格式类似 `ipns/Qmxxx`，默认存储时间为 24h，不过可以通过 -t 参数来设置。  
2. 在本地 DHT Table 中寻找特定数量的相邻节点，寻找的过程为首先通过 Key 与 Peer 节点 ID 进行 XOR 计算，找到对应桶中的节点。
3. 相邻节点收到发布的内容后，会做签名验证验证，然后存储到本地的 LevelDB 中，这步与 1 相同。
4. 当其它节点拿到 IPNS 的 Key 的时候，可以获取与 Key 相邻的节点，与第2步的相邻节点相似，然后向它们请求该 Key 的内容。
5. 相邻节点单收到 Key 查询请求的时候，会检查本地 LevelDB 是否包含该 Key，如果包含，返回其内容，否则返回空。
6. 根据解析到的 Key 对应的具体内容ID来进行内容交换。


### Q&A

a. 默认发布的 Key 过期问题？

可以通过定期 public 或者通过 -t 参数设置较长时间周期。

b. 如何保证存储的可靠性？

节点会向多节邻居发布映射关系，从而在部分节点下线的时候，也能在网络中查询到结果。

c. 数据一致性问题？

当邻居节点已经存储当前最新发布信息，此时下线，那么新的发布信息将丢失，从而导致本地存储的信息过期，当该节点重新上线后，当有节点来向它查询的时候，将会返回旧的结果。
