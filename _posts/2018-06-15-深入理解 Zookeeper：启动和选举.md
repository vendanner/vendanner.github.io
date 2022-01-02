---
layout:     post
title:      深入理解 Zookeeper：启动和选举
subtitle:   
date:       2018-06-15
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Zookeeper
    - bigdata
---

Zookeeper 能实现分布式一致性在于：**ZAB 协议**是为分布式协调服务 Zookeeper 专门设计的一种支持`崩溃恢复`的`原子广播`协议。Zookeeper 使用一个单一的主进程(Leader) 处理客户端的所有事务请求，并采用 ZAB 的原子广播协议，将服务器数据的状态变更以事务 Proposal 的形式广播到所有副本进程(Follower)。

**原子广播**保证数据的有序性和容错性，**崩溃恢复**可以解决 Leader 节点单点故障问题。这两个概念在 Zookeeper 非常重要，本节介绍**崩溃恢复**，下节介绍**原子广播**。

**崩溃恢复**在集群启动和 Leader 故障都会发生，就是平常所说的选举流程。本节结合集群的启动流程详细介绍关于选举的过程。

Zookeeper 集群启动成功：

- 从磁盘恢复数据
- 选举出 Leader 节点
- 半数以上的 Follower 节点从 Leader 节点同步数据

### 恢复数据

Zookeeper 每个节点都在内存中保存**全量数据**，当然数据在内存是不安全的，它也会定时做**快照**并写日志。刚启动时需要从最新的快照中加载数据，如果还有一些事务没在快照中需要从日志 replay。假设集群 ZXID = 100，而快照中最大的 ZXID=97，那么需要从日志中把ZXID= 98-100 的事务重新执行一遍。

总所周知 Zookeeper 是类似文件系统这样的树形结构，它在内存中以 DataTree 结构存在，具体涉及下面几个数据结构

```java
数据模型：DataTree (DataNode)
持久化(FileTxnSnapLog)： 快照(SnapShot) + 日志(TxnLog)
zk数据库(ZKDatabase)：DataTree + FileTxnSnapLog
```

上面是一些理论基础，下面从源码的角度看看如何恢复数据。

```java
// org.apache.zookeeper.server.quorum.QuorumPeerMain#initializeAndRun
protected void initializeAndRun(String[] args) throws ConfigException, IOException {
    QuorumPeerConfig config = new QuorumPeerConfig();
    // 解析配置文件： args[0] = "zoo.cfg的路径"
    if(args.length == 1) {
        config.parse(args[0]);
    }
    // 清理快照的线程：旧的快照无用需要被清理，每隔一段时间清理并保留最近几个
    DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config.getDataDir(),     
            config.getDataLogDir(),       
            config.getSnapRetainCount(),       
            config.getPurgeInterval());        
    purgeMgr.start();
   
    if(args.length == 1 && config.servers.size() > 0) {
        // 集群模式启动
        runFromConfig(config);
    } else {
        LOG.warn("Either no config or no quorum defined in config, running in standalone mode");
        // there is only server in the quorum -- run as standalone
        ZooKeeperServerMain.main(args);
    }
}
```

#### 配置解析

读取 `zoo.cfg` 内容并解析参数

```java
// org.apache.zookeeper.server.quorum.QuorumPeerConfig#parse
-> // org.apache.zookeeper.server.quorum.QuorumPeerConfig#parseProperties
public void parseProperties(Properties zkProp) throws IOException, ConfigException {
  // 配置文件解析大同小异，这里挑几个重要的参数讲解
  if(key.equals("electionAlg")) {      
    // 选举算法的指定：默认值是3： FastLeaderElection
    electionAlg = Integer.parseInt(value);
  } else if(key.equals("peerType")) {
    // 默认 participant，可参与选举，后续可成为 Leader/Follower
    // observer 没有选举权，此类节点只为增加 Zookeeper 集群的查询性能
    if(value.toLowerCase().equals("observer")) {
      peerType = LearnerType.OBSERVER;
    } else if(value.toLowerCase().equals("participant")) {
      peerType = LearnerType.PARTICIPANT;
    } else {
      throw new ConfigException("Unrecognised peertype: " + value);
    }
  } else if(key.equals("autopurge.snapRetainCount")) {
    // 快照清理参数 autopurge.purgeInterval = 3
    snapRetainCount = Integer.parseInt(value);
  } else if(key.equals("autopurge.purgeInterval")) {
    // 快照清理参数，默认为 1 ，单位小时
    purgeInterval = Integer.parseInt(value);
  } 
  
  // 初始化选举何时结束的实例
  if (serverGroup.size() > 0) {
    if(servers.size() != serverGroup.size())
      throw new ConfigException("Every server must be in exactly one group");
    // 默认权重为 1
    for(QuorumServer s : servers.values()) {
      if(!serverWeight.containsKey(s.id))
        serverWeight.put(s.id, (long) 1);
    }
    // quorumVerifier = QuorumHierarchical
    quorumVerifier = new QuorumHierarchical(numGroups, serverWeight, serverGroup);
  } else {
    // 默认 QuorumMaj
    LOG.info("Defaulting to majority quorums");
    quorumVerifier = new QuorumMaj(servers.size());
  }
  ...
}
```

选举算法默认是 `FastLeaderElection`，选举结束如何定义呢？

- 集群节点进行了分区：QuorumHierarchical
- 集群节点没有进行分区：QuorumMaj (少数服从多数)

#### 清理快照

会定时做快照，我们只需要最近几个，之前的快照需要被清理(默认1小时清理只保存最近3个)。

```java
// org.apache.zookeeper.server.DatadirCleanupManager#start
public void start() {
    ...
    // 初始化一个定时调度器
    timer = new Timer("PurgeTask", true);

    // 初始化一个任务对象，清理快照线程
    TimerTask task = new PurgeTask(dataLogDir, snapDir, snapRetainCount);

    // 开始调度（每隔1个小时，来执行一下 task）
    timer.scheduleAtFixedRate(task, 0, TimeUnit.HOURS.toMillis(purgeInterval));
    purgeTaskStatus = PurgeTaskStatus.STARTED;
}
// org.apache.zookeeper.server.DatadirCleanupManager.PurgeTask#run
public void run() {
    LOG.info("Purge task started.");
    try {
        // 执行清理快照
        PurgeTxnLog.purge(new File(logsDir), new File(snapsDir), snapRetainCount);

    } catch(Exception e) {
        LOG.error("Error occurred while purging.", e);
    }
    LOG.info("Purge task completed.");
}
// org.apache.zookeeper.server.PurgeTxnLog#purge
public static void purge(File dataDir, File snapDir, int num) throws IOException {
    // 强制保留最少3个快照
    if(num < 3) {
        throw new IllegalArgumentException(COUNT_ERR_MSG);
    }
    // 初始化一个快照对象 FileTxnSnapLog
    FileTxnSnapLog txnLog = new FileTxnSnapLog(dataDir, snapDir);
    // 先找出最近的 N 个快照
    List<File> snaps = txnLog.findNRecentSnapshots(num);
    int numSnaps = snaps.size();
    // 清除旧快照
    if(numSnaps > 0) {
        purgeOlderSnapshots(txnLog, snaps.get(numSnaps - 1));
    }
}
```

#### ZKDatabase

数据恢复就是从快照和日志中重新构建 `DataTree`。

```java
// runFromConfig(config)
// 构建 quorumPeer 然后 start；quorumPeer 是 Zookeeper 主体
// start 方法包含数据恢复和选举，选举部分待会介绍
// org.apache.zookeeper.server.quorum.QuorumPeer#start
public synchronized void start() {
    loadDataBase();
    ...
}
// org.apache.zookeeper.server.quorum.QuorumPeer#loadDataBase
private void loadDataBase() {
    // 从 snap dir 获取 updatingEpoch 文件
    File updating = new File(getTxnFactory().getSnapDir(), UPDATING_EPOCH_FILENAME);
    
    zkDb.loadDataBase();
    long lastProcessedZxid = zkDb.getDataTree().lastProcessedZxid;

    // 从 zxid(前32位是 epoch 的值) 中获取 epoch ==> 切换leader会导致 epoch 加一，防止脑裂
    long epochOfZxid = ZxidUtils.getEpochFromZxid(lastProcessedZxid);
    ...
}
```

loadDataBase 除了恢复数据，还有一个重要操作就是提取 `zxid` (每个事务都有唯一的 `zxid`)。`zxid`  表示当前节点已处理的最大事务，后续选举时 `zxid` 大的节点会成为 `Leader` (保证事务不丢)。

```java
// org.apache.zookeeper.server.ZKDatabase#loadDataBase
public long loadDataBase() throws IOException {
    // 从磁盘加载数据， 从快照恢复数据
    // snapLog 的实现类是： FileSnapTxnLog
    long zxid = snapLog.restore(dataTree, sessionsWithTimeouts, commitProposalPlaybackListener);

    initialized = true;
    return zxid;
}
public long restore(DataTree dt, Map<Long, Integer> sessions, PlayBackListener listener) throws IOException {

    // 一：先把 已经持久化到磁盘的 快照数据反序列化到内存中
    snapLog.deserialize(dt, sessions);

    // 二：从 ComittedLogs 去加载数据。
    // 在最后一次快照之后，还有一些新的 事务被提交了，这些事务记录在日志文件
    return fastForwardFromEdits(dt, sessions, listener);
}
// org.apache.zookeeper.server.persistence.FileSnap#deserialize
public long deserialize(DataTree dt, Map<Long, Integer> sessions) throws IOException {
    // 倒序返回快照
    List<File> snapList = findNValidSnapshots(100);
    if(snapList.size() == 0) {
        return -1L;
    }
    File snap = null;
    boolean foundValid = false;
    for(int i = 0; i < snapList.size(); i++) {
        snap = snapList.get(i);
        ...
        // 序列化，DataNode 构建成 DataTree
        deserialize(dt, sessions, ia);
        ...
        break;
    }
    ...
    // 获取有效快照中的最大的 zxid， 从文件名中获取
    dt.lastProcessedZxid = Util.getZxidFromName(snap.getName(), SNAPSHOT_FILE_PREFIX);
    return dt.lastProcessedZxid;
}
// org.apache.zookeeper.server.persistence.FileTxnSnapLog#fastForwardFromEdits
public long fastForwardFromEdits(DataTree dt, Map<Long, Integer> sessions, PlayBackListener listener) throws IOException {
    FileTxnLog txnLog = new FileTxnLog(dataDir);

    // 加载 comitted logs 的时候，从 lastProcessedZxid 开始
    TxnIterator itr = txnLog.read(dt.lastProcessedZxid + 1);
    long highestZxid = dt.lastProcessedZxid;

    TxnHeader hdr;
    try {
        while(true) {
            hdr = itr.getHeader();
            if(hdr == null) {
                // 结束返回
                return dt.lastProcessedZxid;
            }
            // 校验
            if(hdr.getZxid() < highestZxid && highestZxid != 0) {
                LOG.error("{}(higestZxid) > {}(next log) for type {}", new Object[]{highestZxid, hdr.getZxid(), hdr.getType()});
            } else {
                // lastProcessedZxid 更新
                highestZxid = hdr.getZxid();
            }
            try {
                // replay log 把事务日志文件中的数据，进行复现，恢复到 DataTree
                processTransaction(hdr, dt, sessions, itr.getTxn());

            } catch(KeeperException.NoNodeException e) {
                throw new IOException("Failed to process transaction type: " + hdr.getType() + " error: " + e.getMessage(), e);
            }
            listener.onTxnLoaded(hdr, itr.getTxn());
            // 结束跳出循环
            if(!itr.next())
                break;
        }
    ...
    // 最新的zxid
    return highestZxid;
}
```

### 选举

上一小节介绍了 Zookeeper 启动时的数据恢复，接下来就进入的选举流程。选举过程涉及到的东西比较多，先看下图有个感性的认识和后再去看源码



### 同步数据







