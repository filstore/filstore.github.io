---
title: "IPFS 二维码添加的问题"
date: 2018-08-09 21:01:37+0800
summary: 本文主要讨论如何将一个 IPFS 的二维码添加到一个 IPFS 博客中。
tags: [ipfs]
---

前段时间在知乎上看到「飞向未来」大大早期文章 [什么是IPFS?(三)](https://zhuanlan.zhihu.com/p/32651288) 中有这样一段描述：

![ipfs03.png](/images/ipfs/ipfs03.png)

但是真的是这样吗？难道我们真的没有办法在一个网站中插入对应二维码？

> 结论当然是可以，因为我们有 IPNS。

下面我将介绍具体的实现步骤：

### Step 1:  占坑 

创建一个目录，添加到 IPFS 网络，先做好 IPNS 映射，这个过程就相当于在逻辑上先定义一个网站。

```
$ mkdir demo
$ ipfs add -r demo

added QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn demo
 0 B / 68 B [---------------------------------------------------------------------------]   0.00%

$ ipfs name public QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn
Published to QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe: /ipfs/QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn
```

此时我们已经可以通过 ipns/QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe 来访问网站了。

### Step 2: 添加网页

在 demo 目录下创建 `a.html` 文件，并且在 `a.html` 中添加如下内容：

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>qrcode demo</title>
</head>

<body style="text-align:center;">
    <h1>this is a ipfs page with qrcode</h1><br/>
    <img src="a.png" />
</body>
</html>
```

### Step 3：添加二维码

随便找一个二维码生成器生成 `a.html` 对应 IPNS 二维码，并存放为 `demo/a.png` 文件。

![ipns01.png](/images/ipfs/ipns01.png)

此时 `demo` 目录结构是这样：

```bash
$ tree demo/

demo/
├── a.png
└── a.html

0 directories, 2 files
```

### Step 4: 更新并发布网站

依次执行：

```
$ ipfs add -r demo

added QmNU5XXUpDnAXv88nDTcafKFY3h5LhKLJnkf2XBBGtc8wW demo/a.html
added QmcZhDddySHAdaJa9zKBRkY9SBrDJTWsq5Fn5chCDPXz3j demo/a.png
added QmSUCYnZ6SrGaAs8gaAAmf5tgjYAGMKjwMX3r4k7FxvCx1 demo
 2.23 KiB / 2.36 KiB [==============================================================>---]  94.37%

$ ipfs name publish QmSUCYnZ6SrGaAs8gaAAmf5tgjYAGMKjwMX3r4k7FxvCx1
```

最后打开浏览器访问地址 https://ipfs.io/ipns/QmR94EL86DjAuRQDdYeShG84ahqH4M39VFD8PbToiVobRe/a.html 将看到刚发布的 `a.html` 页面，并包含对应二维码。

![ipns02.png](/images/ipfs/ipns02.png)

到此为止，这个先有鸡还是先有蛋的问题已被我们解决，通过此例可以让我们加深对 IPNS 的认识。