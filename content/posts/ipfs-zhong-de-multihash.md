---
title: "IPFS 中的 multihash"
date: 2018-10-23 19:28:20+0800
summary: 本文主要向大家介绍 IPFS 中的 multihash 的结构和基本使用。
tags: [ipfs]
---

在 [IPFS 生态概览](/posts/ipfs-sheng-tai-gai-lan) 一文中我们已经提到了 multihash，那我们就来看看在 IPFS 中是如何使用它的？


## 为什么要用 multihash

IPFS 默认使用的 HASH 算法是 sha2-256, 虽然现在 sha2-256 还算安全，但这并不意味它不会被攻破，随着算力的发展，说不定哪天就被破解了，所以 IPFS 需要一种可升级具有自描述的 HASH 格式，这就是 multihash。

## multihash 格式 

multihash 的格式很简单，具体文档参见 [multiformats/multihash](https://github.com/multiformats/multihash)。它其实就是一个字符串，由三部分组成：HASH算法编码、HASH值的长度（字节数）、HASH 值 组成，形如：

```
<varint hash function code><varint digest size in bytes><hash function output>
```

例如：函数为 sha2-256，值为 32 位的 6cb42b2e29c28aeb143c3b9a0d9787f38ce2e5efea68ab01df6e121e4146ed8d 的 multihash 格式为：12206cb42b2e29c28aeb143c3b9a0d9787f38ce2e5efea68ab01df6e121e4146ed8d。

说明：

1.  这里使用的是 sha2-256，其 HASH 编码为 0x12 ，multihash 中所有 HASH 算法编码如下：

```go
// constants
const (
   ID         = 0x00
   SHA1       = 0x11
   SHA2_256   = 0x12
   SHA2_512   = 0x13
   SHA3_224   = 0x17
   SHA3_256   = 0x16
   SHA3_384   = 0x15
   SHA3_512   = 0x14
   SHA3       = SHA3_512
   KECCAK_224 = 0x1A
   KECCAK_256 = 0x1B
   KECCAK_384 = 0x1C
   KECCAK_512 = 0x1D
 
   SHAKE_128 = 0x18
   SHAKE_256 = 0x19
 
   BLAKE2B_MIN = 0xb201
   BLAKE2B_MAX = 0xb240
   BLAKE2S_MIN = 0xb241
   BLAKE2S_MAX = 0xb260
 
   DBL_SHA2_256 = 0x56
 
   MURMUR3 = 0x22
)
```

2. 6cb42b2e29c28aeb143c3b9a0d9787f38ce2e5efea68ab01df6e121e4146ed8d 看上去是 64 位，那是因为用了十六进制的表示方式，每个字符表示4个bit，加在一起就是 256 bit，也就是 32 字节，转化为 16 进制就是 `0x20`，故第二位就是 `20`。

## multihash 基本使用

multihash 有多种实现版本，这里我们使用 go-multihash 。

- go-multihash 的安装

```bash
go get github.com/multiformats/go-multihash/multihash
```

- 基本使用

通过 multihash -h 查看其所有参数选项，可以发现它默认使用的 HASH 算法是 sha2-256，编码格式为 base58。

```bash
usage: multihash [options] [FILE]
Print or check multihash checksums.
With no FILE, or when FILE is -, read standard input.
 
Options:
  -a string
        one of: blake2b-128, blake2b-224, blake2b-256, blake2b-384, blake2b-512, blake2s-256, dbl-sha2-256, id, keccak-224, keccak-256, keccak-384, keccak-512, murmur3, sha1, sha2-256, sha2-512, sha3, sha3-224, sha3-256, sha3-384, sha3-512, shake-128, shake-256 (shorthand) (default "sha2-256")
  -algorithm string
        one of: blake2b-128, blake2b-224, blake2b-256, blake2b-384, blake2b-512, blake2s-256, dbl-sha2-256, id, keccak-224, keccak-256, keccak-384, keccak-512, murmur3, sha1, sha2-256, sha2-512, sha3, sha3-224, sha3-256, sha3-384, sha3-512, shake-128, shake-256 (default "sha2-256")
  -c string
        check checksum matches (shorthand)
  -check string
        check checksum matches
  -e string
        one of: raw, hex, base58, base64 (shorthand) (default "base58")
  -encoding string
        one of: raw, hex, base58, base64 (default "base58")
  -h    display help message (shorthand)
  -help
        display help message
  -l int
        checksums length in bits (truncate). -1 is default (shorthand) (default -1)
  -length int
        checksums length in bits (truncate). -1 is default (default -1)
  -q    quiet output (no newline on checksum, no error text) (shorthand)
  -quiet
        quiet output (no newline on checksum, no error text)
```

下面我们尝试使用 multihash 对一个文件进行校验计算。首先新建一个 enp.txt 文件，输入内容为 “正道成功，EPN”，再执行 multihash epn.txt 命令。

```
$ echo "正道成功，EPN" > epn.txt
$ multihash epn.txt
$ QmbftC2aCSGeFLWu995pBGpMvQGgWoMXBxZG2JxDYd9GLP
```

## multihash 计算过程探究

- 首先对内容进行 sha2-256 HASH 计算。

```bash
$ shasum -a 256 epn.txt
$ c6153579704c93e8d7fbcfbb639b99213c0a6d8397facd0a58d2fc8f4d5dd2e0
```
- multihash 格式封装： 加上 sha2-256 的编码以及 HASH 长度得到的字符串为 1220c6153579704c93e8d7fbcfbb639b99213c0a6d8397facd0a58d2fc8f4d5dd2e0。
- base58 编码： base58 最早使用在 btc 钱包地址上，它相对于传统的 base64 而言，去除了某些不和谐，阅读不友好的字符，例如 +/-,O/0 等。

```bash
$ pip install base58
$ python
  
>>> import base58
>>> base58.b58encode_int(int("1220c6153579704c93e8d7fbcfbb639b99213c0a6d8397facd0a58d2fc8f4d5dd2e0", 16))
'QmbftC2aCSGeFLWu995pBGpMvQGgWoMXBxZG2JxDYd9GLP'
```

可以看到，通过 python 编码得到的 base58 结果和直接使用 go-multihash 结果一致，都是 QmbftC2aCSGeFLWu995pBGpMvQGgWoMXBxZG2JxDYd9GLP。

## multihash 与 IPFS

经常使用 IPFS 的人可能会发现其内容 HASH 都是以 Qm 开头，这和我们刚使用 go-multihash 计算的一致，这是因为 IPFS 直接使用了multihash 的默认选项，即 sha2-256 + base58。

我们尝试使用 ipfs 命令计算  epn.txt 的 HASH 值。

```bash
$ ipfs add epn.txt
added QmarcwWsQp5ew7jnFtqYTLgkWEWaMPde8gme5ApZQXvZkU epn.txt
```

你会发现使用 ipfs 计算得出的 HASH 和直接使用 go-multihash 计算的不同，这是因为 ipfs 对内容进行 sha2-256 计算的时候，会对内容进行一定的加工（首尾添加特定字节），这点可以使用 ipfs object get 来查看。

```bash
$ ipfs object get QmarcwWsQp5ew7jnFtqYTLgkWEWaMPde8gme5ApZQXvZkU
{"Links":[],"Data":"\u0008\u0002\u0012\u0013正道成功，EPN\n\u0018\u0013"}
```

我们可以使用 multihash 对 ipfs 加工后的内容重新进行计算： 

```bash
$ ipfs block get QmarcwWsQp5ew7jnFtqYTLgkWEWaMPde8gme5ApZQXvZkU > epn-ipfs.txt
$ multihash epn-ipfs.txt
QmarcwWsQp5ew7jnFtqYTLgkWEWaMPde8gme5ApZQXvZkU
```

可以看到此时计算出的 HASH 值与 IPFS 计算出的保持一致。

## 总结

我们通过实际例子对 multihash 的格式，计算过程有深刻认识，其次我们还可以发现 ipfs 与 multihash 之间的内在联系，总而言之 IPFS 内容 HASH 计算过程可以大致看成 内容封装 > hash2-256 计算 > multihash 格式封装 > base58 编码。

参考链接: https://zhuanlan.zhihu.com/p/43853989