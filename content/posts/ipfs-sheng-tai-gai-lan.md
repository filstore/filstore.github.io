---
title: "IPFS 生态概览"
date: 2018-10-22 15:27:35+0800
summary: 要想全面了解 IPFS 就不得不提 Protocal labs，本文将介绍 Protocal 的使命以及正在从事的研发工作。
tags: [ipfs, filecoin]
---

要想全面了解 IPFS 就不得不提 [Protocal labs](https://protocol.ai/work/)，目前它主要的项目包括：

- [IPFS](https://ipfs.io/)
- [Filecoin](https://filecoin.io/)
- [libp2p](https://libp2p.io/)
- [IPLD](https://ipld.io/)
- [Multiformats](https://multiformats.io)

那为什么会有这些项目，这些项目产生的背景，以及最核心的功能又是什么？看完本篇概览，你就会找到答案。

## IPFS

> IPFS is a new protocol to decentralize the web. IPFS enables the creation of completely decentralized and distributed applications, using content addressing and digital signatures。

IPFS是一种去中心化网络的新协议。 IPFS支持使用内容寻址和数字签名创建完全去中心化的分布式应用程序。

可以看到， IPFS 主要作为一种网络协议，使得通过它构建完全去中心的分布式应用程序（DApp）成为可能（当然还有其他的方式，例如 ETH），但是就像传统中心化的应用，DApp 对于永久可靠性存储的需求一直都在，并且成为 DApp 能够爆发的关键一环，当然也直接决定了 IPFS 协议最终的成功。

虽然已经有不少去中心化存储网络(DSN)解决方案，但是介于方案的成熟度，性能，以及与 IPFS 整合能力各方面的考量，由 Protocal labs 自己来实现 Filecoin 的方案才是最靠谱的。这样也会大量利用到 IPFS 已有的技术，例如底层网络连接和数据交换（libp2p）以及去中心化数据的通用定义（IPLD）。

在 IPFS 技术栈中不得不提 ipfs-cluster, 它是 IPFS 的 Pinset 编排程序，通过它可以构建更大的容量的 IPFS PinSet 集群，利用它可以构建一个去中心化的永久存储系统。个人认为它和 Filecoin 关注的重点不同，因为 Filecoin 是在公链中构建带有存储激励的存储网络，而 ipfs-cluster 相当于通过自己可控的节点构建超大的 IPFS PinSet 集群，而且目前 ipfs-cluster 的成熟度高于 Filecoin。

## Filecoin

Filecoin is a cryptocurrency powered storage network. Miners earn Filecoin by providing open hard-drive space to the network, while users spend Filecoin to store their files encrypted in the decentralized network.(Filecoin 是一种加密货币驱动的存储网络。 矿工通过向网络提供开放的硬盘空间来获得Filecoin，而用户使用Filecoin来存储他们在分散网络中加密的文件。)

DApp 是需要大量永久性去中心化存储的，这一点 IPFS 无法满足，所以 Filecoin 孕育而生，而且通过 FileCoin + IPFS 可以很好的解决整个 DApp 的内容存储和分发问题。

## libp2p

libp2p is a modular networking stack. libp2p brings together a variety of transports and peer-to-peer protocols, making it easy for developers to build large, robust p2p networks. (libp2p 是一个模块化的网络技术栈。 libp2p汇集了各种传输和点对点协议，使开发人员可以轻松构建大型，强大的 p2p 网络)。

目前去中心化的网络都是基于 p2p 来构建的，例如 BTC, ETH 和 IPFS, 因为 IPFS 包含了一个强大的 p2p 网络，那么将它抽离出来,成为一个单独的项目，能够被其它开源项目所复用是非常有价值的。

## IPLD

IPLD is the data model for the Decentralized Web. It connects all data through cryptographic hashes, and makes it easy to traverse and link to.(IPLD是去中心化 Web 的数据模型。 它通过加密哈希连接所有数据，并使其易于遍历和链接)。

IPFS 属于去中心化的 Web 协议，那么底层就依赖 IPLD，这样的数据结构能够在各种去中心的网络中实现数据共享和互访，例如（IPFS 和 ETH，BTC 等）。

## Multiformats

The Multiformats Project is a collection of protocols to future-proof systems, today. Self-describing formats make your systems interoperable and upgradable.（Multiformats 项目是当今面向未来系统的协议集合。 自描述的格式使你的系统可互操作和可升级）。

Multiformats 主要强调其自描述性，故名思议可以通过格式就能知道其中含义，那么作为一组协议集合，它目前包含了：

- multihash：自描述的 hash，主要用于 IPFS 中对于内容加密 hash 的描述。
- multiaddr：自描述的网络地址，主要用于 libp2p 的网络地址描述。
- multibase： 自我识别的基础编码。
- multicodec： 自描述序列化。
- multistream： 字描述的流式网络协议。

## 总结

IPFS 的目标是要在去中心化网络中主流（甚至唯一）的网络协议（类似 Web 中的 HTTP 的地位），但只有 IPFS协议显然是不够的，去中心化应用（网络）的普及，需要整个生态的共同发展，所以才有了后来的去中心化的数据模型（IPLD），去中心化的字描述数据格式（Multiformats），去中心化的永久性存储（FileCoin) 以及快速构建大型 p2p 应用的 libp2p 。

 