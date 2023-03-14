Etcd 将 raft 协议实现为一个 library，然后本身作为一个应用使用它。当然，可能是为了推广它所实现的这个 library，etcd 还额外提供了一个叫[raftexample](https://github.com/etcd-io/etcd/tree/v3.3.10/contrib/raftexample)的示例程序，向用户展示怎样在它所提供的 raft library 的基础上构建出一个分布式的 KV 存储引擎。

在 etcd 中，raft 作为底层的共识模块，运行在一个`goroutine`里，通过`channel`接受上层（etcdserver）传来的消息，并将处理后的结果通过另一个`channel`返回给上层应用，他们的交互过程大概是这样的：

![Raft Stack](https://blog.betacat.io/image/raft-in-etcd/raft-stack.png)

这种全异步的交互方式好处就是它提高了性能，但坏处就是难以调试，代码看起来会很绕。拿 etcd 举例，很多时候你只看到它把一个消息 push 到一个 slice/channel 里面，然后这部分函数调用链就结束了，你无法直观的追踪到，到底是谁最后处理了这个消息。

我们来看一下这个 raft library 里面都有哪些文件：

```bash
$ tree --dirsfirst -L 1 -I '*test*' -P '*.go'
.
├── raftpb
├── doc.go
├── log.go
├── log_unstable.go
├── logger.go
├── node.go
├── progress.go
├── raft.go
├── rawnode.go
├── read_only.go
├── status.go
├── storage.go
└── util.go
```

下面按功能模块依次介绍：

## raftpb

Raft 中的序列化是借助于[Protocol Buffer](https://developers.google.com/protocol-buffers/)来实现的，这个文件夹就定义了需要序列化的几个数据结构，我们先从`Entry`和`Message`开始看起：

### Entry

从整体上来说，一个集群中的每个节点都是一个状态机，而 raft 管理的就是对这个状态机进行更改的一些操作，这些操作在代码中被封装为一个个`Entry`。

```go
// https://github.com/etcd-io/etcd/blob/v3.3.10/raft/raftpb/raft.pb.go#L203
type Entry struct {
    Term             uint64
    Index            uint64
    Type             EntryType
    Data             []byte
}
```

- Term：选举任期，每次选举之后递增 1。它的主要作用是标记信息的时效性，比方说当一个节点发出来的消息中携带的 term 是 2，而另一个节点携带的 term 是 3，那我们就认为第一个节点的信息过时了。
- Index：当前这个 entry 在整个 raft 日志中的位置索引。有了`Term`和`Index`之后，一个 log entry 就能被唯一标识。
- Type：当前 entry 的类型，目前 etcd 支持两种类型：[EntryNormal](https://github.com/etcd-io/etcd/blob/v3.3.10/raft/raftpb/raft.pb.go#L47)和[EntryConfChange](https://github.com/etcd-io/etcd/blob/v3.3.10/raft/raftpb/raft.pb.go#L48)，EntryNormal 代表当前 Entry 是对状态机的操作，EntryConfChange 则代表对当前集群配置进行更改的操作，比如增加或者减少节点。
- Data：一个被序列化后的 byte 数组，代表当前 entry 真正要执行的操作，比方说如果上面的`Type`是`EntryNormal`，那这里的 Data 就可能是具体要更改的 key-value pair，如果`Type`是`EntryConfChange`，那 Data 就是具体的配置更改项[ConfChange](https://github.com/etcd-io/etcd/blob/v3.3.10/raft/raftpb/raft.pb.go#L283)。raft 算法本身并不关心这个数据是什么，它只是把这段数据当做 log 同步过程中的 payload 来处理，具体对这个数据的解析则有上层应用来完成。

### Message

Raft 集群中节点之间的通讯都是通过传递不同的`Message`来完成的，这个`Message`结构就是一个非常 general 的大容器，它涵盖了各种消息所需的字段。

```go
// https://github.com/etcd-io/etcd/blob/v3.3.10/raft/raftpb/raft.pb.go#L239
type Message struct {
    Type             MessageType
    To               uint64
    From             uint64
    Term             uint64
    LogTerm          uint64
    Index            uint64
    Entries          []Entry
    Commit           uint64
    Snapshot         Snapshot
    Reject           bool
    RejectHint       uint64
    Context          []byte
}
```

- Type：当前传递的消息类型，它的取值有[很多个](https://github.com/etcd-io/etcd/blob/v3.3.10/raft/raftpb/raft.pb.go#L80)，但大致可以分成两类：
  1. Raft 协议相关的，包括心跳 MsgHeartbeat、日志 MsgApp、投票消息 MsgVote 等。
  2. 上层应用触发的（没错，上层应用并不是通过 api 与 raft 库交互的，而是通过发消息），比如应用对数据更改的消息 MsgProp(osal)。

不同类型的消息会用到下面不同的字段：

- To, From 分别代表了这个消息的接受者和发送者。
- Term：这个消息发出时整个集群所处的任期。
- LogTerm：消息发出者所保存的日志中最后一条的任期号，一般`MsgVote`会用到这个字段。
- Index：日志索引号。如果当前消息是`MsgVote`的话，代表这个 candidate 最后一条日志的索引号，它跟上面的`LogTerm`一起代表这个 candidate 所拥有的最新日志信息，这样别人就可以比较自己的日志是不是比 candidata 的日志要新，从而决定是否投票。
- Entries：需要存储的日志。
- Commit：已经提交的日志的索引值，用来向别人同步日志的提交信息。
- Snapshot：一般跟`MsgSnap`合用，用来放置具体的 Snapshot 值。
- Reject，RejectHint：代表对方节点拒绝了当前节点的请求(MsgVote/MsgApp/MsgSnap…)

## log_unstable.go

顾名思义，unstable 数据结构用于还没有被用户层持久化的数据，它维护了两部分内容`snapshot`和`entries`：

```go
// https://github.com/etcd-io/etcd/blob/v3.3.10/raft/log_unstable.go#L23
type unstable struct {
    // the incoming unstable snapshot, if any.
    snapshot *pb.Snapshot
    // all entries that have not yet been written to storage.
    entries []pb.Entry
    offset  uint64

    logger Logger
}
```

`entries`代表的是要进行操作的日志，但日志不可能无限增长，在特定的情况下，某些过期的日志会被清空。那这就引入一个新问题了，如果此后一个新的`follower`加入，而`leader`只有一部分操作日志，那这个新`follower`不是没法跟别人同步了吗？所以这个时候`snapshot`就登场了 - 我无法给你之前的日志，但我给你所有之前日志应用后的结果，之后的日志你再以这个`snapshot`为基础进行应用，那我们的状态就可以同步了。因此它们的结构关系可以用下图表示：

![](https://blog.betacat.io/image/raft-in-etcd/log_unstable.png)

这里的前半部分是快照数据，而后半部分是日志条目组成的数组 entries，另外 unstable.offset 成员保存的是 entries 数组中的第一条数据在 raft 日志中的索引，即第 i 条 entries 在 raft 日志中的索引为`i + unstable.offset`。

## storage.go

这个文件定义了一个[Storage](https://github.com/etcd-io/etcd/blob/v3.3.10/raft/storage.go#L46)接口，因为 etcd 中的 raft 实现并不负责数据的持久化，所以它希望上面的应用层能实现这个接口，以便提供给它查询 log 的能力。

另外，这个文件也提供了`Storage`接口的一个内存版本的实现[MemoryStorage](https://github.com/etcd-io/etcd/blob/v3.3.10/raft/storage.go#L74)，这个实现同样也维护了`snapshot`和`entries`这两部分，他们的排列跟`unstable`中的类似，也是`snapshot`在前，`entries`在后。从代码中看来`etcdserver`和`raftexample`都是直接用的这个实现来提供 log 的查询功能的。

## log.go

有了以上的介绍 unstable、Storage 的准备之后，下面可以来介绍 raftLog 的实现，这个结构体承担了 raft 日志相关的操作。

raftLog 由以下成员组成：

- storage Storage：前面提到的存放已经持久化数据的 Storage 接口。
- unstable unstable：前面分析过的 unstable 结构体，用于保存应用层还没有持久化的数据。
- committed uint64：保存当前提交的日志数据索引。
- applied uint64：保存当前传入状态机的数据最高索引。

需要说明的是，一条日志数据，首先需要被提交（committed）成功，然后才能被应用（applied）到状态机中。因此，以下不等式一直成立：`applied <= committed`。

raftLog 结构体中，几部分数据的排列如下图所示

![RaftLog Layout](https://blog.betacat.io/image/raft-in-etcd/raftlog-layout.png)

这个数据排布的情况，可以从 raftLog 的初始化函数中看出来：

```go
// https://github.com/etcd-io/etcd/blob/v3.3.10/raft/log.go#L45
// newLog returns log using the given storage. It recovers the log to the state
// that it just commits and applies the latest snapshot.
func newLog(storage Storage, logger Logger) *raftLog {
    if storage == nil {
        log.Panic("storage must not be nil")
    }
    log := &raftLog{
        storage: storage,
        logger:  logger,
    }
    firstIndex, err := storage.FirstIndex()
    if err != nil {
        panic(err) // TODO(bdarnell)
    }
    lastIndex, err := storage.LastIndex()
    if err != nil {
        panic(err) // TODO(bdarnell)
    }
    log.unstable.offset = lastIndex + 1
    log.unstable.logger = logger
    // Initialize our committed and applied pointers to the time of the last compaction.
    log.committed = firstIndex - 1
    log.applied = firstIndex - 1

    return log
}
```

因此，从这里的代码可以看出，raftLog 的两部分，持久化存储和非持久化存储，它们之间的分界线就是 lastIndex，在此之前都是`Storage`管理的已经持久化的数据，而在此之后都是`unstable`管理的还没有持久化的数据。

以上分析中还有一个疑问，为什么并没有初始化 unstable.snapshot 成员，也就是 unstable 结构体的快照数据？原因在于，上面这个是初始化函数，也就是节点刚启动的时候调用来初始化存储状态的函数，而 unstable.snapshot 数据，是在启动之后同步数据的过程中，如果需要同步快照数据时才会去进行赋值修改的数据，因此在这里并没有对它进行操作的地方。

## progress.go

Leader 通过`Progress`这个数据结构来追踪一个 follower 的状态，并根据`Progress`里的信息来决定每次同步的日志项。这里介绍三个比较重要的属性：

```go
// https://github.com/etcd-io/etcd/blob/v3.3.10/raft/progress.go#L37
// Progress represents a follower’s progress in the view of the leader. Leader maintains
// progresses of all followers, and sends entries to the follower based on its progress.
type Progress struct {
    Match, Next uint64

    State ProgressStateType

    ins *inflights
}
```

1. 用来保存当前 follower 节点的日志状态的属性：

    - Match：保存目前为止，已复制给该 follower 的日志的最高索引值。如果 leader 对该 follower 上的日志情况一无所知的话，这个值被设为 0。
    - Next：保存下一次 leader 发送 append 消息给该 follower 的日志索引，即下一次复制日志时，leader 会从`Next`开始发送日志。

    在正常情况下，`Next = Match + 1`，也就是下一个要同步的日志应当是对方已有日志的下一条。

2. `State`属性用来保存该节点当前的同步状态，它会有一下几种取值[3](https://blog.betacat.io/post/raft-implementation-in-etcd/#fn:3)：

    - ProgressStateProbe

    探测状态，当 follower 拒绝了最近的 append 消息时，那么就会进入探测状态，此时 leader 会试图继续往前追溯该 follower 的日志从哪里开始丢失的。在 probe 状态时，leader 每次最多 append 一条日志，如果收到的回应中带有`RejectHint`信息，则回退`Next`索引，以便下次重试。在初始时，leader 会把所有 follower 的状态设为 probe，因为它并不知道各个 follower 的同步状态，所以需要慢慢试探。

    - ProgressStateReplicate

    当 leader 确认某个 follower 的同步状态后，它就会把这个 follower 的 state 切换到这个状态，并且用`pipeline`的方式快速复制日志。leader 在发送复制消息之后，就修改该节点的`Next`索引为发送消息的最大索引+1。

    - ProgressStateSnapshot

    接收快照状态。当 leader 向某个 follower 发送 append 消息，试图让该 follower 状态跟上 leader 时，发现此时 leader 上保存的索引数据已经对不上了，比如 leader 在 index 为 10 之前的数据都已经写入快照中了，但是该 follower 需要的是 10 之前的数据，此时就会切换到该状态下，发送快照给该 follower。当快照数据同步追上之后，并不是直接切换到 Replicate 状态，而是首先切换到 Probe 状态。

3. `ins`属性用来做流量控制，因为如果同步请求非常多，再碰上网络分区时，leader 可能会累积很多待发送消息，一旦网络恢复，可能会有非常大流量发送给 follower，所以这里要做 flow control。它的实现有点类似 TCP 的[滑动窗口](https://en.wikipedia.org/wiki/Sliding_window_protocol)，这里不再赘述。

综上，`Progress`其实也是个状态机，下面是它的状态转移图：

![Progress State Machine](https://blog.betacat.io/image/raft-in-etcd/progress-state-machine.png)

## raft.go

前面铺设了一大堆概念，现在终于轮到实现逻辑了。从名字也可以看出，raft 协议的具体实现就在这个文件里。这其中，大部分的逻辑是由`Step`函数驱动的。

```go
// https://github.com/etcd-io/etcd/blob/v3.3.10/raft/raft.go#L752
func (r *raft) Step(m pb.Message) error {
    //...
    switch m.Type {
        case pb.MsgHup:
        //...
        case pb.MsgVote, pb.MsgPreVote:
        //...
        default:
            r.step(r, m)
    }
}
```

`Step`的主要作用是处理不同的[消息](https://blog.betacat.io/post/raft-implementation-in-etcd/#message)，所以以后当我们想知道 raft 对某种消息的处理逻辑时，到这里找就对了。在函数的最后，有个`default`语句，即所有上面不能处理的消息都落入这里，由一个小写的`step`函数处理，这个设计的原因是什么呢？

其实是因为这里的 raft 也被实现为一个状态机，它的`step`属性是一个函数指针，根据当前节点的不同角色，指向不同的消息处理函数：[stepLeader](https://github.com/etcd-io/etcd/blob/v3.3.10/raft/raft.go#L875)/[stepFollower](https://github.com/etcd-io/etcd/blob/v3.3.10/raft/raft.go#L1121)/[stepCandidate](https://github.com/etcd-io/etcd/blob/v3.3.10/raft/raft.go#L1079)。与它类似的还有一个`tick`函数指针，根据角色的不同，也会在[tickHeartbeat](https://github.com/etcd-io/etcd/blob/v3.3.10/raft/raft.go#L608)和[tickElection](https://github.com/etcd-io/etcd/blob/v3.3.10/raft/raft.go#L598)之间来回切换，分别用来触发定时心跳和选举检测。这里的函数指针感觉像实现了`OOP`里的多态。

![Raft State Machine](https://blog.betacat.io/image/raft-in-etcd/raft-function-state-machine.png)

## node.go

`node`的主要作用是应用层（etcdserver）和共识模块（raft）的衔接。将应用层的消息传递给底层共识模块，并将底层共识模块共识后的结果反馈给应用层。所以它的[初始化函数](https://github.com/etcd-io/etcd/blob/v3.3.10/raft/node.go#L243)创建了很多用来通信的`channel`，然后就在另一个`goroutine`里面开始了事件循环，不停的在各种`channel`中倒腾数据（貌似这种由`for-select-channel`组成的事件循环在 Go 里面很受欢迎）。

```go
// https://github.com/etcd-io/etcd/blob/v3.3.10/raft/node.go#L286
for {
  select {
    case m := <-propc:
        r.Step(m)
    case m := <-n.recvc:
        r.Step(m)
    case cc := <-n.confc:
        // Add/remove/update node according to cc.Type
    case <-n.tickc:
        r.tick()
    case readyc <- rd:
        // Cleaning after result is consumed by application
    case <-advancec:
        // Stablize logs
    case c := <-n.status:
        // Update status
    case <-n.stop:
        close(n.done)
        return
    }
}
```

`propc`和`recvc`中拿到的是从上层应用传进来的消息，这个消息会被交给 raft 层的`Step`函数处理，具体处理逻辑我上面有过介绍。

下面来解释下`readyc`的作用。在 etcd 的这个实现中，`node`并不负责数据的持久化、网络消息的通信、以及将已经提交的 log 应用到状态机中，所以`node`使用`readyc`这个`channel`对外通知有数据要处理了，并将这些需要外部处理的数据打包到一个`Ready`结构体中：

```go
// https://github.com/etcd-io/etcd/blob/v3.3.10/raft/node.go#L52
// Ready encapsulates the entries and messages that are ready to read,
// be saved to stable storage, committed or sent to other peers.
// All fields in Ready are read-only.
type Ready struct {
    // The current volatile state of a Node.
    // SoftState will be nil if there is no update.
    // It is not required to consume or store SoftState.
    *SoftState

    // The current state of a Node to be saved to stable storage BEFORE
    // Messages are sent.
    // HardState will be equal to empty state if there is no update.
    pb.HardState

    // ReadStates can be used for node to serve linearizable read requests locally
    // when its applied index is greater than the index in ReadState.
    // Note that the readState will be returned when raft receives msgReadIndex.
    // The returned is only valid for the request that requested to read.
    ReadStates []ReadState

    // Entries specifies entries to be saved to stable storage BEFORE
    // Messages are sent.
    Entries []pb.Entry

    // Snapshot specifies the snapshot to be saved to stable storage.
    Snapshot pb.Snapshot

    // CommittedEntries specifies entries to be committed to a
    // store/state-machine. These have previously been committed to stable
    // store.
    CommittedEntries []pb.Entry

    // Messages specifies outbound messages to be sent AFTER Entries are
    // committed to stable storage.
    // If it contains a MsgSnap message, the application MUST report back to raft
    // when the snapshot has been received or has failed by calling ReportSnapshot.
    Messages []pb.Message

    // MustSync indicates whether the HardState and Entries must be synchronously
    // written to disk or if an asynchronous write is permissible.
    MustSync bool
}
```

应用程序得到这个`Ready`之后，需要：

1. 将 HardState, Entries, Snapshot 持久化到 storage。
2. 将 Messages 广播给其他节点。
3. 将 CommittedEntries（已经 commit 还没有 apply）应用到状态机。
4. 如果发现 CommittedEntries 中有成员变更类型的 entry，调用`node.ApplyConfChange()`方法让`node`知道。
5. 最后再调用`node.Advance()`告诉 raft，这批状态更新处理完了，状态已经演进了，可以给我下一批 Ready 让我处理。

## Life of a Request

前面我们把整个包的结构过了一遍，下面来结合具体的代码看看 raft 对一个请求的处理过程是怎样的。我一直觉得，如果能从代码的层面追踪到一个请求的处理过程，那无论是从宏观还是微观的角度，对理解整个系统都是非常有帮助的。

### Life of a Vote Request

1. 首先，在`node`的大循环里，有一个会定时输出的`tick channel`，它来触发`raft.tick()`函数，根据上面的介绍可知，如果当前节点是 follower，那它的`tick`函数会指向`tickElection`。`tickElection`的处理逻辑是给自己发送一个`MsgHup`的内部消息，`Step`函数看到这个消息后会调用`campaign`函数，进入竞选状态。

    ```go
    // tickElection is run by followers and candidates after r.electionTimeout.
    func (r *raft) tickElection() {
        r.electionElapsed++

        if r.promotable() && r.pastElectionTimeout() {
            r.electionElapsed = 0
            r.Step(pb.Message{From: r.id, Type: pb.MsgHup})
        }
    }

    func (r *raft) Step(m pb.Message) error {
        //...
        switch m.Type {
        case pb.MsgHup:
            r.campaign(campaignElection)
        }
    }
    ```

2. `campaign`则会调用`becomeCandidate`把自己切换到 candidate 模式，并递增`Term`值。然后再将自己的`Term`及日志信息发送给其他的节点，请求投票。

    ```go
    func (r *raft) campaign(t CampaignType) {
        //...
        r.becomeCandidate()
        // Get peer id from progress
        for id := range r.prs {
            //...
            r.send(pb.Message{Term: term, To: id, Type: voteMsg, Index: r.raftLog.lastIndex(), LogTerm: r.raftLog.lastTerm(), Context: ctx})
        }
    }
    ```

3. 另一方面，其他节点在接受到这个请求后，会首先比较接收到的`Term`是不是比自己的大，以及接受到的日志信息是不是比自己的要新，从而决定是否投票。这个逻辑我们还是可以从`Step`函数中找到：

    ```go
    func (r *raft) Step(m pb.Message) error {
        //...
        switch m.Type {
        case pb.MsgVote, pb.MsgPreVote:
            // We can vote if this is a repeat of a vote we've already cast...
            canVote := r.Vote == m.From ||
                // ...we haven't voted and we don't think there's a leader yet in this term...
                (r.Vote == None && r.lead == None) ||
                // ...or this is a PreVote for a future term...
                (m.Type == pb.MsgPreVote && m.Term > r.Term)
            // ...and we believe the candidate is up to date.
            if canVote && r.raftLog.isUpToDate(m.Index, m.LogTerm) {
                r.send(pb.Message{To: m.From, Term: m.Term, Type: voteRespMsgType(m.Type)})
            } else {
                r.send(pb.Message{To: m.From, Term: r.Term, Type: voteRespMsgType(m.Type), Reject: true})
            }
        }
    }
    ```

4. 最后当 candidate 节点收到投票回复后，就会计算收到的选票数目是否大于所有节点数的一半，如果大于则自己成为 leader，并昭告天下，否则将自己置为 follower：

    ```go
    func (r *raft) Step(m pb.Message) error {
        //...
        switch m.Type {
        case myVoteRespType:
            gr := r.poll(m.From, m.Type, !m.Reject)
            switch r.quorum() {
            case gr:
                if r.state == StatePreCandidate {
                    r.campaign(campaignElection)
                } else {
                    r.becomeLeader()
                    r.bcastAppend()
                }
            case len(r.votes) - gr:
                r.becomeFollower(r.Term, None)
        }
    }
    ```

### Life of a Write Request

1. 一个写请求一般会通过调用`node.Propose`开始，`Propose`方法将这个写请求封装到一个`MsgProp`消息里面，发送给自己处理。
2. 消息处理函数`Step`无法直接处理这个消息，它会调用那个小写的`step`函数，来根据当前的状态进行处理。

    - 如果当前是 follower，那它会把这个消息转发给 leader。

    ```go
    func stepFollower(r *raft, m pb.Message) error {
        switch m.Type {
        case pb.MsgProp:
            //...
            m.To = r.lead
            r.send(m)
        }
    }
    ```

3. Leader 收到这个消息后（不管是 follower 转发过来的还是自己内部产生的）会有两步操作：

    1. 将这个消息添加到自己的 log 里
    2. 向其他 follower 广播这个消息

    ```go
    func stepLeader(r *raft, m pb.Message) error {
        switch m.Type {
        case pb.MsgProp:
            //...
            if !r.appendEntry(m.Entries...) {
                return ErrProposalDropped
            }
            r.bcastAppend()
            return nil
        }
    }
    ```

4. 在 follower 接受完这个 log 后，会返回一个`MsgAppResp`消息。
5. 当 leader 确认已经有足够多的 follower 接受了这个 log 后，它首先会 commit 这个 log，然后再广播一次，告诉别人它的 commit 状态。这里的实现就有点像两阶段提交了。

    ```go
    func stepLeader(r *raft, m pb.Message) error {
        switch m.Type {
        case pb.MsgAppResp:
            //...
            if r.maybeCommit() {
                r.bcastAppend()
            }
        }
    }

    // maybeCommit attempts to advance the commit index. Returns true if
    // the commit index changed (in which case the caller should call
    // r.bcastAppend).
    func (r *raft) maybeCommit() bool {
        //...
        mis := r.matchBuf[:len(r.prs)]
        idx := 0
        for _, p := range r.prs {
            mis[idx] = p.Match
            idx++
        }
        sort.Sort(mis)
        mci := mis[len(mis)-r.quorum()]
        return r.raftLog.maybeCommit(mci, r.Term)
    }
    ```

## Conclusion

Etcd 里的 raft 模块只实现了 raft 共识算法，而像消息的网络传输，数据存储都由上层应用来完成。这篇文章先介绍了基本的数据结构，然后在这些数据结构的基础上引入了 raft 算法。同时，这里还以一个投票请求和写请求为例，介绍了一个请求从接受到应答的完整处理过程。

但到目前为止，我们还有很多细节没有涉及，比如说 Linearizable Read，snapshot 机制，WAL 的存储与回放，所以希望你能以这篇文章为基础，顺藤摸瓜，继续深入研究下去。
