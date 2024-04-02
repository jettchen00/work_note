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


## 四. 集群间的网络通信
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

# 1. 写请求的处理流程
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