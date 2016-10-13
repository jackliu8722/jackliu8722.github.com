---
layout: post
title: "fabric共识机制分析1"
date: 2016-10-13 18:25
comments: true
categories: 区块链{blockchain}
---

共识机制的实现提供了一系列的接口，下列首先对接口进行一下简单的介绍，详见注释。

##1 ExecutionConsumer接口

```go
// ExecutionConsumer接口实现异步回调的功能
type ExecutionConsumer interface {
	Executed(tag interface{})                                // 事务成功执行完成后调用
	Committed(tag interface{}, target *pb.BlockchainInfo)    // 事务成功commit之后调用
	RolledBack(tag interface{})                              // 事务成功回滚之后调用
	StateUpdated(tag interface{}, target *pb.BlockchainInfo) // 状态转换完成之后调用,如果参数为nil，说明转换失败，并要提供新的target
}
```

该接口中用到了pb.BlockchainInfo，pb.BlockchainInfo是通过protobuf定义生成，表示区块的信息，其详细定义如下：

```go
// 区块链账本的信息：高度、当前区块的hash、前一个区块的hash
type BlockchainInfo struct {
	Height            uint64 `protobuf:"varint,1,opt,name=height" json:"height,omitempty"`
	CurrentBlockHash  []byte `protobuf:"bytes,2,opt,name=currentBlockHash,proto3" json:"currentBlockHash,omitempty"`
	PreviousBlockHash []byte `protobuf:"bytes,3,opt,name=previousBlockHash,proto3" json:"previousBlockHash,omitempty"`
}
```

##2 Consenter接口

```go
type Consenter interface {
	RecvMsg(msg *pb.Message, senderHandle *pb.PeerID) error
	ExecutionConsumer
}
```
Consenter接口用于从网络上接上消息，每个共识plugin必须实现该接口。当其它节点通过gRPC发送消息时，当前节点对接受到的消息依次调用RecvMsg方法进行处理，该方法需要提供两个参数：消息内容和发送节点的信息，对应的结构类分别为pb.Message和pb.PeerID，其详细定义见如下代码和注释：

```go
// 消息结构类
type Message struct {
	// 消息类型
	Type      Message_Type               `protobuf:"varint,1,opt,name=type,enum=protos.Message_Type" json:"type,omitempty"`
	// 时间
	Timestamp *google_protobuf.Timestamp `protobuf:"bytes,2,opt,name=timestamp" json:"timestamp,omitempty"`
	// 消息内容
	Payload   []byte                     `protobuf:"bytes,3,opt,name=payload,proto3" json:"payload,omitempty"`
	// 消息签名
	Signature []byte                     `protobuf:"bytes,4,opt,name=signature,proto3" json:"signature,omitempty"`
}

// 发送者节点信息
type PeerID struct {
	// 发送者名称
	Name string `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
}
```
Consenter接口还包含了ExecutionConsumer接口的方法，这里就不多介绍了。

##3 Inquirer接口
Inquirer接口主要是获取网络节点信息，详细定义如下：

```go
type Inquirer interface {
	// 获取节点信息
	GetNetworkInfo() (self *pb.PeerEndpoint, network []*pb.PeerEndpoint, err error)
	// 获取节点ID
	GetNetworkHandles() (self *pb.PeerID, network []*pb.PeerID, err error)
}
```
pg.PeerEndpoint存储了节点的信息，其定义如下：

```go
type PeerEndpoint struct {
	// 节点ID
	ID      *PeerID           `protobuf:"bytes,1,opt,name=ID" json:"ID,omitempty"`
	// 节点地址 ip:port
	Address string            `protobuf:"bytes,2,opt,name=address" json:"address,omitempty"`
	// 节点类型
	Type    PeerEndpoint_Type `protobuf:"varint,3,opt,name=type,enum=protos.PeerEndpoint_Type" json:"type,omitempty"`
	// 公钥ID
	PkiID   []byte            `protobuf:"bytes,4,opt,name=pkiID,proto3" json:"pkiID,omitempty"`
}
```

##4 Communicator接口
Communicator接口提供了向节点广播和单播消息的方法，具体定义如下：

```go
type Communicator interface {
	// 向网络中的其它节点广播消息
	Broadcast(msg *pb.Message, peerType pb.PeerEndpoint_Type) error
	// 向指定的节点发送消息
	Unicast(msg *pb.Message, receiverHandle *pb.PeerID) error
}
```

##5 NetworkStack接口
NetworkStack接口的方法来自于Communicator接口和Inquirer接口，主要是发送消息和获取消息的一个封装。

```go
type NetworkStack interface {
	Communicator
	Inquirer
}
```

##6 SecurityUtils接口
SecurityUtils接口用于对消息进行签名和验证。

```go
type SecurityUtils interface {
	Sign(msg []byte) ([]byte, error)
	Verify(peerID *pb.PeerID, signature []byte, message []byte) error
}
```

##7 ReadOnlyLedger接口
ReadOnlyLedger接口用于访问区块链的信息，只能进行读取不能进行写操作，详细定义如下：

```go
type ReadOnlyLedger interface {
	// 根据ID获取一个区块
	GetBlock(id uint64) (block *pb.Block, err error)
	// 获取区块的大小
	GetBlockchainSize() uint64
	// 获取区块的高度和hash等信息
	GetBlockchainInfo() *pb.BlockchainInfo
	// 获取区块的数据
	GetBlockchainInfoBlob() []byte
	// 获取区块的haader的meta信息
	GetBlockHeadMetadata() ([]byte, error)
}
```
pb.Block表示一个区块，包括版本、hash值、transaction等，具体这里就不多介绍了。

##8 LegacyExecutor接口
LegacyExecutor接口用于执行事务相关的保重，会修改账本的内容，提供的方法如下：

```go
type LegacyExecutor interface {
	// 开启一个事务，并提供事务id
	BeginTxBatch(id interface{}) error
	// 执行事务id
	ExecTxs(id interface{}, txs []*pb.Transaction) ([]byte, error)
	// 提交事务id
	CommitTxBatch(id interface{}, metadata []byte) (*pb.Block, error)
	// 回滚事务id
	RollbackTxBatch(id interface{}) error
	// 事务提交前的预览
	PreviewCommitTxBatch(id interface{}, metadata []byte) ([]byte, error)
}
```

##9 Executor接口
Executor接口最终会替换掉老的事务执行接口，也就是LegacyExecutor接口。调用LegacyExecutor接口进行事务操作时，为了避免资源竞争和损坏账本，必须与状态转移进行协调。（个人理解就是需要锁来进行协调）

```go
type Executor interface {
	// 获取所需要的资源
	Start()         
	// 释放所占有的资源                            
	Halt()        
	// 执行一系列的事务                           
	Execute(tag interface{}, txs []*pb.Transaction)
	// 事务执行完成之后进行提交
	Commit(tag interface{}, metadata []byte)  
	// 事务的回滚操作
	Rollback(tag interface{})  
	// 更新状态
	UpdateState(tag interface{}, target *pb.BlockchainInfo, peers []*pb.PeerID)
}
```

##10 LedgerManager接口
LedgerManager接口用于管理账本的状态。

```go
type LedgerManager interface {
	// 使账本为无效状态，在该状态应该拒绝查询
	InvalidateState()
	// 使用账本为有效状态，当前状态下应该恢复为可提供查询操作
	ValidateState()
}
```

##11 StatePersistor接口
StatePersistor接口用于保存共识状态，防止进程崩溃丢失状态。

```go
type StatePersistor interface {
	// 存储状态
	StoreState(key string, value []byte) error
	// 读取状态
	ReadState(key string) ([]byte, error)
	// 按前缀读取状态信息
	ReadStateSet(prefix string) (map[string][]byte, error)
	// 删除状态
	DelState(key string)
}
```
##12 Stack接口
Stack接口仅仅是对共识plugin需要实现的接口进行了一层封装，具体如下：

```go
type Stack interface {
	NetworkStack
	SecurityUtils
	Executor
	LegacyExecutor
	LedgerManager
	ReadOnlyLedger
	StatePersistor
}
```

以上是对consensus的接口的介绍，目前fabric提供了两种共识算法的实现，分别是noops和pbft，noops主要是针对单节点的实现，主要用于测试；而pbft是一种拜占庭容错算法，用于多节点的共识机制。

fabric多节点的共识机制使用的是pbft。
