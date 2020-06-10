---
title: "IPFS 三端可用性调研"
date: 2018-09-19 16:03:24+0800
summary: 本文主要讨论 IPFS 在浏览器，Android, iOS 运行的可行性和方案。
tags: [ipfs]
---

## 三端指什么

三端指浏览器，Android, iOS ，当前绝大多数互联网应用都跑在这三个终端上，所以 IPFS 在它们上的运行成熟度直接决定了 IPFS 最终流行程度，我们有必要提前了解并做好准备。

## 浏览器

- 状态： 可行
- 语言： JS
- https://github.com/ipfs/in-web-browsers
- https://github.com/ipfs/js-ipfs

## Android

- 状态： 可行
- 语言：kotlin
- https://github.com/ligi/ipfs-api-kotlin
- https://github.com/ligi/IPFSDroid

## iOS

- 状态： 可行
- 语言： swift
- https://github.com/ipfs/swift-ipfs-api

可以看到，社区很多人都在三端进行尝试，都希望 IPFS 能够在更多的平台上进行运行，尤其是移动端，这样可以大大扩充节点的数量。
