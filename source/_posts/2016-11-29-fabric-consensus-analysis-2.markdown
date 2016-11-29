---
layout: post
title: "fabric共识机制分析2"
date: 2016-11-29 11:56
comments: true
categories: 区块链{blockchain}
---

在[fabric 共识机制分析1](http://jackliu8722.github.io/blog/2016/10/13/fabric-consensus-analysis-1/)中我们介绍了实现共识相关的接口，本文主要介绍fabric中pbft的使用，pbft的上具体算法不是本文的重点，有兴趣可以去拜读pbft的论文。

##1 fabric如何配置使用pbft
我们知道，fabric目前支持两种共识算法：pbft和noops。当系统启动时怎么知道使用哪种共识算法呢？大家脑子转个0.1s就想到了，肯定是在配置文件中进行配置的呀。恭喜你，猜对了，那配置在哪里呢？首先我们在看看源文件fabric/consensus/conroller/conroller.go中的部分代码。

```go
func NewConsenter(stack consensus.Stack) consensus.Consenter {

	plugin := strings.ToLower(viper.GetString("peer.validator.consensus.plugin"))
	if plugin == "pbft" {
		logger.Infof("Creating consensus plugin %s", plugin)
		return pbft.GetPlugin(stack)
	}
	logger.Info("Creating default consensus plugin (noops)")
	return noops.GetNoops(stack)

}
```
从源码中可以看见，具体使用哪种共识算法通过viper获取，viper是fabric依赖第三方go库，这里不做过多的介绍。通过顺藤摸瓜，我们找到peer.validator.consensus.plugin在配置在fabric/peer/core.yaml文件中，大家可以自己去看看，这里就不介绍了。

##2 创建pbft核心引擎
从上面的代码中我们知道，当plugin为pbft时，调用了方法pbft.GetPlugin，我们看看这个方法做了些什么工作呢？在fabric/consensus/pbft/pbft.go文件找到相应的代码如下：

```go
var pluginInstance consensus.Consenter // 单例
......
func GetPlugin(c consensus.Stack) consensus.Consenter {
	if pluginInstance == nil {
		pluginInstance = New(c)
	}
	return pluginInstance
} 

func New(stack consensus.Stack) consensus.Consenter {
	handle, _, _ := stack.GetNetworkHandles()
	id, _ := getValidatorID(handle)

	switch strings.ToLower(config.GetString("general.mode")) {
	case "batch":
		return newObcBatch(id, config, stack)
	default:
		panic(fmt.Errorf("Invalid PBFT mode: %s", config.GetString("general.mode")))
	}
}
```
GetPlugin方法创建一个consensus.Consenter的单例，具体的实现调用了New方法，New方法最终委托给newObcBatch方法，newObcBatch方法主要是创建obcBatch对象，并设置相应的属性，我们来看看obcBatch对象的具体定义和newObcBatch的实现:

```go
type obcBatch struct {
	obcGeneric
	externalEventReceiver //consensus.Consenter接口的实现
	pbft        *pbftCore //pbft核心引擎
	broadcaster *broadcaster // 广播相关的对象

	batchSize        int    //batch的大小
	batchStore       []*Request //请求列表
	batchTimer       events.Timer
	batchTimerActive bool
	batchTimeout     time.Duration

	manager events.Manager // 事件管理器，后面可能会被移除

	incomingChan chan *batchMessage // 正在处理的消息队列
	idleChan     chan struct{}     

	reqStore *requestStore 

	deduplicator *deduplicator

	persistForward
}

func newObcBatch(id uint64, config *viper.Viper, stack consensus.Stack) *obcBatch {
	var err error

	op := &obcBatch{
		obcGeneric: obcGeneric{stack: stack},
	}

	op.persistForward.persistor = stack

	logger.Debugf("Replica %d obtaining startup information", id)

	op.manager = events.NewManagerImpl() // 创建事件管理器
	op.manager.SetReceiver(op)
	etf := events.NewTimerFactoryImpl(op.manager)
	op.pbft = newPbftCore(id, config, op, etf) // 创建pbft引擎
	op.manager.Start()
	op.externalEventReceiver.manager = op.manager
	op.broadcaster = newBroadcaster(id, op.pbft.N, op.pbft.f, op.pbft.broadcastTimeout, stack)

	op.batchSize = config.GetInt("general.batchsize")
	op.batchStore = nil
	......
	op.incomingChan = make(chan *batchMessage)

	op.batchTimer = etf.CreateTimer()

	op.reqStore = newRequestStore()

	op.deduplicator = newDeduplicator()

	op.idleChan = make(chan struct{})
	close(op.idleChan) // TODO remove eventually

	return op
}
```
从以上代码中我们可以看到，pbft引擎的创建最终调用了newPbftCore，后面我们详细的介绍。

##3 consensus.Consenter接口的实现
我们知道上节中介绍的New方法要求返回一个consenus.Consenter接口，而newObcBatch返回的是一个obcBatch对象，那说明obcBatch实现了consensus.Consenter接口。obcBatch定义了一个属性externalEventReceiver，该对象实现了consensus.Consenter接口的方法，我们来看看externalEventReceiver的相关源码。

```go
type externalEventReceiver struct {
	manager events.Manager
}

// 当接收到新消息时，用stack调用该方法对消息进行处理
func (eer *externalEventReceiver) RecvMsg(ocMsg *pb.Message, senderHandle *pb.PeerID) error {
	eer.manager.Queue() <- batchMessageEvent{
		msg:    ocMsg,
		sender: senderHandle,
	}
	return nil
}
```

externalEventReceiver结构体只包含了一个events.Manager对象，events.Manager是一个事件管理器，后面会详细介绍。RecvMsg方法主要将接收到新消息包装成batchMessageEvent对象通过chan的方法交给了事件管理器处理。externalEventReceiver将新接收到消息交给events.Manager进行处理，包括事务的执行、提交、回滚等事件都交给了events.Manager处理，详细的代码可以在源文件fabric/consensus/pbft/external.go中查看。

##4 事件管理器
事件管理器相关的实现在源文件fabric/consensus/util/events/events.go中

```go

type Receiver interface {
	ProcessEvent(e Event) Event
}

// 事件管理器接口
type Manager interface {
	Inject(Event)         
	Queue() chan<- Event 
	SetReceiver(Receiver) 
	Start()               
	Halt()                
}

// 事件管理器接口的实现
type managerImpl struct {
	threaded
	receiver Receiver
	events   chan Event
}
```

事件管理器的接口提供了一个Queue方法，该方法返回一个只可写的Event类型的chan，需要处理的事件只需要往这个chan队列上写入事件即可。

事件管理器中有个属性recevier，其类型为Reveiver接口，通过上面第二节的newObcBatch方法我们可以看到，manager的receiver属性被设置成了obcBatch对象，也就是obcBatch实现了Receiver接口的ProcessEvent方法。

我们看再看看事件管理器其它方法的实现：

```go
func (em *managerImpl) Start() {
	go em.eventLoop()
}

func (em *managerImpl) Queue() chan<- Event {
	return em.events
}
func SendEvent(receiver Receiver, event Event) {
	next := event
	for {
		next = receiver.ProcessEvent(next)
		if next == nil {
			break
		}
	}
}
func (em *managerImpl) Inject(event Event) {
	if em.receiver != nil {
		SendEvent(em.receiver, event)
	}
}
func (em *managerImpl) eventLoop() {
	for {
		select {
		case next := <-em.events:
			em.Inject(next)
		case <-em.exit:
			logger.Debug("eventLoop told to exit")
			return
		}
	}
}
```
Manager的Start方法主要是启动了一个go的routine去调用eventLoop方法，而eventLoop方法使用了一个无限循环去读取events的事件，当events中有事件需要处理时，便取出一个事件交由Inject方法进行处理，当recevier不为空，Inject方法又调用了SendEvent，而SendEvent方法调用了recevier的ProcessEvent方法对事件进行处理，也就是调用了obcBatch的ProcessEvent方法，这里需要注意了，SendEvent方法里有一个循环，循环结束的条件是ProcessEvent返回的值为空，如果不为空，则继续调用ProcessEvent进行处理，也就是说ProcessEvent处理后可能返回另外另外一个事件。

##5 obcBatch的ProcessEvent方法
obcBatch的ProcessEvent方法主要根据不同的事件类型，调用相应的处理方法，具体代码如下：

```go
func (op *obcBatch) ProcessEvent(event events.Event) events.Event {
	logger.Debugf("Replica %d batch main thread looping", op.pbft.id)
	switch et := event.(type) {
	case batchMessageEvent:
		ocMsg := et
		return op.processMessage(ocMsg.msg, ocMsg.sender)
	case executedEvent:
		op.stack.Commit(nil, et.tag.([]byte))
	case committedEvent:
		logger.Debugf("Replica %d received committedEvent", op.pbft.id)
		return execDoneEvent{}
	case execDoneEvent:
		if res := op.pbft.ProcessEvent(event); res != nil {
			return res
		}
		return op.resubmitOutstandingReqs()
	case batchTimerEvent:
		logger.Infof("Replica %d batch timer expired", op.pbft.id)
		if op.pbft.activeView && (len(op.batchStore) > 0) {
			return op.sendBatch()
		}
	case *Commit:
		res := op.pbft.ProcessEvent(event)
		op.startTimerIfOutstandingRequests()
		return res
	case viewChangedEvent:
		op.batchStore = nil
		op.pbft.outstandingReqBatches = make(map[string]*RequestBatch)

		logger.Debugf("Replica %d batch thread recognizing new view", op.pbft.id)
		if op.batchTimerActive {
			op.stopBatchTimer()
		}

		if op.pbft.skipInProgress {
			op.reqStore.outstandingRequests.empty()
		}

		op.reqStore.pendingRequests.empty()
		for i := op.pbft.h + 1; i <= op.pbft.h+op.pbft.L; i++ {
			if i <= op.pbft.lastExec {
				continue
			}

			cert, ok := op.pbft.certStore[msgID{v: op.pbft.view, n: i}]
			if !ok || cert.prePrepare == nil {
				continue
			}

			if cert.prePrepare.BatchDigest == "" {
				continue
			}

			if cert.prePrepare.RequestBatch == nil {
				logger.Warningf("Replica %d found a non-null prePrepare with no request batch, ignoring")
				continue
			}

			op.reqStore.storePendings(cert.prePrepare.RequestBatch.GetBatch())
		}

		return op.resubmitOutstandingReqs()
	case stateUpdatedEvent:
		op.reqStore = newRequestStore()
		return op.pbft.ProcessEvent(event)
	default:
		return op.pbft.ProcessEvent(event)
	}

	return nil
}
```

我们先看看事件类型为batchMessageEvent的情况，当事件类型为batchMessageEvent时，ProcessEvent方法最终调用了obcBatch的processMessage方法，而processMessage方法主要处理两类消息：pb.Message_CHAIN_TRANSACTION和pb.Message_CONSENSUS，代码如下：

```go
func (op *obcBatch) processMessage(ocMsg *pb.Message, senderHandle *pb.PeerID) events.Event {
	// 处理事务事件
	if ocMsg.Type == pb.Message_CHAIN_TRANSACTION {
		req := op.txToReq(ocMsg.Payload)
		return op.submitToLeader(req)
	}
	// 处理共识消息事件
	if ocMsg.Type != pb.Message_CONSENSUS {
		logger.Errorf("Unexpected message type: %s", ocMsg.Type)
		return nil
	}

	batchMsg := &BatchMessage{}
	err := proto.Unmarshal(ocMsg.Payload, batchMsg)
	if err != nil {
		logger.Errorf("Error unmarshaling message: %s", err)
		return nil
	}

	if req := batchMsg.GetRequest(); req != nil {
		if !op.deduplicator.IsNew(req) {
			logger.Warningf("Replica %d ignoring request as it is too old", op.pbft.id)
			return nil
		}

		op.logAddTxFromRequest(req)
		op.reqStore.storeOutstanding(req)
		if (op.pbft.primary(op.pbft.view) == op.pbft.id) && op.pbft.activeView {
			return op.leaderProcReq(req)
		}
		op.startTimerIfOutstandingRequests()
		return nil
	} else if pbftMsg := batchMsg.GetPbftMessage(); pbftMsg != nil {
		senderID, err := getValidatorID(senderHandle) // who sent this?
		if err != nil {
			panic("Cannot map sender's PeerID to a valid replica ID")
		}
		msg := &Message{}
		err = proto.Unmarshal(pbftMsg, msg)
		if err != nil {
			logger.Errorf("Error unpacking payload from message: %s", err)
			return nil
		}
		return pbftMessageEvent{
			msg:    msg,
			sender: senderID,
		}
	}

	logger.Errorf("Unknown request: %+v", batchMsg)

	return nil
}

// 将事务消息转换为Request对象
func (op *obcBatch) txToReq(tx []byte) *Request {
	now := time.Now()
	req := &Request{
		Timestamp: &google_protobuf.Timestamp{
			Seconds: now.Unix(),
			Nanos:   int32(now.UnixNano() % 1000000000),
		},
		Payload:   tx,
		ReplicaId: op.pbft.id,
	}
	// XXX sign req
	return req
}
```

当消息类型为pb.Message\_CHAIN\_TRANSACTION时，通过调用obcBatch的txToReq方法将消息转换为Request对象，然后调用submitToLeader方法将消息进行广播，通过跟踪源码发现，如果当前节点是leader，则将消息封装成RequestBatch对象并返回；当消息类型为pb.Message_CONSENSUS时，通过分析源码，最终processEvent方法可能会返回RequestBatch对象、pbftMessageEvent对象或者nil。因此，在ProcessEvent方法中，当事件类型为batchMessageEvent时，ProcessEvent可能返回RequestBatch、pbftMessageEvent事件或者nil，通过上节的分析我们知道，当ProcessEvent返回不为空时，会将返回值作为参数继续调用ProcessEvent进行处理。通过分析ProcessEvent方法我们知道，当事件类型为RequestBatch和pbftMessageEvent时，调用了obcBatch.pbft.ProcessEvent方法进行处理，pbft的ProcessEvent方法是pbft共识机制的入口，后面会进行详细的分析。

##6 pbft核心引擎
pbft引擎的核心对象是pbftCore，存储了pbft算法相关的参数和状态，具体的代码如下：

```go
type pbftCore struct {
	...
	// PBFT data
	activeView    bool              // 当前view是否处于活动状态
	byzantine     bool              // 当前节点是否为拜占庭节点
	f             int               // 最大能容忍失败的节点
	N             int               // 网络中最大的合法节点数
	h             uint64            
	id            uint64            // 节点id
	K             uint64            // 检查点周期
	logMultiplier uint64            
	L             uint64           
	lastExec      uint64            // 最后一次执行的命令id
	replicaCount  int               // 副本数
	seqNo         uint64            // 单调递增的序列号
	view          uint64            // 当前的view
	chkpts        map[uint64]string 
	pset          map[uint64]*ViewChange_PQ // 当前节点上一个view里到达pre-prepared状态的请求的一些信息
	qset          map[qidx]*ViewChange_PQ // 当前节点在上一个view中达到prepared状态的请求的一些信息
	...

	currentExec           *uint64                  // 正在执行的命令
	...
	
	certStore       map[msgID]*msgCert       // 存储每个请求每个阶段的投票情况 
	...
}

type msgCert struct {
	digest      string
	prePrepare  *PrePrepare
	sentPrepare bool
	prepare     []*Prepare
	sentCommit  bool
	commit      []*Commit
}
```

pbftCore对象的结构就按照pbft算法要求设计的，具体的算法不是本文的重点，需要研究pbft算法的，请看pbft的原始论文。

上节中我们分析到，pbft算法的入口pbftCore的ProcessEvent方法，我们来看看该方法具体做了些什么。

```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
	var err error
	logger.Debugf("Replica %d processing event", instance.id)
	switch et := e.(type) {
	case viewChangeTimerEvent:
		logger.Infof("Replica %d view change timer expired, sending view change: %s", instance.id, instance.newViewTimerReason)
		instance.timerActive = false
		instance.sendViewChange()
	case *pbftMessage:
		return pbftMessageEvent(*et)
	case pbftMessageEvent:
		msg := et
		logger.Debugf("Replica %d received incoming message from %v", instance.id, msg.sender)
		next, err := instance.recvMsg(msg.msg, msg.sender)
		if err != nil {
			break
		}
		return next
	case *RequestBatch:
		err = instance.recvRequestBatch(et)
	case *PrePrepare:
		err = instance.recvPrePrepare(et)
	case *Prepare:
		err = instance.recvPrepare(et)
	case *Commit:
		err = instance.recvCommit(et)
	case *Checkpoint:
		return instance.recvCheckpoint(et)
	case *ViewChange:
		return instance.recvViewChange(et)
	case *NewView:
		return instance.recvNewView(et)
	case *FetchRequestBatch:
		err = instance.recvFetchRequestBatch(et)
	case returnRequestBatchEvent:
		return instance.recvReturnRequestBatch(et)
	case stateUpdatedEvent:
		update := et.chkpt
		instance.stateTransferring = false
		// If state transfer did not complete successfully, or if it did not reach our low watermark, do it again
		if et.target == nil || update.seqNo < instance.h {
			if et.target == nil {
				logger.Warningf("Replica %d attempted state transfer target was not reachable (%v)", instance.id, et.chkpt)
			} else {
				logger.Warningf("Replica %d recovered to seqNo %d but our low watermark has moved to %d", instance.id, update.seqNo, instance.h)
			}
			if instance.highStateTarget == nil {
				logger.Debugf("Replica %d has no state targets, cannot resume state transfer yet", instance.id)
			} else if update.seqNo < instance.highStateTarget.seqNo {
				logger.Debugf("Replica %d has state target for %d, transferring", instance.id, instance.highStateTarget.seqNo)
				instance.retryStateTransfer(nil)
			} else {
				logger.Debugf("Replica %d has no state target above %d, highest is %d", instance.id, update.seqNo, instance.highStateTarget.seqNo)
			}
			return nil
		}
		logger.Infof("Replica %d application caught up via state transfer, lastExec now %d", instance.id, update.seqNo)
		// XXX create checkpoint
		instance.lastExec = update.seqNo
		instance.moveWatermarks(instance.lastExec) // The watermark movement handles moving this to a checkpoint boundary
		instance.skipInProgress = false
		instance.consumer.validateState()
		instance.executeOutstanding()
	case execDoneEvent:
		instance.execDoneSync()
		if instance.skipInProgress {
			instance.retryStateTransfer(nil)
		}
		// We will delay new view processing sometimes
		return instance.processNewView()
	case nullRequestEvent:
		instance.nullRequestHandler()
	case workEvent:
		et() // Used to allow the caller to steal use of the main thread, to be removed
	case viewChangeQuorumEvent:
		logger.Debugf("Replica %d received view change quorum, processing new view", instance.id)
		if instance.primary(instance.view) == instance.id {
			return instance.sendNewView()
		}
		return instance.processNewView()
	case viewChangedEvent:
		// No-op, processed by plugins if needed
	case viewChangeResendTimerEvent:
		if instance.activeView {
			logger.Warningf("Replica %d had its view change resend timer expire but it's in an active view, this is benign but may indicate a bug", instance.id)
			return nil
		}
		logger.Debugf("Replica %d view change resend timer expired before view change quorum was reached, resending", instance.id)
		instance.view-- // sending the view change increments this
		return instance.sendViewChange()
	default:
		logger.Warningf("Replica %d received an unknown message type %T", instance.id, et)
	}

	if err != nil {
		logger.Warning(err.Error())
	}

	return nil
}
```

这么多代码，是不是被吓着了，偷笑中...。其实ProcessEvent没有做太复杂的工作，仅仅是根据不同的事件类型调用不同的方法，包括处理pre-prepare、prepare、commit、viewChange等事件，熟悉pbft算法的同学直接看源码就行，源码比较清晰，这里就不多介绍了。

##7 总结
本文主要带大家粗略地过了一下fabric中使用pbft实现共识机制的过程，为大家提供一个分析共识机制源码的架子。当然还有未涉及的地方，如消息广播的实现等。有了这个架子大家便可以快速的对源码进行分析或者修改，希望对大家有帮助。












