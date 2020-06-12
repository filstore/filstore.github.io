---
title: "Filecoin 是什么，看完这篇文章你就懂了"
date: 2019-02-15 14:36:16+0800
summary: 昨天是 2019 年的情人节（2/14），没想到 Filecoin 在 Twitter 放出一条重磅消息，宣布已将开源,那么本篇文章就带着大家来尝鲜，看看 Filecoin 的基本用法。
tags: [filecoin]
---

昨天是 2019 年的情人节（2/14），没想到 Filecoin 在 Twitter 放出一条重磅消息，宣布已将 [Filecoin 项目](https://github.com/filecoin-project/go-filecoin)开源，提供测试网络，详细报道可以参考官方[博文](https://filecoin.io/blog/opening-filecoin-project-repos/)。

![filecoin.png](/images/filecoin/filecoin.png)

### 什么是 Filecoin?

Filecoin 是一个去中心化的存储网络，能将海量用户的闲散存储资源充分利用起来，从而构建一套超低成本的可靠存储系统。

因为它没有中心，所以会用到区块链技术，只不过一般区块链使用的是算力挖矿的工作量证明（Proof-of-Work mining），而 Filecoin 使用的是存储工作量证明(Proof-of-Storage)。

### 与 IPFS 的关系

在谈 Filecoin 之前想必大家对 IPFS 已经有过了解，那二者有什么关系呢？

简单来说，IPFS 主要负责 P2P 网络中的内容寻值和内容传输，而 Filecoin 是内容永久存储的激励层，它们之间是相互补充的关系。

> IPFS addresses and moves content; Filecoin is the missing incentive layer。

### Filecoin 架构

我们可以把 Filecoin 简化为两层，一个是区块链，主要是记录全网状态（一个去中心化的状态机），包括所有钱包账户和余额，市场订单匹配记录。另外一个是存储的解决方案，主要包括副本策略（纠删码，多副本），端到端加密，存储时间和续费策略，存储市场匹配等。

它主要包括 4 种角色：

- 存储矿工（Storage miners）： 将自己的磁盘抵押出来，提供存储服务，类似云厂商的对象存储
- 检索矿工（Retrieval miners）：可以帮助用户快速的检索到内容，类似于传统意义的 CDN
- 存储客户端（Storage clients）：能够上传内容到 Filecoin，类似对象存储上传客户端
- 检索客户端（Retrieval clients）：能够从 Filecin 检索到内容，类似对象存储下载客户端

### 如何挖矿？

挖矿就是在 Filecoin 网络中获取虚拟货币（FIL），主要可以通过以下途径：

- 成为存储矿工，在市场中通过报价的方式出租存储空间，当匹配成功，将获取相应报酬
- 成为检索矿工，帮助内容下载客户端下载内容
- 成为数据修复矿工，帮助修复丢失或损坏的数据

目前版本主要实现的是存储矿工部分，其它以后会相继完成。

那如何成为存储矿工呢？

Step 1: 通过抵押资产创建存储矿工

例如： 创建一个矿工承诺10个扇区（目前每个256 MiB）和100 FIL作为抵押，消息气价格为0 FIL /unit，限制1000个气体单位。

```
go-filecoin miner create 10 100 --price=0 --limit=1000 --peerid `go-filecoin id | jq -r '.ID'`
```

这一步可能需要较长时间，主要确认账户余额是否足够抵押，如果创建成功将返回矿工地址。

Step 2: 启动矿工

```
go-filecoin mining start
```

Step 3: 创建存储订单请求

```
// 获取存储矿工地址
export MINER_ADDR=`go-filecoin config mining.minerAddress | tr -d \"` 

// 获取矿工拥有者（Filecoin 节点） 地址 
export MINER_OWNER_ADDR=`go-filecoin miner owner $MINER_ADDR`

// 创建一个报价请求，例如以0.000000001 FIL/字节/块的价格询问，对2880块有效，消息气价为0 FIL/unit，限制为1000气体单位
go-filecoin miner set-price --from=$MINER_OWNER_ADDR --miner=$MINER_ADDR --price=0 --limit=1000 0.000000001 2880 # output: CID of the ask

// 查询报价列表
go-filecoin client list-asks --enc=json | jq 
```

### 如何存取数据？

想使用 Filecoin 来存储数据，必须保证节点钱包具有充足的余额（FIL），这个等以后上线主网后可以通过交易市场购买，目前测试网络可以通过 [faucet](https://github.com/filecoin-project/go-filecoin/wiki/Getting-Started#get-fil-from-the-filecoin-faucet) 免费获取。

Step 1： 添加存储的内容到本地节点

这点很像 `ipfs add xx`， 例如：

```
echo "Hi my name is $USER"> hello.txt
export CID=`go-filecoin client import ./hello.txt`
go-filecoin client cat $CID
```

也可以通过文件导入：

```
export CID=`go-filecoin client import ~/Desktop/sample-data-master/camel.jpg`
go-filecoin client cat $CID > image.png && open image.png
```

Step 2: 查询存储市场，获取报价

可以通过命令查询存储矿工的报价列表：

```
go-filecoin client list-asks --enc=json | jq
```

结果类似：

```
{
  "Miner": "fcqxvnl37zdv8clc26j6r43zn8md7tc2mrfx77vru",
  "Price": "2.5",
  "Expiry": 588,
  "ID": 0,
  "Error": null
}
```

Step 3: 匹配报价

```
go-filecoin client propose-storage-deal <miner> <data> <ask> <duration>
```

其中：

- miner: 通过 `go-filecoin client list-asks` 出来的矿工地址
- data: 本地导入内容的 CID
- ask:  通过 `go-filecoin client list-asks` 列出来的 ID（通常都为 0）
-duration: 希望存储的时间周期，30秒为一个blocks, 所有一天就是 (2 blocks/min * 60 min/hr * 24 hr/day) = 2880 blocks。

Step 4: 矿工存储数据

当匹配订单(`propose-storage-deal`)创建成功后， 本地数据将自动通过 [bitswap](https://github.com/ipfs/specs/tree/master/bitswap) 传输到矿工节点。

Step 5: 客户端检索数据

客户端可以通过提交查询订单和数据恢复操作来下载数据，例如：


```
// 创建查询订单
go-filecoin client query-storage-deal <dealID>

// 当查询订单创建后，可以通过订单 minner 的地址以及想查询的内容 ID 来获取内容
go-filecoin retrieval-client retrieve-piece <minerAddress> <CID> # 可能需要1分钟
```

### 现状

当前 Filecoin 还是较为早期阶段，部分功能还在开发中，而且存在一定[安全问题](https://github.com/filecoin-project/go-filecoin/blob/master/KNOWN_ISSUES.md)。现在上线的是测试网络，主网预计 2019 Q3 上线。

核心功能进展：

- 存储矿工： 基本完成
- 检索矿工： 待开发
- 数据修复矿工： 待开发
- 客户端加密： 待开发
- 纠删码存储： 待开发

### 思考和展望

Filecoin 是完全去中心化的存储系统，优劣式明显。

劣势：

- 去中心化方案，引入区块链，必然会带来性能的降低（QPS 较低，确认时间长），这决定了它不能像标准对象存储（3副本）那样提供高速的上传下载能力。
- Filecoin 主要激励机制是靠虚拟货币，但它具有很大的价格波动性，而存储文件需要很长的时效性，币价的波动性会带来存储的不稳定性，矿工存储利益不平衡。

解决办法：

- Filecoin 为我们提供了将无限闲置资源利用起来的可能性，这在未来数据大爆炸时代是很有用的。虽然它存在上诉说的性能问题，但它可以用作低频存储（冷数据）而且这部分的存储量是巨大的，只要它具有超级吸引力的价格，而在热数据方面我们可以考虑通过 IPFS 来做缓存（类似 CDN），从而形成闭环。
- 对于币价的不稳定性这点，可以通过快速变现的方式减少持有的货币，或者降低订单存储结算周期，按照每周付费。

### 结语

在 Filecoin 之前已经有不少去中心化的存储，比如（Sia, StorJ），但它们都没有真正流行起来，很大原因就是价格上没有绝对优势。

Filecoin 能否流行起来，需要时间来证明，不过就目前来看，Filecoin 绝对是去中心存储市场中最耀眼的那一个。

参考链接：

- [How Filecoin Works](https://github.com/filecoin-project/go-filecoin/wiki/How-Filecoin-Works)
- [Mining Filecoin](https://github.com/filecoin-project/go-filecoin/wiki/Mining-Filecoin)
- [Storing on Filecoin](https://github.com/filecoin-project/go-filecoin/wiki/Storing-on-Filecoin)
 