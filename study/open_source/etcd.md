- [参考文档](#参考文档)
- [一. 配置相关](#一-配置相关)
- [二. 启动流程](#二-启动流程)
- [三. 选举流程](#三-选举流程)
- [四. 集群间的网络通信](#四-集群间的网络通信)
- [五. 数据存储](#五-数据存储)
- [六. 日志模块](#六-日志模块)
  - [日志复制流程](#日志复制流程)
  - [日志的管理](#日志的管理)
  - [日志的压缩](#日志的压缩)
- [七. 写请求的处理流程](#七-写请求的处理流程)
- [八. 读请求的处理流程](#八-读请求的处理流程)
  - [九. 租约的实现](#九-租约的实现)
- [十. watch机制](#十-watch机制)
  - [十一. 成员变更](#十一-成员变更)

# 参考文档
[彻底搞懂mvcc与watch](https://zhuanlan.zhihu.com/p/502786815)

[boltdb 源码导读-数据组织](https://zhuanlan.zhihu.com/p/332439403)

[boltdb 源码导读-索引设计](https://zhuanlan.zhihu.com/p/341416264)

[boltdb 源码导读-事务实现](https://zhuanlan.zhihu.com/p/363795675)

[自底向上分析 BoltDB 源码](https://www.bookstack.cn/books/jaydenwen123-boltdb_book)

[boltdb源码阅读](https://mp.weixin.qq.com/s/QfcHJ7dazjRUSC3vCMuofQ)

[Etcd Raft库的日志存储](https://www.codedump.info/post/20210628-etcd-wal/)

[Etcd Raft库的工程化实现](https://www.codedump.info/post/20210515-raft/)

[etcd中的Lease机制](https://www.cnblogs.com/ricklz/p/15232204.html)

[万字长文解析 etcd 如何实现 watch 机制](https://zhuanlan.zhihu.com/p/502786815)

[watch机制原理分析](https://www.lixueduan.com/post/etcd/05-watch/)

[watch 机制源码分析](https://www.lixueduan.com/post/etcd/13-watch-analyze-1/)

[Raft成员变更的工程实践](https://zhuanlan.zhihu.com/p/359206808)

[etcd 3.5版本的joint consensus实现解析](https://www.codedump.info/post/20220101-etcd3.5-joint-consensus/)

[etcd-raft的线性一致读](https://zhuanlan.zhihu.com/p/31050303)


# 一. 配置相关
1. 在启动etcd的时候，所有需要支持的配置项，首先需要在func newConfig() *config函数中通过以下函数初始化好每一个配置项的名称以及默认值，如下例子所示
```
fs.StringVar(&amp;cfg.ec.Dir, "data-dir", cfg.ec.Dir, "Path to the data directory.")
fs.UintVar(&amp;cfg.ec.ElectionMs, "election-timeout", cfg.ec.ElectionMs, "Time (in milliseconds) for an election to timeout.")
fs.BoolVar(&amp;cfg.ec.EnableV2, "enable-v2", cfg.ec.EnableV2, "Accept etcd V2 client requests.")
```
2. 然后通过func (f *FlagSet) Parse(arguments []string) error函数解析命令行参数，用命令行的配置值去覆盖1.1步骤设定的默认值。如果命令行输入了1.1步骤不曾初始化的配置项，则解析失败，退出启动。
3. 当然，所有配置项都可以写在配置文件中，然后通过func (cfg *config) configFromFile(path string)函数读取所有配置项。


# 二. 启动流程
1. func startEtcdOrProxyV2() ---> func StartEtcd(inCfg *Config) (e *Etcd, err error)
2. 在函数StartEtcd中，会创建一个Etcd对象，启动监听以处理集群其他节点以及客户端的请求，另外还会通过func NewServer(cfg ServerConfig) (srv *EtcdServer, err error)函数根据配置文件启动一个EtcdServer对象。在启动EtcdServer对象时，会根据是否存在wal日志来新启动一个raft.Node对象或者重启一个raft.Node对象。
3. 启动时会创建Etcd对象，而Etcd对象包含1个EtcdServer对象，EtcdServer对象包含1个raftNode对象，raftNode对象包含1个raft.Node对象，raft.Node对象包含1个RawNode对象，而RawNode对象包含1个raft对象。
4. EtcdServer对象初始化好之后，会依次启动多个go routine，如下所示。
```
func (s *EtcdServer) Start() {
    s.start()
    s.goAttach(func() { s.adjustTicks() })
    s.goAttach(func() { s.publish(s.Cfg.ReqTimeout()) })
    s.goAttach(s.purgeFile)
    s.goAttach(func() { monitorFileDescriptor(s.getLogger(), s.stopping) })
    s.goAttach(s.monitorVersions)
    s.goAttach(s.linearizableReadLoop)
    s.goAttach(s.monitorKVHash)
}

if err = e.servePeers(); err != nil {
	return e, err
}
if err = e.serveClients(); err != nil {
    return e, err
}
if err = e.serveMetrics(); err != nil {
    return e, err
}
```


# 三. 选举流程
1. etcd在启动过程中，raftNode对象会创建一个ticker定时器（默认是100ms的定时器）。然后在如下的调用栈中，会监听这个ticker定时器。每隔100ms触发调用一次 func (r *raftNode) tick() 函数，进而调用到 func (n *node) Tick()函数，这个函数会往node.tickc通道中发送一个空结构体的信号。
2. func (n *node) run()函数会监听node.tickc通道，收到信号后会调用func (rn *RawNode) Tick()函数。
3. follower角色的tick函数是func (r *raft) tickElection()，这个函数会判断是否已经过了随机超时时间，如果是则构造消息pb.Message{From: r.id, Type: pb.MsgHup}。后面的流程需要根据是否开启preVote做不同的处理，通过下面这个配置项来控制是否开启preVote。
4. 配置开启了preVote的情况：
    4.1： 通过func (r *raft) campaign(t CampaignType)函数竞争campaignPreElection，切换为StatePreCandidate状态，维持自身的term不变，向其他节点广播MsgPreVote消息，这个消息中的term=r.Term+1，会将消息append到r.msgs
    4.2： func (n *node) run()函数会将msgs打包到Ready结构，并将这个结构发送到n.readyc通道中
    4.3： func (r *raftNode) start(rh *raftReadyHandler)函数监听到n.readyc通道中的Ready消息，通过func (t *Transport) Send(msgs []raftpb.Message) ---> func (p *peer) send(m raftpb.Message)的调用栈将消息发送给其它peer节点。
    4.4： func (cr *streamReader) decodeLoop(rc io.ReadCloser, t streamType)，对端通过这个函数解析出收到的包，然后将包发送到cr.recvc通道中（如果是raftpb.MsgProp的类型消息，则发送到cr.propc通道）。cr.recvc和cr.propc通道，其实就是peer.recvc和peer.propc通道。
    4.5： func startPeer(t *Transport, urls types.URLs, peerID types.ID, fs *stats.FollowerStats)函数中，通过调用func (rc *raftNode) Process(ctx context.Context, m raftpb.Message)函数处理peer.recvc和peer.propc通道中的消息。
    4.6： 消息最终通过func (n *node) Step(ctx context.Context, m pb.Message)函数，进入到n.recvc或者n.propc通道中。
    4.7： func (n *node) run()函数会处理n.recvc或者n.propc通道中的消息，进入到func (r *raft) Step(m pb.Message)函数中。
    4.8： 如果该节点所认为的lead还在租约期内则会忽略该消息，否则如果满足投票条件，就回复同意的MsgPreVoteResp消息
    4.9： func (r *raft) Step(m pb.Message)函数中，如果存在一个对端的term大于本节点term并且其回复了一个拒绝的MsgPreVoteResp消息，则该节点重新切换为follower
    4.10： func stepCandidate(r *raft, m pb.Message) 函数会处理同意的MsgVoteResp消息，如果得到多数派同意则切换为StateCandidate状态，然后进入正式竞争选举流程
5. 配置没有开启preVote的情况：
    5.1： 通过func (r *raft) campaign(t CampaignType)函数竞争campaignElection，切换为StateCandidate状态，将自身的term加1，向其他节点广播MsgVote消息，会将消息append到r.msgs。
    5.2： func (n *node) run()函数会将msgs打包到Ready结构，并将这个结构发送到n.readyc通道中。
    5.3： func (rc *raftNode) serveChannels()函数监听到n.readyc通道中的Ready消息，通过func (t *Transport) Send(msgs []raftpb.Message) 函数将消息发送给其它peer节点。
    5.4： func (cr *streamReader) decodeLoop(rc io.ReadCloser, t streamType)，对端通过这个函数解析出收到的包，然后将包发送到cr.recvc通道中（如果是raftpb.MsgProp的类型消息，则发送到cr.propc通道）。cr.recvc和cr.propc通道，其实就是peer.recvc和peer.propc通道。
    5.5： func startPeer(t *Transport, urls types.URLs, peerID types.ID, fs *stats.FollowerStats)函数中，通过调用func (rc *raftNode) Process(ctx context.Context, m raftpb.Message)函数处理peer.recvc和peer.propc通道中的消息。
    5.6: 消息最终通过func (n *node) Step(ctx context.Context, m pb.Message)函数，进入到n.recvc或者n.propc通道中。
    5.7： func (n *node) run()函数会处理n.recvc或者n.propc通道中的消息，进入到func (r *raft) Step(m pb.Message)函数中。
    5.8: 如果满足投票条件，就回复同意的MsgVoteResp消息
    5.9： func stepCandidate(r *raft, m pb.Message)函数会处理投票回包。多数派同意，则通过func (r *raft) becomeLeader()函数切换到leader状态


# 四. 集群间的网络通信
1. 集群节点的配置：通过将如下所示的配置项存储到cfg.ec.InitialCluster中，然后通过urlsmap, token, err = cfg.PeerURLsMapAndToken("etcd")函数将成员节点的name和url存储到urlsmap，再通过cl, err = membership.NewClusterFromURLsMap(cfg.Logger, cfg.InitialClusterToken, cfg.InitialPeerURLsMap)函数生成RaftCluster的对象cl，这里cfg.InitialPeerURLsMap其实就是前面的urlsmap。
```
--initial-cluster 'infra1=http://127.0.0.1:12380,infra2=http://127.0.0.1:22380,infra3=http://127.0.0.1:32380'
```
2. 与其他节点的通信通道初始化：func NewServer(cfg ServerConfig) (srv *EtcdServer, err error)函数中会通过以下代码启动transport模块，并且开启与其他节点的通信（其中remote代表不参与选举的节点，peer代表参与选举的节点）。代码如下所示：
```
tr := &rafthttp.Transport{
        Logger:      cfg.Logger,
        TLSInfo:     cfg.PeerTLSInfo,
        DialTimeout: cfg.peerDialTimeout(),
        ID:          id,
        URLs:        cfg.PeerURLs,
        ClusterID:   cl.ID(),
        Raft:        srv,
        Snapshotter: ss,
        ServerStats: sstats,
        LeaderStats: lstats,
        ErrorC:      srv.errorc,
    }
    if err = tr.Start(); err != nil {
        return nil, err
    }
    // add all remotes into transport
    for _, m := range remotes {
        if m.ID != id {
            tr.AddRemote(m.ID, m.PeerURLs)
        }
    }
    for _, m := range cl.Members() {
        if m.ID != id {
            tr.AddPeer(m.ID, m.PeerURLs)
        }
    }
    srv.r.transport = tr
```
```
func startPeer(t *Transport, urls types.URLs, peerID types.ID, fs *stats.FollowerStats) *peer {
    if t.Logger != nil {
        t.Logger.Info("starting remote peer", zap.String("remote-peer-id", peerID.String()))
    }
    defer func() {
        if t.Logger != nil {
            t.Logger.Info("started remote peer", zap.String("remote-peer-id", peerID.String()))
        }
    }()

    status := newPeerStatus(t.Logger, t.ID, peerID)
    picker := newURLPicker(urls)
    errorc := t.ErrorC
    r := t.Raft
    pipeline := &amp;pipeline{
        peerID:        peerID,
        tr:            t,
        picker:        picker,
        status:        status,
        followerStats: fs,
        raft:          r,
        errorc:        errorc,
    }
    pipeline.start()

    p := &amp;peer{
        lg:             t.Logger,
        localID:        t.ID,
        id:             peerID,
        r:              r,
        status:         status,
        picker:         picker,
        msgAppV2Writer: startStreamWriter(t.Logger, t.ID, peerID, status, fs, r),
        writer:         startStreamWriter(t.Logger, t.ID, peerID, status, fs, r),
        pipeline:       pipeline,
        snapSender:     newSnapshotSender(t, picker, peerID, status),
        recvc:          make(chan raftpb.Message, recvBufSize),
        propc:          make(chan raftpb.Message, maxPendingProposals),
        stopc:          make(chan struct{}),
    }

    ctx, cancel := context.WithCancel(context.Background())
    p.cancel = cancel
    go func() {
        for {
            select {
            case mm := <-p.recvc:
                if err := r.Process(ctx, mm); err != nil {
                    if t.Logger != nil {
                        t.Logger.Warn("failed to process Raft message", zap.Error(err))
                    }
                }
            case <-p.stopc:
                return
            }
        }
    }()

    // r.Process might block for processing proposal when there is no leader.
    // Thus propc must be put into a separate routine with recvc to avoid blocking
    // processing other raft messages.
    go func() {
        for {
            select {
            case mm := <-p.propc:
                if err := r.Process(ctx, mm); err != nil {
                    if t.Logger != nil {
                        t.Logger.Warn("failed to process Raft message", zap.Error(err))
                    }
                }
            case <-p.stopc:
                return
            }
        }
    }()

    p.msgAppV2Reader = &amp;streamReader{
        lg:     t.Logger,
        peerID: peerID,
        typ:    streamTypeMsgAppV2,
        tr:     t,
        picker: picker,
        status: status,
        recvc:  p.recvc,
        propc:  p.propc,
        rl:     rate.NewLimiter(t.DialRetryFrequency, 1),
    }
    p.msgAppReader = &amp;streamReader{
        lg:     t.Logger,
        peerID: peerID,
        typ:    streamTypeMessage,
        tr:     t,
        picker: picker,
        status: status,
        recvc:  p.recvc,
        propc:  p.propc,
        rl:     rate.NewLimiter(t.DialRetryFrequency, 1),
    }

    p.msgAppV2Reader.start()
    p.msgAppReader.start()

    return p
}
```
3. 读取对端peer节点消息的流程，remote节点类似：peer对象首先会创建streamReader对象，然后启动一个协程。协程的任务：不停地通过net.http.RoundTrip（go语言的基础库）从对端请求数据，然后通过newMsgAppV2Decoder对象解析数据得到raft消息，如果是提议消息则将消息push到streamReader对象的propc管道，否则就push到recvc管道中。代码如下：
```
func (cr *streamReader) start() {
    cr.done = make(chan struct{})
    if cr.errorc == nil {
        cr.errorc = cr.tr.ErrorC
    }
    if cr.ctx == nil {
        cr.ctx, cr.cancel = context.WithCancel(context.Background())
    }
    go cr.run()
}

func (cr *streamReader) run() {
    t := cr.typ

    if cr.lg != nil {
        cr.lg.Info(
            "started stream reader with remote peer",
            zap.String("stream-reader-type", t.String()),
            zap.String("local-member-id", cr.tr.ID.String()),
            zap.String("remote-peer-id", cr.peerID.String()),
        )
    }

    for {
        rc, err := cr.dial(t)
        if err != nil {
            if err != errUnsupportedStreamType {
                cr.status.deactivate(failureType{source: t.String(), action: "dial"}, err.Error())
            }
        } else {
            cr.status.activate()
            if cr.lg != nil {
                cr.lg.Info(
                    "established TCP streaming connection with remote peer",
                    zap.String("stream-reader-type", cr.typ.String()),
                    zap.String("local-member-id", cr.tr.ID.String()),
                    zap.String("remote-peer-id", cr.peerID.String()),
                )
            }
            err = cr.decodeLoop(rc, t)
            if cr.lg != nil {
                cr.lg.Warn(
                    "lost TCP streaming connection with remote peer",
                    zap.String("stream-reader-type", cr.typ.String()),
                    zap.String("local-member-id", cr.tr.ID.String()),
                    zap.String("remote-peer-id", cr.peerID.String()),
                    zap.Error(err),
                )
            }
            switch {
            // all data is read out
            case err == io.EOF:
            // connection is closed by the remote
            case transport.IsClosedConnError(err):
            default:
                cr.status.deactivate(failureType{source: t.String(), action: "read"}, err.Error())
            }
        }
        // Wait for a while before new dial attempt
        err = cr.rl.Wait(cr.ctx)
        if cr.ctx.Err() != nil {
            if cr.lg != nil {
                cr.lg.Info(
                    "stopped stream reader with remote peer",
                    zap.String("stream-reader-type", t.String()),
                    zap.String("local-member-id", cr.tr.ID.String()),
                    zap.String("remote-peer-id", cr.peerID.String()),
                )
            }
            close(cr.done)
            return
        }
        if err != nil {
            if cr.lg != nil {
                cr.lg.Warn(
                    "rate limit on stream reader with remote peer",
                    zap.String("stream-reader-type", t.String()),
                    zap.String("local-member-id", cr.tr.ID.String()),
                    zap.String("remote-peer-id", cr.peerID.String()),
                    zap.Error(err),
                )
            }
        }
    }
}
```
4. peer对象还启动了两个协程，分别处理propc和recvc管道中的消息，然后调用func (rc *raftNode) Process(ctx context.Context, m raftpb.Message)函数处理。
5. 发送消息到对端peer节点的流程，remote节点类似：peer对象在创建的时候会初始化一个streamWriter对象。上层应用发送消息的话需要调用func (t *Transport) Send(msgs []raftpb.Message)接口，进而通过func (p *peer) send(m raftpb.Message)函数，将消息push到streamWriter对象的msgc管道中。代码如下：
```
func (t *Transport) Send(msgs []raftpb.Message) {
    for _, m := range msgs {
        if m.To == 0 {
            // ignore intentionally dropped message
            continue
        }
        to := types.ID(m.To)

        t.mu.RLock()
        p, pok := t.peers[to]
        g, rok := t.remotes[to]
        t.mu.RUnlock()

        if pok {
            if m.Type == raftpb.MsgApp {
                t.ServerStats.SendAppendReq(m.Size())
            }
            p.send(m)
            continue
        }

        if rok {
            g.send(m)
            continue
        }

        if t.Logger != nil {
            t.Logger.Debug(
                "ignored message send request; unknown remote peer target",
                zap.String("type", m.Type.String()),
                zap.String("unknown-target-peer-id", to.String()),
            )
        }
    }
}
```
6. peer对象初始化streamWriter对象后，streamWriter对象会启动一个协程，这个协程的任务是：不停地监听其他peer节点的连接，对于每个连接创建一个msgAppV2Encoder对象，然后从msgc管道中读取消息，通过func (enc *msgAppV2Encoder) encode(m *raftpb.Message)函数将消息序列化为二进制buffer，最后通过go语言的io.Writer基础库将消息发送到对端。代码如下：
```
func (cw *streamWriter) run() {
    var (
        msgc       chan raftpb.Message
        heartbeatc <-chan time.Time
        t          streamType
        enc        encoder
        flusher    http.Flusher
        batched    int
    )
    tickc := time.NewTicker(ConnReadTimeout / 3)
    defer tickc.Stop()
    unflushed := 0

    if cw.lg != nil {
        cw.lg.Info(
            "started stream writer with remote peer",
            zap.String("local-member-id", cw.localID.String()),
            zap.String("remote-peer-id", cw.peerID.String()),
        )
    }

    for {
        select {
        case <-heartbeatc:
            err := enc.encode(&amp;linkHeartbeatMessage)
            unflushed += linkHeartbeatMessage.Size()
            if err == nil {
                flusher.Flush()
                batched = 0
                sentBytes.WithLabelValues(cw.peerID.String()).Add(float64(unflushed))
                unflushed = 0
                continue
            }

            cw.status.deactivate(failureType{source: t.String(), action: "heartbeat"}, err.Error())

            sentFailures.WithLabelValues(cw.peerID.String()).Inc()
            cw.close()
            if cw.lg != nil {
                cw.lg.Warn(
                    "lost TCP streaming connection with remote peer",
                    zap.String("stream-writer-type", t.String()),
                    zap.String("local-member-id", cw.localID.String()),
                    zap.String("remote-peer-id", cw.peerID.String()),
                )
            }
            heartbeatc, msgc = nil, nil

        case m := <-msgc:
            err := enc.encode(&amp;m)
            if err == nil {
                unflushed += m.Size()

                if len(msgc) == 0 || batched > streamBufSize/2 {
                    flusher.Flush()
                    sentBytes.WithLabelValues(cw.peerID.String()).Add(float64(unflushed))
                    unflushed = 0
                    batched = 0
                } else {
                    batched++
                }

                continue
            }

            cw.status.deactivate(failureType{source: t.String(), action: "write"}, err.Error())
            cw.close()
            if cw.lg != nil {
                cw.lg.Warn(
                    "lost TCP streaming connection with remote peer",
                    zap.String("stream-writer-type", t.String()),
                    zap.String("local-member-id", cw.localID.String()),
                    zap.String("remote-peer-id", cw.peerID.String()),
                )
            }
            heartbeatc, msgc = nil, nil
            cw.r.ReportUnreachable(m.To)
            sentFailures.WithLabelValues(cw.peerID.String()).Inc()

        case conn := <-cw.connc:
            cw.mu.Lock()
            closed := cw.closeUnlocked()
            t = conn.t
            switch conn.t {
            case streamTypeMsgAppV2:
                enc = newMsgAppV2Encoder(conn.Writer, cw.fs)
            case streamTypeMessage:
                enc = &amp;messageEncoder{w: conn.Writer}
            default:
                if cw.lg != nil {
                    cw.lg.Panic("unhandled stream type", zap.String("stream-type", t.String()))
                }
            }
            if cw.lg != nil {
                cw.lg.Info(
                    "set message encoder",
                    zap.String("from", conn.localID.String()),
                    zap.String("to", conn.peerID.String()),
                    zap.String("stream-type", t.String()),
                )
            }
            flusher = conn.Flusher
            unflushed = 0
            cw.status.activate()
            cw.closer = conn.Closer
            cw.working = true
            cw.mu.Unlock()

            if closed {
                if cw.lg != nil {
                    cw.lg.Warn(
                        "closed TCP streaming connection with remote peer",
                        zap.String("stream-writer-type", t.String()),
                        zap.String("local-member-id", cw.localID.String()),
                        zap.String("remote-peer-id", cw.peerID.String()),
                    )
                }
            }
            if cw.lg != nil {
                cw.lg.Info(
                    "established TCP streaming connection with remote peer",
                    zap.String("stream-writer-type", t.String()),
                    zap.String("local-member-id", cw.localID.String()),
                    zap.String("remote-peer-id", cw.peerID.String()),
                )
            }
            heartbeatc, msgc = tickc.C, cw.msgc

        case <-cw.stopc:
            if cw.close() {
                if cw.lg != nil {
                    cw.lg.Warn(
                        "closed TCP streaming connection with remote peer",
                        zap.String("stream-writer-type", t.String()),
                        zap.String("remote-peer-id", cw.peerID.String()),
                    )
                }
            }
            if cw.lg != nil {
                cw.lg.Info(
                    "stopped TCP streaming connection with remote peer",
                    zap.String("stream-writer-type", t.String()),
                    zap.String("remote-peer-id", cw.peerID.String()),
                )
            }
            close(cw.done)
            return
        }
    }
}
```
7. 第6步骤中的连接，是在收到对方请求时，在func (h *streamHandler) ServeHTTP(w http.ResponseWriter, r *http.Request)中分配的，streamHandler对象是在Transport初始化时注册好的。可以看到，整个通信流程属于流式传输。代码如下：
```
func (t *Transport) Handler() http.Handler {
    pipelineHandler := newPipelineHandler(t, t.Raft, t.ClusterID)
    streamHandler := newStreamHandler(t, t, t.Raft, t.ID, t.ClusterID)
    snapHandler := newSnapshotHandler(t, t.Raft, t.Snapshotter, t.ClusterID)
    mux := http.NewServeMux()
    mux.Handle(RaftPrefix, pipelineHandler)
    mux.Handle(RaftStreamPrefix+"/", streamHandler)
    mux.Handle(RaftSnapshotPrefix, snapHandler)
    mux.Handle(ProbingPrefix, probing.NewHandler())
    return mux
}
```

# 五. 数据存储
1. etcd v2版本：只保存了key的最新的value，之前的value会被直接覆盖，如果需要知道一个key的历史记录，需要对该key维护一个历史变更的窗口，默认保存最新的1000个变更，但是当数据更新较快时，这1000个变更其实“不够用”，因为数据会被快速覆盖，之前的记录还是找不到。
2. etcd v3版本：etcd v3摒弃了v2不稳定的“滑动窗口”式设计，引入MVCC机制，采用从历史记录为主索引的存储结构，保存了key的所有历史记录变更，并支持数据在无锁状态下的的快速查询。由于etcd v3实现了MVCC，保存了每个key-value pair的历史版本，数据了大了很多，不能将整个数据库都存放到内存中。因此etcd v3摒弃了内存数据库，转为磁盘数据库，即整个数据都存储在磁盘上，底层的存储引擎使用的是BoltDB。
[Etcd V2与V3版本比较](https://zhuanlan.zhihu.com/p/369782579)
[Etcd V2与V3存储差异](https://zhuanlan.zhihu.com/p/271232304)
3. 应用状态机初始化的代码流程如下：
```
srv.kv = mvcc.New(srv.getLogger(), srv.be, srv.lessor, srv.consistIndex, mvcc.StoreConfig{CompactionBatchLimit: cfg.CompactionBatchLimit})

srv.applyV2 = NewApplierV2(cfg.Logger, srv.v2store, srv.cluster)

srv.applyV3Base = srv.newApplierV3Backend()
srv.applyV3Internal = srv.newApplierV3Internal()
if err = srv.restoreAlarms(); err != nil {
    return nil, err
}

func (s *EtcdServer) restoreAlarms() error {
    s.applyV3 = s.newApplierV3()
    as, err := v3alarm.NewAlarmStore(s.lg, s)
    if err != nil {
        return err
    }
    s.alarmStore = as
    if len(as.Get(pb.AlarmType_NOSPACE)) > 0 {
        s.applyV3 = newApplierV3Capped(s.applyV3)
    }
    if len(as.Get(pb.AlarmType_CORRUPT)) > 0 {
        s.applyV3 = newApplierV3Corrupt(s.applyV3)
    }
    return nil
}
```
4. 后端存储初始化的代码流程如下：
```
func newBackend(bcfg BackendConfig) *backend {
    if bcfg.Logger == nil {
        bcfg.Logger = zap.NewNop()
    }

    bopts := &bolt.Options{}
    if boltOpenOptions != nil {
        *bopts = *boltOpenOptions
    }
    bopts.InitialMmapSize = bcfg.mmapSize()
    bopts.FreelistType = bcfg.BackendFreelistType
    bopts.NoSync = bcfg.UnsafeNoFsync
    bopts.NoGrowSync = bcfg.UnsafeNoFsync

    db, err := bolt.Open(bcfg.Path, 0600, bopts)
    if err != nil {
        bcfg.Logger.Panic("failed to open database", zap.String("path", bcfg.Path), zap.Error(err))
    }

    // In future, may want to make buffering optional for low-concurrency systems
    // or dynamically swap between buffered/non-buffered depending on workload.
    b := &backend{
        db: db,

        batchInterval: bcfg.BatchInterval,
        batchLimit:    bcfg.BatchLimit,

        readTx: &readTx{
            buf: txReadBuffer{
                txBuffer: txBuffer{make(map[string]*bucketBuffer)},
            },
            buckets: make(map[string]*bolt.Bucket),
            txWg:    new(sync.WaitGroup),
        },

        stopc: make(chan struct{}),
        donec: make(chan struct{}),

        lg: bcfg.Logger,
    }
    b.batchTx = newBatchTxBuffered(b)
    go b.run()
    return b
}
```
5. 数据写入的代码流程如下：
```
func (a *applierV3backend) Apply(r *pb.InternalRaftRequest) *applyResult

func (aa *authApplierV3) Put(ctx context.Context, txn mvcc.TxnWrite, r *pb.PutRequest) (*pb.PutResponse, *traceutil.Trace, error)

func (a *quotaApplierV3) Put(ctx context.Context, txn mvcc.TxnWrite, p *pb.PutRequest) (*pb.PutResponse, *traceutil.Trace, error)

func (a *applierV3backend) Put(ctx context.Context, txn mvcc.TxnWrite, p *pb.PutRequest) (resp *pb.PutResponse, trace *traceutil.Trace, err error)

func (s *watchableStore) Write(trace *traceutil.Trace) TxnWrite {
    return &watchableStoreTxnWrite{s.store.Write(trace), s}
}

func (s *store) Write(trace *traceutil.Trace) TxnWrite {
    s.mu.RLock()
    tx := s.b.BatchTx()
    tx.Lock()
    tw := &storeTxnWrite{
        storeTxnRead: storeTxnRead{s, tx, 0, 0, trace},
        tx:           tx,
        beginRev:     s.currentRev,
        changes:      make([]mvccpb.KeyValue, 0, 4),
    }
    return newMetricsTxnWrite(tw)
}

func (tw *storeTxnWrite) Put(key, value []byte, lease lease.LeaseID)

func (t *batchTxBuffered) UnsafeSeqPut(bucketName []byte, key []byte, value []byte)

func (t *batchTx) UnsafeSeqPut(bucketName []byte, key []byte, value []byte)

// t.tx这个变量是通过func (b *backend) begin(write bool) *bolt.Tx函数得到的
func (t *batchTx) unsafePut(bucketName []byte, key []byte, value []byte, seq bool) {
    bucket := t.tx.Bucket(bucketName)
    if bucket == nil {
        t.backend.lg.Fatal(
            "failed to find a bucket",
            zap.String("bucket-name", string(bucketName)),
        )
    }
    if seq {
        // it is useful to increase fill percent when the workloads are mostly append-only.
        // this can delay the page split and reduce space usage.
        bucket.FillPercent = 0.9
    }
    if err := bucket.Put(key, value); err != nil {
        t.backend.lg.Fatal(
            "failed to write to a bucket",
            zap.String("bucket-name", string(bucketName)),
            zap.Error(err),
        )
    }
    t.pending++
}
```
6. 数据压缩
&emsp;&emsp;启动压缩goroutine，代码如下：
```
if num := cfg.AutoCompactionRetention; num != 0 {
        srv.compactor, err = v3compactor.New(cfg.Logger, cfg.AutoCompactionMode, num, srv.kv, srv)
        if err != nil {
            return nil, err
        }
        srv.compactor.Run()
    }
```
&emsp;&emsp;压缩的函数执行调用栈如下：
```
1. pc.c.Compact(pc.ctx, &pb.CompactionRequest{Revision: rev})

2. func (s *EtcdServer) Compact(ctx context.Context, r *pb.CompactionRequest) (*pb.CompactionResponse, error)

3. ar.resp, ar.physc, ar.trace, ar.err = a.s.applyV3.Compaction(r.Compaction)

4. func (a *applierV3backend) Compaction(compaction *pb.CompactionRequest) (*pb.CompactionResponse, <-chan struct{}, *traceutil.Trace, error)

5. ch, err := a.s.KV().Compact(trace, compaction.Revision)

6. func (s *store) Compact(trace *traceutil.Trace, rev int64) (<-chan struct{}, error)

7. func (s *store) compact(trace *traceutil.Trace, rev int64) (<-chan struct{}, error)

8. func (ti *treeIndex) Compact(rev int64) map[revision]struct{}

9. func (s *store) scheduleCompaction(compactMainRev int64, keep map[revision]struct{})
```
6. 关键的数据结构，如下所示
```
type store struct {
    ReadView
    WriteView

    cfg StoreConfig

    // mu read locks for txns and write locks for non-txn store changes.
    mu sync.RWMutex
	// 这个字段是通过func NewConsistentIndex(tx backend.BatchTx)函数创建的
    ci cindex.ConsistentIndexer
	// 这个字段是通过func newBackend(cfg ServerConfig)函数创建的
    b       backend.Backend
	// 这个字段是通过func newTreeIndex(lg *zap.Logger)函数创建的
    kvindex index
	// 这个字段是通过func NewLessor(lg *zap.Logger, b backend.Backend, cfg LessorConfig, ci cindex.ConsistentIndexer)函数创建的
    le lease.Lessor

    // revMuLock protects currentRev and compactMainRev.
    // Locked at end of write txn and released after write txn unlock lock.
    // Locked before locking read txn and released after locking.
    revMu sync.RWMutex
    // currentRev is the revision of the last completed transaction.
    currentRev int64
    // compactMainRev is the main revision of the last compaction.
    compactMainRev int64

    fifoSched schedule.Scheduler

    stopc chan struct{}

    lg *zap.Logger
}
```


# 六. 日志模块
## 日志复制流程
1. leader节点收到提议请求，首先通过func (r *raft) appendEntry(es ...pb.Entry)函数将日志保存到本地，如下所示：
```
func (r *raft) appendEntry(es ...pb.Entry) (accepted bool) {
    li := r.raftLog.lastIndex()
    for i := range es {
        es[i].Term = r.Term
        es[i].Index = li + 1 + uint64(i)
    }
    // Track the size of this uncommitted proposal.
    if !r.increaseUncommittedSize(es) {
        r.logger.Debugf(
            "%x appending new entries to log would exceed uncommitted entry size limit; dropping proposal",
            r.id,
        )
        // Drop the proposal.
        return false
    }
    // use latest "last" index after truncate/append
    li = r.raftLog.append(es...)
    r.prs.Progress[r.id].MaybeUpdate(li)
    // Regardless of maybeCommit's return, our caller will call bcastAppend.
    r.maybeCommit()
    return true
}
```
2. follower节点收到日志复制消息，会通过func (r *raft) handleAppendEntries(m pb.Message)函数将日志保存到本地，如下所示：
```
func (r *raft) handleAppendEntries(m pb.Message) {
    if m.Index < r.raftLog.committed {
        r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: r.raftLog.committed})
        return
    }

    if mlastIndex, ok := r.raftLog.maybeAppend(m.Index, m.LogTerm, m.Commit, m.Entries...); ok {
        r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: mlastIndex})
    } else {
        r.logger.Debugf("%x [logterm: %d, index: %d] rejected MsgApp [logterm: %d, index: %d] from %x",
            r.id, r.raftLog.zeroTermOnErrCompacted(r.raftLog.term(m.Index)), m.Index, m.LogTerm, m.Index, m.From)
        r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: m.Index, Reject: true, RejectHint: r.raftLog.lastIndex()})
    }
}
```
3. follower节点在向leader节点回复pb.MsgAppResp消息之前，会将日志落地到WAL模块，并且也会写入内存中的稳定日志结构，如下所示：
```
// gofail: var raftBeforeSave struct{}
if err := r.storage.Save(rd.HardState, rd.Entries); err != nil {
    .lg.Fatal("failed to save Raft hard state and entries", zap.Error(err))
}

// 写入内存中的稳定日志结构。
r.raftStorage.Append(rd.Entries)
```
4. leader节点收到pb.MsgAppResp消息后，首先更新follower节点的日志进度，然后判断是否达到多数派确认，如果多数派确认了并且日志term等于自身term，则更新可提交的日志索引committed，如下所示：
```
pr := r.prs.Progress[m.From]
pr.MaybeUpdate(m.Index)

// maybeCommit attempts to advance the commit index. Returns true if
// the commit index changed (in which case the caller should call
// r.bcastAppend).
func (r *raft) maybeCommit() bool {
    mci := r.prs.Committed()
    return r.raftLog.maybeCommit(mci, r.Term)
}

func (l *raftLog) maybeCommit(maxIndex, term uint64) bool {
    if maxIndex > l.committed && l.zeroTermOnErrCompacted(l.term(maxIndex)) == term {
        l.commitTo(maxIndex)
        return true
    }
    return false
}
```
## 日志的管理
emsp;&emsp;&日志管理对象主要分三种，其中两种在内存中管理，主要分为unstable和Storage对象，第三种是wal日志文件。unstable对象的作用是发送提交请求到follower节点前，管理请求日志，在发送日志复制请求之前，这些日志是不需要持久化的，所以放在内存中管理，也有利于做批量的日志复制。Storage对象的作用是在内存中维护一份已经持久化好的日志，因为wal日志文件的读取性能不高，只负责持久化日志并在重启时恢复。内存中的日志管理对象，初始化如下。当日志应用到状态机之后，会通过r.raftLog.stableTo(e.Index, e.Term)销毁unstable中的日志。
```
s = raft.NewMemoryStorage()   // 这个对象负责存储内存中的stable日志
c := &raft.Config{
    ID:              uint64(id),
    ElectionTick:    cfg.ElectionTicks,
    HeartbeatTick:   1,
    Storage:         s,
    MaxSizePerMsg:   maxSizePerMsg,
    MaxInflightMsgs: maxInflightMsgs,
    CheckQuorum:     true,
    PreVote:         cfg.PreVote,
}
raftlog := newLogWithSize(c.Storage, c.Logger, c.MaxCommittedSizePerReady)
r := &raft{
    id:                        c.ID,
    lead:                      None,
    isLearner:                 false,
    raftLog:                   raftlog,  // 这个对象负责存储内存中的unstable以及stable日志
    maxMsgSize:                c.MaxSizePerMsg,
    maxUncommittedSize:        c.MaxUncommittedEntriesSize,
    prs:                       tracker.MakeProgressTracker(c.MaxInflightMsgs),
    electionTimeout:           c.ElectionTick,
    heartbeatTimeout:          c.HeartbeatTick,
    logger:                    c.Logger,
    checkQuorum:               c.CheckQuorum,
    preVote:                   c.PreVote,
    readOnly:                  newReadOnly(c.ReadOnlyOption),
    disableProposalForwarding: c.DisableProposalForwarding,
}

type raftLog struct {
    // storage contains all stable entries since the last snapshot.
    storage Storage

    // unstable contains all unstable entries and snapshot.
    // they will be saved into storage.
    unstable unstable

    // committed is the highest log position that is known to be in
    // stable storage on a quorum of nodes.
    committed uint64
    // applied is the highest log position that the application has
    // been instructed to apply to its state machine.
    // Invariant: applied <= committed
    applied uint64

    logger Logger

    // maxNextEntsSize is the maximum number aggregate byte size of the messages
    // returned from calls to nextEnts.
    maxNextEntsSize uint64
}
```
&emsp;&emsp;持久化到文件的日志管理对象，初始化如下：
```
// 负责写snapshot文件的对象
ss := snap.New(cfg.Logger, cfg.SnapDir())

srv = &EtcdServer{
    readych:     make(chan struct{}),
    Cfg:         cfg,
    lgMu:        new(sync.RWMutex),
    lg:          cfg.Logger,
    errorc:      make(chan error, 1),
    v2store:     st,
    snapshotter: ss,
    r: *newRaftNode(
        raftNodeConfig{
            lg:          cfg.Logger,
            isIDRemoved: func(id uint64) bool { return cl.IsIDRemoved(types.ID(id)) },
            Node:        n,
            heartbeat:   heartbeat,
            raftStorage: s,   // 这个对象负责存储内存中的stable日志
            storage:     NewStorage(w, ss),  // 这个对象负责写wal日志和snapshot文件
        },
    ),
    id:               id,
    attributes:       membership.Attributes{Name: cfg.Name, ClientURLs: cfg.ClientURLs.StringSlice()},
    cluster:          cl,
    stats:            sstats,
    lstats:           lstats,
    SyncTicker:       time.NewTicker(500 * time.Millisecond),
    peerRt:           prt,
    reqIDGen:         idutil.NewGenerator(uint16(id), time.Now()),
    forceVersionC:    make(chan struct{}),
    AccessController: &AccessController{CORS: cfg.CORS, HostWhitelist: cfg.HostWhitelist},
    consistIndex:     cindex.NewConsistentIndex(be.BatchTx()),
}


func NewStorage(w *wal.WAL, s *snap.Snapshotter) Storage {
    return &storage{w, s}
}
```
## 日志的压缩
1. v2版本的数据压缩，代码如下：
```
1. func (s *EtcdServer) applyAll(ep *etcdProgress, apply *apply)

2. func (s *EtcdServer) triggerSnapshot(ep *etcdProgress)

3. func (s *EtcdServer) snapshot(snapi uint64, confState raftpb.ConfState)

4. func (ms *MemoryStorage) CreateSnapshot(i uint64, cs *pb.ConfState, data []byte) (pb.Snapshot, error)

5. func (s *EtcdServer) purgeFile()
```
2. v3版本的数据压缩：一般情况下，v3版本的数据不需要生成历史数据的快照。但如果follower节点进度比较落后，leader节点在复制日志的时候，leader将v2版本的snapshot数据 + 当前boltdb的数据 合并成一个MergedSnapshot发送给follower。follower节点收到后依次恢复v2和v3版本的数据，此时v2和v3版本数据的进度存在不一致（v3版本的数据比较新），随后v2版本的数据通过日志回放追赶上，而v3版本的数据通过boltdb中的consistentIndex确保日志回放的幂等性。代码如下：
```
// 日志复制的时候，如果日志已被压缩，则发送MsgSnap消息给follower
1. func (r *raft) maybeSendAppend(to uint64, sendIfEmpty bool)

// leader节点发消息给follower之前，对于MsgSnap类型的消息，先将该msg塞到r.msgSnapC通道中，然后通过ms[i].To = 0清掉目标follower。
2. func (r *raftNode) processMessages(ms []raftpb.Message)

// 发消息接口会忽略掉ms[i].To为0的消息
3. func (t *Transport) Send(msgs []raftpb.Message)

// 该函数通过select监听到r.msgSnapC有可读消息，
4. func (s *EtcdServer) applyAll(ep *etcdProgress, apply *apply)

// leader通过将当前v2版本的snapshot数据 + 当前boltdb的数据 合并成一个MergedSnapshot发送给follower。其中boltdb的数据是通过开启一个读事务，然后创建一个pipe管道，读事务每次最多从boltdb文件读取32K写入到pipe的write端，另一个协程则与读事务协程交替着读取、写入pipe，读取pipe内容后，通过http的流式传输发送给follower
5. func (s *EtcdServer) createMergedSnapshotMessage(m raftpb.Message, snapt, snapi uint64, confState raftpb.ConfState)

// 发送MergedSnapshot数据给follower，该消息的类型为MsgSnap
6. func (s *EtcdServer) sendMergedSnap(merged snap.Message)

// 发送/raft/snapshot路径的post请求
7. func (s *snapshotSender) send(merged snap.Message)

// follower节点对/raft/snapshot路径http请求的处理函数。这个函数会生成boltdb的快照文件，然后调用Process处理该条消息，该消息的类型为MsgSnap
8. func (h *snapshotHandler) ServeHTTP(w http.ResponseWriter, r *http.Request)

// 根据MsgSnap消息恢复follwer节点的unstable日志数据，此时unstable日志中会保存该v2版本的数据快照
9. func (r *raft) handleSnapshot(m pb.Message)

// 最后这个unstable日志中的快照数据会装载到ready结构中，然后调用func (s *EtcdServer) applyAll(ep *etcdProgress, apply *apply)函数
10. func (s *EtcdServer) applyAll(ep *etcdProgress, apply *apply)

// 通过snapshot恢复boltdb（这个恢复过程中用到了前面生成的boltdb的快照文件），然后也恢复下v2store存储
11. func (s *EtcdServer) applySnapshot(ep *etcdProgress, apply *apply)
```

# 七. 写请求的处理流程
1. func (s \*EtcdServer) Put(ctx context.Context, r \*pb.PutRequest) (*pb.PutResponse, error)
2. func (s *EtcdServer) processInternalRaftRequestOnce(ctx context.Context, r pb.InternalRaftRequest)
3. func (n *node) Propose(ctx context.Context, data []byte)函数将该消息发送到node.propc通道中，然后等待pm.result的结果。
4. func (n *node) run() 函数中，如果集群有leader节点则会处理node.propc通道中的消息，然后进入到func (r *raft) Step(m pb.Message)函数。对于propose消息，会满足m.Term == 0的条件。
5. 如果是leader，则进入处理函数：func stepLeader(r *raft, m pb.Message)。先通过func (r *raft) appendEntry(es ...pb.Entry)函数将日志存到本地，再通过func (r *raft) bcastAppend()函数将日志广播给follower，消息类型是pb.MsgApp。如果是成员变更的提交日志，如果上一次成员变更日志尚未应用到状态机，则拒绝再次进行成员变更。如果是follower，则进入处理函数：func stepFollower(r *raft, m pb.Message)。对于pb.MsgProp这种消息，如果开启转发功能则转发给leader节点，否则直接丢弃。
6. func (n *node) run()函数通过将msgs消息打包成ready结构，push到node.readyc通道中，然后func (r *raftNode) start(rh *raftReadyHandler)函数会处理这个ready，将消息发送到follower节点。
7. 经过etcd节点间的网络通信，日志复制消息会进入对端peer的recvc管道。经过peer启动的协程处理，最后func (s *EtcdServer) Process(ctx context.Context, m raftpb.Message)函数和func (n *node) Step(ctx context.Context, m pb.Message)函数会处理到该日志复制消息，最终消息进入node.recvc管道。
8. func (n *node) run() 函数中，会处理node.recvc通道中的消息，进入func stepFollower(r *raft, m pb.Message)函数中，然后调用func (r *raft) handleAppendEntries(m pb.Message)函数将日志保存到本地，然后回复pb.MsgAppResp消息。
9. func (n *node) run()函数会将pb.MsgAppResp消息发送到node.readyc通道中。
10. func (r *raftNode) start(rh *raftReadyHandler)函数会处理node.readyc通道中的消息，更新可提交日志的索引，将可提交的日志项raftNode.applyc通道中，然后通过func (ms *MemoryStorage) Append(entries []pb.Entry)函数将日志项保存起来，并且通过节点间网络通信将pb.MsgAppResp消息回复给leader节点。
11. leader节点通过func stepLeader(r *raft, m pb.Message)函数处理pb.MsgAppResp消息，如果多数派复制ok则说明日志可提交，然后再广播pb.MsgApp消息给所有follower（因为需要让follower立马知道有新的日志可提交）。如果某个节点复制日志失败，则调整该follower节点的复制进度next后重新发送pb.MsgApp消息。日志提交之后，会通过func (n *node) run()函数打包成ready数据，并push到node.readyc通道中。
12. func (r *raftNode) start(rh *raftReadyHandler)函数会读取ready数据，将待应用到状态机的日志发送到raftNode.applyc通道中。
13. func (s *EtcdServer) run()函数会读取r.applyc通道中的日志，通过func (s *EtcdServer) applyAll(ep *etcdProgress, apply *apply)函数应用到存储状态机。

# 八. 读请求的处理流程
```
1. func (s *EtcdServer) Range(ctx context.Context, r *pb.RangeRequest) (*pb.RangeResponse, error)

2. func (a *applierV3backend) Range(ctx context.Context, txn mvcc.TxnRead, r *pb.RangeRequest) (*pb.RangeResponse, error)

func (s *store) Read(trace *traceutil.Trace) TxnRead {
    s.mu.RLock()
    s.revMu.RLock()
    // backend holds b.readTx.RLock() only when creating the concurrentReadTx. After
    // ConcurrentReadTx is created, it will not block write transaction.
    tx := s.b.ConcurrentReadTx()
    tx.RLock() // RLock is no-op. concurrentReadTx does not need to be locked after it is created.
    firstRev, rev := s.compactMainRev, s.currentRev
    s.revMu.RUnlock()
    return newMetricsTxnRead(&amp;storeTxnRead{s, tx, firstRev, rev, trace})
}

// ConcurrentReadTx creates and returns a new ReadTx, which:
// A) creates and keeps a copy of backend.readTx.txReadBuffer,
// B) references the boltdb read Tx (and its bucket cache) of current batch interval.
func (b *backend) ConcurrentReadTx() ReadTx {
    b.readTx.RLock()
    defer b.readTx.RUnlock()
    // prevent boltdb read Tx from been rolled back until store read Tx is done. Needs to be called when holding readTx.RLock().
    b.readTx.txWg.Add(1)
    // TODO: might want to copy the read buffer lazily - create copy when A) end of a write transaction B) end of a batch interval.
    return &concurrentReadTx{
        buf:     b.readTx.buf.unsafeCopy(),
        tx:      b.readTx.tx,
        txMu:    &b.readTx.txMu,
        buckets: b.readTx.buckets,
        txWg:    b.readTx.txWg,
    }
}

3. func (tw *metricsTxnWrite) Range(key, end []byte, ro RangeOptions) (*RangeResult, error)

4. func (tr *storeTxnRead) Range(key, end []byte, ro RangeOptions) (r *RangeResult, err error)

// 首先通过keyIndex获取到相关的所有版本号，然后再根据每个版本号读取数据
5. func (tr *storeTxnRead) rangeKeys(key, end []byte, curRev int64, ro RangeOptions) (*RangeResult, error)

// 结合txReadBuffer和boltdb，读取数据
6. func (rt *concurrentReadTx) UnsafeRange(bucketName, key, endKey []byte, limit int64) ([][]byte, [][]byte)
```

## 九. 租约的实现
1. 租约相关的api接口，如下所示：
```
pb.RegisterLeaseServer(grpcServer, NewQuotaLeaseServer(s))

var _Lease_serviceDesc = grpc.ServiceDesc{
    ServiceName: "etcdserverpb.Lease",
    HandlerType: (*LeaseServer)(nil),
    Methods: []grpc.MethodDesc{
        {
            MethodName: "LeaseGrant",
            Handler:    _Lease_LeaseGrant_Handler,
        },
        {
            MethodName: "LeaseRevoke",
            Handler:    _Lease_LeaseRevoke_Handler,
        },
        {
            MethodName: "LeaseTimeToLive",
            Handler:    _Lease_LeaseTimeToLive_Handler,
        },
        {
            MethodName: "LeaseLeases",
            Handler:    _Lease_LeaseLeases_Handler,
        },
    },
    Streams: []grpc.StreamDesc{
        {
            StreamName:    "LeaseKeepAlive",
            Handler:       _Lease_LeaseKeepAlive_Handler,
            ServerStreams: true,
            ClientStreams: true,
        },
    },
    Metadata: "rpc.proto",
}

func (ls *LeaseServer) LeaseGrant(ctx context.Context, cr *pb.LeaseGrantRequest) (*pb.LeaseGrantResponse, error)

func (s *EtcdServer) LeaseGrant(ctx context.Context, r *pb.LeaseGrantRequest) (*pb.LeaseGrantResponse, error)

func (ls *LeaseServer) LeaseRevoke(ctx context.Context, rr *pb.LeaseRevokeRequest) (*pb.LeaseRevokeResponse, error)

func (s *EtcdServer) LeaseRevoke(ctx context.Context, r *pb.LeaseRevokeRequest) (*pb.LeaseRevokeResponse, error)

func (ls *LeaseServer) LeaseTimeToLive(ctx context.Context, rr *pb.LeaseTimeToLiveRequest) (*pb.LeaseTimeToLiveResponse, error)

func (s *EtcdServer) LeaseTimeToLive(ctx context.Context, r *pb.LeaseTimeToLiveRequest) (*pb.LeaseTimeToLiveResponse, error)

func (ls *LeaseServer) LeaseLeases(ctx context.Context, rr *pb.LeaseLeasesRequest) (*pb.LeaseLeasesResponse, error)

func (s *EtcdServer) LeaseLeases(ctx context.Context, r *pb.LeaseLeasesRequest) (*pb.LeaseLeasesResponse, error)

func (ls *LeaseServer) LeaseKeepAlive(stream pb.Lease_LeaseKeepAliveServer)
```
2. 租约管理实例的初始化
```
srv.lessor = lease.NewLessor(
        srv.getLogger(),
        srv.be,
        lease.LessorConfig{
            MinLeaseTTL:                int64(math.Ceil(minTTL.Seconds())),
            CheckpointInterval:         cfg.LeaseCheckpointInterval,
            ExpiredLeasesRetryInterval: srv.Cfg.ReqTimeout(),
        },
        srv.consistIndex,
    )
	
func NewStore(lg *zap.Logger, b backend.Backend, le lease.Lessor, ci cindex.ConsistentIndexer, cfg StoreConfig) *store {
    s := &store{
        cfg:     cfg,
        b:       b,
        ci:      ci,
        kvindex: newTreeIndex(lg),

        le: le,

        currentRev:     1,
        compactMainRev: -1,

        fifoSched: schedule.NewFIFOScheduler(),

        stopc: make(chan struct{}),

        lg: lg,
    }
	// 省略若干代码
    return s
}
```
3. 生成租约的代码流程如下：
```
// 生成一个请求：pb.InternalRaftRequest{LeaseGrant: r}
1. func (s *EtcdServer) LeaseGrant(ctx context.Context, r *pb.LeaseGrantRequest) (*pb.LeaseGrantResponse, error)

2. func (s *EtcdServer) raftRequestOnce(ctx context.Context, r pb.InternalRaftRequest) (proto.Message, error)

3. func (s *EtcdServer) processInternalRaftRequestOnce(ctx context.Context, r pb.InternalRaftRequest) (*applyResult, error)

func (n *node) Propose(ctx context.Context, data []byte) error {
    return n.stepWait(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Data: data}}})
}

// 经过raft算法达成一致后，通过func (s *EtcdServer) applyEntryNormal(e *raftpb.Entry)应用到状态机
4. func (a *applierV3backend) Apply(r *pb.InternalRaftRequest)
5. func (a *applierV3backend) LeaseGrant(lc *pb.LeaseGrantRequest) (*pb.LeaseGrantResponse, error)

6. func (le *lessor) Grant(id LeaseID, ttl int64) (*Lease, error)
```
4. 根据租约ID撤销租约的代码流程如下：
```
// 生成一个请求：pb.InternalRaftRequest{LeaseRevoke: r}
1. func (s *EtcdServer) LeaseRevoke(ctx context.Context, r *pb.LeaseRevokeRequest) (*pb.LeaseRevokeResponse, error)

2. func (s *EtcdServer) raftRequestOnce(ctx context.Context, r pb.InternalRaftRequest) (proto.Message, error)

3. func (s *EtcdServer) processInternalRaftRequestOnce(ctx context.Context, r pb.InternalRaftRequest) (*applyResult, error)

// 经过raft算法达成一致后，通过func (s *EtcdServer) applyEntryNormal(e *raftpb.Entry)应用到状态机
4. func (a *applierV3backend) Apply(r *pb.InternalRaftRequest)

5. func (a *applierV3backend) LeaseRevoke(lc *pb.LeaseRevokeRequest) (*pb.LeaseRevokeResponse, error)

6. func (le *lessor) Revoke(id LeaseID)
```
5. 根据租约ID查询租约的剩余生命周期（可以指定回包是否带上关联该租约ID的key列表），代码如下：
```
1. func (s *EtcdServer) LeaseTimeToLive(ctx context.Context, r *pb.LeaseTimeToLiveRequest) (*pb.LeaseTimeToLiveResponse, error)
// 如果leader节点收到请求则直接查询，如果是其它节点收到则转发给leader节点处理
2. func (le *lessor) Lookup(id LeaseID) *Lease
```
6. 查看目前存在的所有租约ID
```
1. func (s *EtcdServer) LeaseLeases(ctx context.Context, r *pb.LeaseLeasesRequest) (*pb.LeaseLeasesResponse, error)

2. func (le *lessor) Leases() []*Lease
```
7. 为数据key附上租约的代码流程
```
// raft日志应用到状态机，key-value写入存储层时boltdb底层存储版本号对应的value时，value包含租约ID，用于重启时恢复
func (tw *storeTxnWrite) put(key, value []byte, leaseID lease.LeaseID) {
	// 省略若干代码
	kv := mvccpb.KeyValue{
        Key:            key,
        Value:          value,
        CreateRevision: c,
        ModRevision:    rev,
        Version:        ver,
        Lease:          int64(leaseID),
    }
	
	if leaseID != lease.NoLease {
        if tw.s.le == nil {
            panic("no lessor to attach lease")
        }
        err = tw.s.le.Attach(leaseID, []lease.LeaseItem{{Key: string(key)}})
        if err != nil {
            panic("unexpected error from lease Attach")
        }
    }
}

func (le *lessor) Attach(id LeaseID, items []LeaseItem) error {
    le.mu.Lock()
    defer le.mu.Unlock()

    l := le.leaseMap[id]
    if l == nil {
        return ErrLeaseNotFound
    }

    l.mu.Lock()
    for _, it := range items {
        l.itemSet[it] = struct{}{}
        le.itemMap[it] = id
    }
    l.mu.Unlock()
    return nil
}
```
8. 租约的checkpoint实现：checkpoint的处理钩子函数在etcd初始化时就设置了，其实现的代码流程如下：
```
srv.lessor.SetCheckpointer(func(ctx context.Context, cp *pb.LeaseCheckpointRequest) {
            srv.raftRequestOnce(ctx, pb.InternalRaftRequest{LeaseCheckpoint: cp})
        })
```
```
srv.lessor = lease.NewLessor(
        srv.getLogger(),
        srv.be,
        lease.LessorConfig{
            MinLeaseTTL:                int64(math.Ceil(minTTL.Seconds())),
            CheckpointInterval:         cfg.LeaseCheckpointInterval,
            ExpiredLeasesRetryInterval: srv.Cfg.ReqTimeout(),
        },
        srv.consistIndex,
    )
	
func NewStore(lg *zap.Logger, b backend.Backend, le lease.Lessor, ci cindex.ConsistentIndexer, cfg StoreConfig) *store {
    s := &store{
        cfg:     cfg,
        b:       b,
        ci:      ci,
        kvindex: newTreeIndex(lg),

        le: le,

        currentRev:     1,
        compactMainRev: -1,

        fifoSched: schedule.NewFIFOScheduler(),

        stopc: make(chan struct{}),

        lg: lg,
    }
	// 省略若干代码
    return s
}
```
9. 租约过期检查的流程
```
// 每500ms运行一次逻辑
1. func (le *lessor) runLoop()
// 从最小堆中获取当前已过期的租约，只有leader节点才有这个逻辑
2. func (le *lessor) revokeExpiredLeases()
// 最多返回leaseRevokeRate / 2个租约，对于过期的租约会在堆中重新更新它的过期时间（过期时间 = 当前时间 + expiredLeaseRetryInterval，用于失败重试机制）
3. func (le *lessor) findExpiredLeases(limit int) []*Lease

4. 将获取到的过期租约列表写入le.expiredC通道

// 这个函数会读取le.expiredC通道，并且对于每个过期租约，生成一个raft请求：pb.LeaseRevokeRequest{ID: int64(lid)}
5. func (s *EtcdServer) run()

// 经过raft算法达成一致后，通过func (s *EtcdServer) applyEntryNormal(e *raftpb.Entry)应用到状态机
6. func (a *applierV3backend) Apply(r *pb.InternalRaftRequest) *applyResult

7. func (le *lessor) Revoke(id LeaseID)
```
10. 租约保活机制的流程
```
1. func _Lease_LeaseKeepAlive_Handler(srv interface{}, stream grpc.ServerStream)

2. func (ls *LeaseServer) LeaseKeepAlive(stream pb.Lease_LeaseKeepAliveServer)

// 不断地收到客户端的请求，server端则不断地更新租约的过期时间
3. func (ls *LeaseServer) leaseKeepAlive(stream pb.Lease_LeaseKeepAliveServer)

4. func (s *EtcdServer) LeaseRenew(ctx context.Context, id lease.LeaseID)

// leader节点直接处理，非leader节点将续期请求转发给leader节点
5. func (le *lessor) Renew(id LeaseID)
```


# 十. watch机制
1. watch存储实例的初始化
```
srv.kv = mvcc.New(srv.getLogger(), srv.be, srv.lessor, srv.consistIndex, mvcc.StoreConfig{CompactionBatchLimit: cfg.CompactionBatchLimit})

func newWatchableStore(lg *zap.Logger, b backend.Backend, le lease.Lessor, ci cindex.ConsistentIndexer, cfg StoreConfig) *watchableStore {
    if lg == nil {
        lg = zap.NewNop()
    }
    s := &watchableStore{
        store:    NewStore(lg, b, le, ci, cfg),  // 提供mvcc的存储机制
        victimc:  make(chan struct{}, 1),
        unsynced: newWatcherGroup(),
        synced:   newWatcherGroup(),
        stopc:    make(chan struct{}),
    }
    s.store.ReadView = &readView{s}
    s.store.WriteView = &writeView{s}
    if s.le != nil {
        // use this store as the deleter so revokes trigger watch events
        s.le.SetRangeDeleter(func() lease.TxnDelete { return s.Write(traceutil.TODO()) })
    }
    s.wg.Add(2)
    go s.syncWatchersLoop()
    go s.syncVictimsLoop()
    return s
}

func NewStore(lg *zap.Logger, b backend.Backend, le lease.Lessor, ci cindex.ConsistentIndexer, cfg StoreConfig) *store {
    if lg == nil {
        lg = zap.NewNop()
    }
    if cfg.CompactionBatchLimit == 0 {
        cfg.CompactionBatchLimit = defaultCompactBatchLimit
    }
    s := &store{
        cfg:     cfg,
        b:       b,   // 底层是boltdb实现
        ci:      ci,
        kvindex: newTreeIndex(lg),  // B树维护key的版本号列表信息

        le: le,

        currentRev:     1,
        compactMainRev: -1,

        fifoSched: schedule.NewFIFOScheduler(),

        stopc: make(chan struct{}),

        lg: lg,
    }
    s.ReadView = &readView{s}
    s.WriteView = &writeView{s}
    if s.le != nil {
        s.le.SetRangeDeleter(func() lease.TxnDelete { return s.Write(traceutil.TODO()) })
    }

    tx := s.b.BatchTx()
    tx.Lock()
    tx.UnsafeCreateBucket(keyBucketName)
    tx.UnsafeCreateBucket(metaBucketName)
    tx.Unlock()
    s.b.ForceCommit()

    s.mu.Lock()
    defer s.mu.Unlock()
	// 根据boltdb数据恢复kvindex的B树索引
    if err := s.restore(); err != nil {
        // TODO: return the error instead of panic here?
        panic("failed to recover store from backend")
    }

    return s
}
```
2. watch请求的处理流程
&emsp;&emsp;注册处理watch请求的watch handle函数
```
func StartEtcd(inCfg *Config) (e *Etcd, err error)
func (e *Etcd) serveClients() (err error)
func (sctx *serveCtx) serve(
    s *etcdserver.EtcdServer,
    tlsinfo *transport.TLSInfo,
    handler http.Handler,
    errHandler func(error),
    gopts ...grpc.ServerOption) (err error)
	
func Server(s *etcdserver.EtcdServer, tls *tls.Config, gopts ...grpc.ServerOption) *grpc.Server
pb.RegisterWatchServer(grpcServer, NewWatchServer(s))

var _Watch_serviceDesc = grpc.ServiceDesc{
    ServiceName: "etcdserverpb.Watch",
    HandlerType: (*WatchServer)(nil),
    Methods:     []grpc.MethodDesc{},
    Streams: []grpc.StreamDesc{
        {
            StreamName:    "Watch",
            Handler:       _Watch_Watch_Handler,
            ServerStreams: true,
            ClientStreams: true,
        },
    },
    Metadata: "rpc.proto",
}
func RegisterWatchServer(s *grpc.Server, srv WatchServer) {
    s.RegisterService(&_Watch_serviceDesc, srv)
}

func _Watch_Watch_Handler(srv interface{}, stream grpc.ServerStream) error {
    return srv.(WatchServer).Watch(&watchWatchServer{stream})
}
```
&emsp;&emsp;watch handle函数的处理流程
```
func (ws *watchServer) Watch(stream pb.Watch_WatchServer)

func (sws *serverWatchStream) recvLoop()

func (ws *watchStream) Watch(id WatchID, key, end []byte, startRev int64, fcs ...FilterFunc)

func (s *watchableStore) watch(key, end []byte, startRev int64, id WatchID, ch chan<- WatchResponse, fcs ...FilterFunc) (*watcher, cancelFunc)
```
&emsp;&emsp;watch变更事件的生成、通知流程
```
// 先构造一个TxnWrite
func (s *store) Write(trace *traceutil.Trace) TxnWrite

// 写请求结束后，会将本次所有的数据变更打包成Event事件列表
func (tw *watchableStoreTxnWrite) End()

// 根据Event事件筛选出相关的watcher，然后依次通知给各个watcher
func (s *watchableStore) notify(rev int64, evs []mvccpb.Event)

// 将Event事件写入到watcher的通道中，这个通道其实就是watchStream的通道，并且这个通道可以由多个watcher共享（1个客户端可以创建多个watcher）
func (w *watcher) send(wr WatchResponse)

// 将每个watcher的Event事件依次通过gRPCStream发给客户端
func (sws *serverWatchStream) sendLoop()

func (x *watchWatchServer) Send(m *WatchResponse)
```

## 十一. 成员变更
1. 成员变更的api接口：首先是注册http api接口，其次是ClusterServer结构实现具体接口
```
var _Cluster_serviceDesc = grpc.ServiceDesc{
    ServiceName: "etcdserverpb.Cluster",
    HandlerType: (*ClusterServer)(nil),
    Methods: []grpc.MethodDesc{
        {
            MethodName: "MemberAdd",
            Handler:    _Cluster_MemberAdd_Handler,
        },
        {
            MethodName: "MemberRemove",
            Handler:    _Cluster_MemberRemove_Handler,
        },
        {
            MethodName: "MemberUpdate",
            Handler:    _Cluster_MemberUpdate_Handler,
        },
        {
            MethodName: "MemberList",
            Handler:    _Cluster_MemberList_Handler,
        },
        {
            MethodName: "MemberPromote",
            Handler:    _Cluster_MemberPromote_Handler,
        },
    },
    Streams:  []grpc.StreamDesc{},
    Metadata: "rpc.proto",
}
```
```
pb.RegisterClusterServer(grpcServer, NewClusterServer(s))

// 往集群添加1个节点
func (cs *ClusterServer) MemberAdd(ctx context.Context, r *pb.MemberAddRequest) (*pb.MemberAddResponse, error)

// 集群移除1个节点
func (cs *ClusterServer) MemberRemove(ctx context.Context, r *pb.MemberRemoveRequest) (*pb.MemberRemoveResponse, error)

// 更新集群中的某个节点
func (cs *ClusterServer) MemberUpdate(ctx context.Context, r *pb.MemberUpdateRequest) (*pb.MemberUpdateResponse, error)
```
```
// EtcdServer层实现具体逻辑
func (s *EtcdServer) AddMember(ctx context.Context, memb membership.Member) ([]*membership.Member, error)

func (s *EtcdServer) RemoveMember(ctx context.Context, id uint64) ([]*membership.Member, error)
```
2. 成员变更的代码流程，如下：
```
// leader节点收到成员变更的提交请求，将其当成普通日志一样，先append到unstable日志，广播给follower后再刷到wal日志文件中。另外，如果发现上一次的成员变更尚未完成则会忽略本次成员变更
1. func stepLeader(r *raft, m pb.Message)

// follower节点收到日志复制请求，也将其当成普通日志一样，先append到unstable日志
2. func stepFollower(r *raft, m pb.Message)

// follower节点处理ready结构时，如果发现可提交的日志中存在成员变更日志，则需要同步等待这些可提交日志全部应用到状态机中才能将msg发送出去
3. func (r *raftNode) start(rh *raftReadyHandler)

// leader节点判断日志已复制到多数派，则更新commit_index。如果是联合共识状态，此时这个函数会取2个多数派中较小的那个commit_index
4. func (r *raft) maybeCommit()

// 将可提交的日志打包到ready结构中
5. func newReady(r *raft, prevSoftSt *SoftState, prevHardSt pb.HardState)

// 将可提交的日志应用到状态机，接着调用func (s *EtcdServer) apply函数，从这个函数可以看到etcd只支持raftpb.EntryConfChange类型的单成员变更
6. func (s *EtcdServer) applyAll(ep *etcdProgress, apply *apply)

// 对于成员变更，则调用这个函数。这个函数校验成员变更请求通过，将变更请求push到node.confc通道中。func (n *node) run()函数会消费这个成员变更请求更新集群的成员配置。applyConfChange函数一定会等到func (n *node) run()函数处理完毕后才会继续apply后续的日志
7. func (s *EtcdServer) applyConfChange(cc raftpb.ConfChange, confState *raftpb.ConfState)
```
&emsp;&emsp;将成员变更日志apply时，如果是需要进入联合共识状态，则会创建出Cold和Cnew两个多数派；如果是需要离开联合共识状态，则会清掉Cold那个多数派，最终只留下Cnew这一个多数派。代码如下：
```
func (r *raft) applyConfChange(cc pb.ConfChangeV2) pb.ConfState {
    cfg, prs, err := func() (tracker.Config, tracker.ProgressMap, error) {
        changer := confchange.Changer{
            Tracker:   r.prs,
            LastIndex: r.raftLog.lastIndex(),
        }
        if cc.LeaveJoint() {
            return changer.LeaveJoint()
        } else if autoLeave, ok := cc.EnterJoint(); ok {
            return changer.EnterJoint(autoLeave, cc.Changes...)
        }
        return changer.Simple(cc.Changes...)
    }()

    if err != nil {
        // TODO(tbg): return the error to the caller.
        panic(err)
    }

    return r.switchToConfig(cfg, prs)
}
```
&emsp;&emsp;leader节点复制日志消息的时候，根据当前Cold和Cnew的并集去广播消息。
&emsp;&emsp;如果是自动离开join consensus状态，则leader在将join consensus的日志应用到状态机后，会自动创建一个空的日志变更请求。代码如下：
```
if r.prs.Config.AutoLeave && oldApplied < r.pendingConfIndex && newApplied >= r.pendingConfIndex && r.state == StateLeader {
    // If the current (and most recent, at least for this leader's term)
    // configuration should be auto-left, initiate that now. We use a
    // nil Data which unmarshals into an empty ConfChangeV2 and has the
    // benefit that appendEntry can never refuse it based on its size
    // (which registers as zero).
    ent := pb.Entry{
        Type: pb.EntryConfChangeV2,
        Data: nil,
    }
    // There's no way in which this proposal should be able to be rejected.
    if !r.appendEntry(ent) {
        panic("refused un-refusable auto-leaving ConfChangeV2")
    }
    r.pendingConfIndex = r.raftLog.lastIndex()
    r.logger.Infof("initiating automatic transition out of joint configuration %s", r.prs.Config)
}
```
3. 不在Cnew中的节点，如何退出服务
&emsp;&emsp;func (s *EtcdServer) apply(es []raftpb.Entry,confState *raftpb.ConfState,) (appliedt uint64, appliedi uint64, shouldStop bool) 函数中，调用applyConfChange的返回值会告诉自身服务是否需要stop，如果需要stop则etcdserver会在1秒后退出，代码如下：
```
removedSelf, err := s.applyConfChange(cc, confState)

var shouldstop bool
    if ep.appliedt, ep.appliedi, shouldstop = s.apply(ents, &ep.confState); shouldstop {
        go s.stopWithDelay(10*100*time.Millisecond, fmt.Errorf("the member has been permanently removed from the cluster"))
    }
	
func (s *EtcdServer) stopWithDelay(d time.Duration, err error) {
    select {
    case <-time.After(d):
    case <-s.done:
    }
    select {
    case s.errorc <- err:
    default:
    }
}
```
&emsp;&emsp;func (s *EtcdServer) run()函数，读取到s.errorc通道中的错误，就会退出。
