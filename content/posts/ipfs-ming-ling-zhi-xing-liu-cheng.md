---
title: "IPFS 源码阅读 -- 命令执行流程"
date: 2019-03-01 20:07:52+0800
summary: 本文将带着大家一起通过源码来学习 IPFS 的命令执行过程，加深对 IPFS 的认识。
tags: [ipfs]
---

使用 IPFS 的命令（包括 daemon）会按照一个通用的流程来执行，下面我们就来看看整个命令执行过程。

## IPFS command 定义

Command 结构体是对 ipfs 命令的一个抽象，其中最核心的是  PreRun, Run, PostRun，通常执行的过程是先执行 PreRun -> go PostRun -> Run。

```go
// Command is a runnable command, with input arguments and options (flags).
// It can also have Subcommands, to group units of work into sets.
type Command struct {
   Options   []cmdkit.Option
   Arguments []cmdkit.Argument
   PreRun    func(req *Request, env Environment) error
 
   // Run is the function that processes the request to generate a response.
   // Note that when executing the command over the HTTP API you can only read
   // after writing when using multipart requests. The request body will not be
   // available for reading after the HTTP connection has been written to.
   Run      Function
   PostRun  PostRunMap
   Encoders EncoderMap
   Helptext cmdkit.HelpText
 
   // External denotes that a command is actually an external binary.
   // fewer checks and validations will be performed on such commands.
   External bool
 
   // Type describes the type of the output of the Command's Run Function.
   // In precise terms, the value of Type is an instance of the return type of
   // the Run Function.
   //
   // ie. If command Run returns &Block{}, then Command.Type == &Block{}
   Type        interface{}
   Subcommands map[string]*Command
}
```

## 执行流程

![cmd-01](/images/cmd-01.png)


流程说明：

1. 定义 buildEnv， makeExecutor 函数
2. 通过 cli.Run() 执行命令:  buildEnv() -> makeExecutor() 
3. 执行 Run 的过程中还会涉及到到配置和节点实例化

## oldcmds.Context 详解

一个 ipfs cmd 运行时会携带一个运行时的上下文，即一个  commands/request.go 中的 Context 对象，它定义如下：

```go
type Context struct {
   Online     bool
   ConfigRoot string
   ReqLog     *ReqLog
 
   config     *config.Config
   LoadConfig func(path string) (*config.Config, error)
 
   api           coreiface.CoreAPI
   node          *core.IpfsNode
   ConstructNode func() (*core.IpfsNode, error)
}
```

其中最关键的是 LoadConfig 和 ConstructNode，具体含义如下：

- LoadConfig: 加载配置文件
- ConstructNode: 节点构造函数

而在 cmd/ipfs/main.go 的 mainRet() 函数中，它的定义如下：

```go
buildEnv := func(ctx context.Context, req *cmds.Request) (cmds.Environment, error) {
   // 此处省略代码
   // this sets up the function that will initialize the node
   // this is so that we can construct the node lazily.
   return &oldcmds.Context{
      ConfigRoot: repoPath,
      LoadConfig: loadConfig,
      ReqLog:     &oldcmds.ReqLog{},
      ConstructNode: func() (n *core.IpfsNode, err error) {
         // 此处省略代码
         r, err := fsrepo.Open(repoPath)
         if err != nil { // repo is owned by the node
            return nil, err
         }
 
         // ok everything is good. set it on the invocation (for ownership)
         // and return it.
         n, err = core.NewNode(ctx, &core.BuildCfg{
            Repo: r,
         })
         // 此处省略代码
      },
   }, nil
}
```

## 配置实例化

go-ipfs-config/config.go 中的 Config 是整个 IPFS 的配置说明，它由上面提到的命令运行的上下文 Context 来管理。

- 定义:

```go
// Config is used to load ipfs config files.
type Config struct {
   Identity  Identity  // local node's peer identity
   Datastore Datastore // local node's storage
   Addresses Addresses // local node's addresses
   Mounts    Mounts    // local node's mount points
   Discovery Discovery // local node's discovery mechanisms
   Routing   Routing   // local node's routing settings
   Ipns      Ipns      // Ipns settings
   Bootstrap []string  // local nodes's bootstrap peer addresses
   Gateway   Gateway   // local node's gateway server options
   API       API       // local node's API settings
   Swarm     SwarmConfig
   Pubsub    PubsubConfig
 
   Reprovider   Reprovider
   Experimental Experiments
}
```

- 实例化：

调用 cmd.Context  中的 LoadConfig 函数（在 buildEnv 中有定义），实际调用的是 cmd/ipfs/main.go 中的 loadConfig() 函数，本质是一个简单的 JSON unmarshal 操作,具体过程如下：

![cmd-02](/images/cmd-02.png)

## Node 实例化

IPFS 运行时候需要实例化一个 core/core.go  IpfsNode 对象，它用来封装对 整个 IPFS 节点的管理。

- 定义：

```go
// IpfsNode is IPFS Core module. It represents an IPFS instance.
type IpfsNode struct {
 
   // Self
   Identity peer.ID // the local node's identity
 
   Repo repo.Repo
 
   // Local node
   Pinning         pin.Pinner // the pinning manager
   Mounts          Mounts     // current mount state, if any.
   PrivateKey      ic.PrivKey // the local node's private Key
   PNetFingerprint []byte     // fingerprint of private network
 
   // Services
   Peerstore       pstore.Peerstore     // storage for other Peer instances
   Blockstore      bstore.GCBlockstore  // the block store (lower level)
   Filestore       *filestore.Filestore // the filestore blockstore
   BaseBlocks      bstore.Blockstore    // the raw blockstore, no filestore wrapping
   GCLocker        bstore.GCLocker      // the locker used to protect the blockstore during gc
   Blocks          bserv.BlockService   // the block service, get/add blocks.
   DAG             ipld.DAGService      // the merkle dag service, get/add objects.
   Resolver        *resolver.Resolver   // the path resolution system
   Reporter        metrics.Reporter
   Discovery       discovery.Service
   FilesRoot       *mfs.Root
   RecordValidator record.Validator
 
   // Online
   PeerHost     p2phost.Host        // the network host (server+client)
   Bootstrapper io.Closer           // the periodic bootstrapper
   Routing      routing.IpfsRouting // the routing system. recommend ipfs-dht
   Exchange     exchange.Interface  // the block exchange + strategy (bitswap)
   Namesys      namesys.NameSystem  // the name system, resolves paths to hashes
   Reprovider   *rp.Reprovider      // the value reprovider system
   IpnsRepub    *ipnsrp.Republisher
 
   PubSub   *pubsub.PubSub
   PSRouter *psrouter.PubsubValueStore
   DHT      *dht.IpfsDHT
   P2P      *p2p.P2P
 
   proc goprocess.Process
   ctx  context.Context
 
   mode         mode
   localModeSet bool
}
```

- 实例化：

调用 cmd.Context  中的 ConstructNode 函数（在 buildEnv 中有定义），具体过程如下：

![cmd-03](/images/cmd-03.png)

可以看到，整个过程主要分为以下三步：

- FSRepo 初始化
- BuildCfg 初始化
- Node 的初始化

### FSRepo 初始化

- repo/fsrepo/fsrepo.go 中的 FSRepo 的定义：

```go
// FSRepo represents an IPFS FileSystem Repo. It is safe for use by multiple
// callers.
type FSRepo struct {
   // has Close been called already
   closed bool
   // path is the file-system path
   path string
   // lockfile is the file system lock to prevent others from opening
   // the same fsrepo path concurrently
   lockfile io.Closer
   config   *config.Config
   ds       repo.Datastore
   keystore keystore.Keystore
   filemgr  *filestore.FileManager
}
```

- 初始化流程：

![cmd-04](/images/cmd-04.png)

IPFS 整个数据存储都是在这步骤完成的，其中主要包含存储 blocks 块的 FlatfsDatastore 以及 KV 元信息存储的 LeveldsDatastore。

### Node 初始化设置

![cmd-05](/images/cmd-05.png)

到此，我们通过阅读 IPFS 源代码，基本了解了其 Command 的执行流程，也对 IPFS 的启动有了一个深入的了解。