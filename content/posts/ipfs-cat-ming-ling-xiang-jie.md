---
title: "IPFS cat 命令详解"
date: 2019-01-18 16:10:41+0800
summary: 本文将介绍 cat 命令的基本使用，原理分析，以及测试示例，带领大家对 IPFS 文件的存储和读取有个大致的了解。
tags: [ipfs]
---

## 命令介绍

我们可以使用 `ipfs cat` 命令直接查看一个文件对象（object）的内容，也可以使用 IO 重定向 `>` 来进行下载，例如：

```bash
ipfs cat QmWBbKvLhVnkryKG6F5YdkcnoVahwD7Qi3CeJeZgM6Tq68 > 神秘巨星.mp4
```

备注： 资源来自[链接](https://zhuanlan.zhihu.com/p/35342854)。

`cat` 命令支持 -o 和 -l 参数，分别表示文件读取的偏移量（offset）以及读取文件内容的长度，例如：

```bash
ipfs cat QmWBbKvLhVnkryKG6F5YdkcnoVahwD7Qi3CeJeZgM6Tq68 -o 256 -l 10
```

注： 表示从文件 QmWBbKvLhVnkryKG6F5YdkcnoVahwD7Qi3CeJeZgM6Tq68 的第 256 个字节开始，查看10个字节的内容，这个功能很好用，我们可以用它来实现分片下载以及断点续传等功能。


## IPFS 文件组织方式

在具体讲解 cat 命令的实现逻辑之前，我们先来了解下 IPFS 文件对象的组织方式，其实 IPFS 对象是一棵树，每个节点下面最多包含174个 Links(字节点)。文件对象的节点主要分类两种(Data_File 和 Data_Raw), 只有最下面的叶子节点才会存储真实的数据（Data_Raw），其它节点只是数据节点的元信息，如图：

![object.png](/images/ipfs_cat01.png)

注： 红色（Data_File）、绿色（Data_Raw）。

我们可以 QmWBbKvLhVnkryKG6F5YdkcnoVahwD7Qi3CeJeZgM6Tq68 为例，通过 `ipfs object get` 命令来对照查看：

```bash
ipfs object get QmWBbKvLhVnkryKG6F5YdkcnoVahwD7Qi3CeJeZgM6Tq68

# output
{ 
   "Links":[ 
      { 
         "Name":"",
         "Hash":"QmbmtXF9hZ6hLEubdLFP78MSPTWxUe78sYwjpCyiEJat3m",
         "Size":45623854
      }, 
       ... 省略 32 行 ...
 
      { 
         "Name":"",
         "Hash":"QmXEqLu1hXe3MfDngRdV1yWmtFQbpVz1WyDHY17r6BbsYj",
         "Size":31327573
      }
   ],
   "Data":"\u0008\u0002\u0018..."
}
```

可以看到 `QmWBbKvLhVnkryKG6F5YdkcnoVahwD7Qi3CeJeZgM6Tq68` 总共包含了 34 个 Link ，其中前 33 个 Link 的数据大小都为 45623854 ，最后一个大小为 31327573。然后我们再递归调用 `ipfs object get` 分别获取所有 Link 的 内容，例如第一个 Link：

```bash
ipfs object get QmbmtXF9hZ6hLEubdLFP78MSPTWxUe78sYwjpCyiEJat3m

# output
{ 
   "Links":[ 
      { 
         "Name":"",
         "Hash":"QmUgjKGQqNA4sJWbao2nBiANs46RjLAPaQrGLPHpR6Mh59",
         "Size":262158
      },
      ... 省略 172 个...
      { 
         "Name":"",
         "Hash":"QmSDaeEJ9ZsEGaFNx4ZMTAVcv8ncgMURKTjBeJ3gfhqkX9",
         "Size":262158
      }
   ],
   "Data":"\u0008\u0002\u0018..."
}
```

可以看到 `QmbmtXF9hZ6hLEubdLFP78MSPTWxUe78sYwjpCyiEJat3m`  下面包含了 174 个文件，每个文件大小为 262158 字节，因为这些都是叶子节点，存储的都是真实的数据，而默认IPFS数据分片大小为 256KB，还添加了12个字节作为数据校准。再随意查看一个 Link ，可以看到它的实际内容：

```bash
ipfs object get QmUgjKGQqNA4sJWbao2nBiANs46RjLAPaQrGLPHpR6Mh59
```

![store.png](/images/ipfs_cat02.png)

到这一步，我们对IPFS文件对象的组织方式有了大概了解，那么接下来看看 cat 命令具体的实现逻辑。


## Cat 命令实现逻辑

cat 命令的实现逻辑大致分为三步：

### Step 1: 参数解析

获得有效的 IPFS 或者 IPNS 对象的访问路径，例如上面的 `QmWBbKvLhVnkryKG6F5YdkcnoVahwD7Qi3CeJeZgM6Tq68`
获得命令的运行参数，主要是 -o 和 -l

### Step 2: 递归下载 Block 内容

- 首先获取 root cid 的内容，任何一个 cid 的内容获取过程如下：

    - 根据 cid 还原为 Block 文件 path,  先在 $IPFS_PATH/blocks/ 目录中查看文件是否存在，如果存在读取文件内容并解析。
    - 如果没有找到，尝试使用 cid 的 v1 版本，重复 1 的过程，如果没有找到，再通过 BlockService 在 bitswap p2p 网络中寻找，如果找到，进行 3），没有找到进行4）。
    - 解析内容，转化为 一个 ipld.Node ，并存储到本地的 blocks  内容中，并更新本地 BlockStorage 缓存。
    -  退出任务，提示失败。

- 当获取 root cid 内容的时候， IPFS 会将它封装了一个 file 对象。

    - file 对象的读取是一个递归调用过程，直到内容读取完成或者对应长度。
    - file 中定义了一个 4096 的缓冲区，将 Block 中的内容顺序 flush 给客户端。
    - file 中的 Read 函数中定义了一个 precalcNextBuf 的函数，为了加快速度，该函数起了10个并发，可以同时从本地或者网络中准备好获取这10个Link 的内容到内存中（即使传入 -o、-l 参数），所以单个文件下载大约会消耗256KB * 10 = 2.5MB 内存。

- 根据输入参数 -o 和 -l 设置文件读取的游标位置，主要调用 file.Seek() 函数。

以上过程，我整理一个流程图，大致如下：

![catflow.png](/images/ipfs_cat03.png)

## 总结

- IPFS 文件可以使用 -o -l 实现 range 下载。
- IPFS 对于单个文件是顺序下载的，在下载过程中会提前准备好10个 Blocks 的内容，所以单个文件下载最多消耗256KB*10=2.5MB 内存。
- 下载消耗的总内存 = 总下载任务数 * 2.5MB。