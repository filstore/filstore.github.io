---
title: "使用 IPFS 搭建个人博客"
date: 2018-08-08 20:35:00+0800
summary: 本文主要讲解如何使用 IPFS 搭建个人博客，从而实现个人博客在网络中的“永生”。
tags: [ipfs]
---

本文主要讲解如何使用 IPFS 搭建个人博客，从而实现个人博客在网络中的“永生”。

主要内容：

- IPFS 简介
- IPFS 安装和基本使用
- IPFS 的内容发布
- IPNS 的内容发布

### IPFS 简介

[IPFS](https://ipfs.io/) 是 InterPlanetary File System 的简称，中文名叫星际文件系统，是一个旨在创建持久且分布式存储和共享文件的网络传输协议。

它是一种内容可寻址的对等超媒体分发协议，在 IPFS 网络中的所有节点将构成一个分布式文件系统，使用点对点的超媒体协议，从而让网络更快、更安全、更开放。

### IPFS 安装

访问页面 https://github.com/ipfs/go-ipfs/releases 下载对应版本，解压，运行安装文件即可。

```bash
$ wget https://github.com/ipfs/go-ipfs/releases/download/v0.4.17/go-ipfs_v0.4.17_darwin-amd64.tar.gz

$ tar xvf go-ipfs_v0.4.17_darwin-amd64.tar.gz
$ cd go-ipfs && ./install.sh
```

安装成功后，执行 `ipfs version` 你将看到类似内容：

```bash
$ ipfs version

ipfs version 0.4.17
```

注意： 

- 命令 `ipfs -h` 来查看更多命令
- 命令 `cat ~/.ipfs/config` 可以查看所有默认配置项


### 新建 IPFS 节点

- 运行 `ipfs init` 初始化节点

``` bash
$ ipfs init
initializing IPFS node at /Users/sjy/.ipfs
generating 2048-bit RSA keypair...done
peer identity: QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe
to get started, enter:

	ipfs cat /ipfs/QmS4ustL54uo8FzR9455qaxZwuMiUhyvMcX9Ba8nUH4uVv/readme
```

初始化成功后，可以使用 `ipfs cat` 命令来查看 IPFS 网络中的文件，例如：

```
$ ipfs cat /ipfs/QmS4ustL54uo8FzR9455qaxZwuMiUhyvMcX9Ba8nUH4uVv/readme
Hello and Welcome to IPFS!

██╗██████╗ ███████╗███████╗
██║██╔══██╗██╔════╝██╔════╝
██║██████╔╝█████╗  ███████╗
██║██╔═══╝ ██╔══╝  ╚════██║
██║██║     ██║     ███████║
╚═╝╚═╝     ╚═╝     ╚══════╝

If you're seeing this, you have successfully installed
IPFS and are now interfacing with the ipfs merkledag!

 -------------------------------------------------------
| Warning:                                              |
|   This is alpha software. Use at your own discretion! |
....
```

- 使用 `ipfs id` 查看节点基本信息

```
$ ipfs id

{
	"ID": "QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe",
	"PublicKey": "...",
	"Addresses": [
		"/ip4/127.0.0.1/tcp/4001/ipfs/QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe",
		"/ip4/100.100.58.79/tcp/4001/ipfs/QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe",
		"/ip4/100.100.6.124/tcp/4001/ipfs/QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe",
		"/ip6/::1/tcp/4001/ipfs/QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe",
		"/ip4/103.20.32.163/tcp/4001/ipfs/QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe"
	],
	"AgentVersion": "go-ipfs/0.4.17/",
	"ProtocolVersion": "ipfs/0.1.0"
}
```

- 运行 `ipfs daemon` 连接网络

```
$ ipfs daemon
Initializing daemon...
Swarm listening on /ip4/100.100.58.79/tcp/4001
Swarm listening on /ip4/100.100.6.124/tcp/4001
Swarm listening on /ip4/127.0.0.1/tcp/4001
Swarm listening on /ip6/::1/tcp/4001
Swarm listening on /p2p-circuit/ipfs/QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe
Swarm announcing /ip4/100.100.58.79/tcp/4001
Swarm announcing /ip4/100.100.6.124/tcp/4001
Swarm announcing /ip4/127.0.0.1/tcp/4001
Swarm announcing /ip6/::1/tcp/4001
Error: serveHTTPApi: manet.Listen(/ip4/127.0.0.1/tcp/5001) failed: listen tcp4 127.0.0.1:5001: bind: address already in use
Received interrupt signal, shutting down...
(Hit ctrl-c again to force-shutdown the daemon.)
```

有时你会遇到端口冲突的问题，可以使用 `vi ~/.ipfs/config` 修改默认配置来解决，相关内容为：

```
// from
"API": "/ip4/127.0.0.1/tcp/5001",
"Gateway": "/ip4/127.0.0.1/tcp/8080"

// to 
"API": "/ip4/127.0.0.1/tcp/9091",
"Gateway": "/ip4/127.0.0.1/tcp/9090"
```

保存修改后，再次运行 `ipfs daemon`，可以看到如下输出：

```
$ ipfs daemon

Initializing daemon...
Swarm listening on /ip4/100.100.58.79/tcp/4001
Swarm listening on /ip4/100.100.6.124/tcp/4001
Swarm listening on /ip4/127.0.0.1/tcp/4001
Swarm listening on /ip6/::1/tcp/4001
Swarm listening on /p2p-circuit/ipfs/QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe
Swarm announcing /ip4/100.100.58.79/tcp/4001
Swarm announcing /ip4/100.100.6.124/tcp/4001
Swarm announcing /ip4/127.0.0.1/tcp/4001
Swarm announcing /ip6/::1/tcp/4001
API server listening on /ip4/127.0.0.1/tcp/9091
Gateway (readonly) server listening on /ip4/127.0.0.1/tcp/9090
Daemon is ready
```

然后打开浏览器访问 `http://localhost:9091/webui` 可以看到如下内容：

![webui](/images/ipfs/webui.png)

这说明我们已经成功加入 IPFS 网络。

### 添加个人博客

长期以来我都是使用 Jekyll 来写博客，这里就用它来示范，当然大家也可以使用其它静态网站生成器，例如（hugo）。

- 使用 `jekyll build` 生成最新博客内容 
- 使用 `ipfs add -r _site` 添加博客站

```
$ ipfs add -r _site

added QmRum1Gg9DjYm8xosqaEd2YMdYZaVjUqifNC7xEQcxgCyn _site/404.html
added QmXMGUT9SMFa9nx4FcN5AQmPV4GBChAzUNzN2KUqJSQb5Q _site/CNAME
added Qme8R4NpB93Tcy55vhZhgReCkaehfhB6NGcMyHWFhNjGvJ _site/Gemfile
added QmPWR1QN9wVUtBGUm6PhwpwhuRHQXttrHufm6y8AYWzP4p _site/Gemfile.lock
added QmXWVVA17LnCZMqsTx75hZDeFhoJRJeKWEeU3dufQUkhmo _site/LICENSE
added Qmdvx2PCV3kQD8PvCFtsCi68mapc1fpkgS8FrQod42HxWh _site/README.md
added QmRHRq49njYPheLBMV8RTCnuDWcapkwRBhVNWXQPvThUyh _site/about.html
added QmPmmp8C2BaLvhFb2M43xdTgvUAtV5xREG2AAMiNpfgF2A _site/feed.xml
added QmPE53kbhLfSPUdAkieGR4NB6aA79ojkcQVB5WgksgiCoi _site/images/404.png
.....
added QmZw6t1FiWMMKgB5o4KohJ7wqHDvHRRBRvc8FmRwshejpR _site
58.51 MiB / 116.73 MiB [===========================================>------------------------------------------]  50.13% 00m03s
```

可以看到整个网站以 `_site` 为根目录，当前的版本为 `QmZw6t1FiWMMKgB5o4KohJ7wqHDvHRRBRvc8FmRwshejpR`，然后我们就可以通过链接 `https://ipfs.io/ipfs/QmZw6t1FiWMMKgB5o4KohJ7wqHDvHRRBRvc8FmRwshejpR` 在浏览器中查看到最新的博客内容了。

![ipfs01.jpg](/images/ipfs/ipfs01.jpg)

### 发布到 IPNS

当博客更新后，会生成新的 hash，要访问最新版本的内容就需要新生成的 hash，那如何使用一个固定的链接来访问最新版本的内容呢？

这就需要使用到 IPNS，IPNS 是 IPFS 的命名系统，它允许用户使用一个私有密钥来对哈希附加一个引用，使用一个公共密钥哈希（简称 pubkeyhash）表示你的网站的最新版本。

具体操作如下：

```
$ ipfs name publish QmZw6t1FiWMMKgB5o4KohJ7wqHDvHRRBRvc8FmRwshejpR

Published to QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe: /ipfs/QmZw6t1FiWMMKgB5o4KohJ7wqHDvHRRBRvc8FmRwshejpR
```

执行完命令后，我们就完成了博客和一个固定的 link 的绑定，我们可以使用 `ipfs name resolve xxx` 来查看固定 link 绑定的版本。

```
$ ipfs name resolve QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe

/ipfs/QmZw6t1FiWMMKgB5o4KohJ7wqHDvHRRBRvc8FmRwshejpR
```

然后我们通过 ipns + 生成的固定 hash （QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe）来访问博客了，例如:

![ipfs02.png](/images/ipfs/ipfs02.jpg)

当我们博客有更新的时候，只需要再次执行：

```
$ jekyll build
$ ipfs add -r _site
$ ipfs name publish <最新 hash>
```

并使用固定的 IPNS hash  `QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe` 来访问即可。

至此，我们基本完成了个人博客的 IPFS 发布，最后就剩的个人域名的绑定了，这个留到后面的文章来讲解。

### 总结

我们通过 IPFS 的安装、创建网络节点、添加网站、发布固定网站，学习了 IPFS 基础概念、基本操作、基于内容寻址的访问方式、以及 IPNS 和 IPFS 的对应关系。

可以看到使用 IPFS 搭建个人博客，乃至 WIKI 等内容网站是非常高效的，大家不妨尝试一下。