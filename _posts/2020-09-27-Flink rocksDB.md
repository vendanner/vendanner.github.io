---
layout:     post
title:      Flink rocksDB
subtitle:   
date:       2020-09-27
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - bigdata
    - RocksDB
---

Flink 状态后端有三种：

- Memory
- FileSystem
- RocksDB

大状态的作业下，RocksDB 是首选，它支持**增量**。当使用 RocksDB 作为状态后端时，常常会发现相同的错误导致 `task` 失败。具体报错信息如下：

Flink Web-UI 上的报错信息

```shell
org.apache.flink.runtime.io.network.netty.exception.RemoteTransportException: Connection unexpectedly closed by remote task manager 'ip:2899'. This might indicate that the remote task manager was lost.
	at org.apache.flink.runtime.io.network.netty.CreditBasedPartitionRequestClientHandler.channelInactive(CreditBasedPartitionRequestClientHandler.java:136)
	at org.apache.flink.shaded.netty4.io.netty.channel.AbstractChannelHandlerContext.invokeChannelInactive(AbstractChannelHandlerContext.java:257)
	at org.apache.flink.shaded.netty4.io.netty.channel.AbstractChannelHandlerContext.invokeChannelInactive(AbstractChannelHandlerContext.java:243)
	at org.apache.flink.shaded.netty4.io.netty.channel.AbstractChannelHandlerContext.fireChannelInactive(AbstractChannelHandlerContext.java:236)
	...
```

Yarn 日志

```shell
Current usage: 10.0 GB of 10 GB physical memory used; 
19.1 GB of 21 GB virtual memory used.
Killing container
```

结合上面的日志，我们可以得出结论：Flink TM 上的物理内存超了，Yarn 就将对应的Container kill ，这样 Flink 程序就找不到 TM 导致 task 失败。

结合 [Flink 内存管理](https://vendanner.github.io/2020/08/25/Flink-%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/) 可知，状态管理内存是堆外内存，至此我们猜想大概率是 RocksDB 导致 `task` 失败。

### 参数

既然知道是 RocksDB 的锅，那我们就需要调调参数(以下内容摘抄[参考资料二](https://www.jianshu.com/p/b337b693fb8d))。

#### Tuning MemTable

memtable 作为 LSM Tree 体系里的读写缓存，对写性能有较大的影响。以下是一些值得注意的参数。为方便对比，下文都会将 RocksDB 的原始参数名与 Flink 配置中的参数名一并列出，用竖线分割。

- `write_buffer_size` | `state.backend.rocksdb.writebuffer.size`

单个 memtable 的大小，默认是 64MB。当 memtable 大小达到此阈值时，就会被标记为不可变。一般来讲，适当增大这个参数可以减小写放大带来的影响，但同时会增大 flush 后 L0、L1 层的压力，所以还需要配合修改 compaction 参数，后面再提。

- `max_write_buffer_number` | `state.backend.rocksdb.writebuffer.count`

memtable的最大数量（包含活跃的和不可变的），默认是 2。当全部 memtable 都写满但是 flush 速度较慢时，就会造成写停顿，所以如果内存充足或者使用的是机械硬盘，建议适当调大这个参数，如4。

- `min_write_buffer_number_to_merge` | `state.backend.rocksdb.writebuffer.number-to-merge`

在flush发生之前被合并的 memtable 最小数量，默认是1。举个例子，如果此参数设为2，那么当有至少两个不可变memtable 时，才有可能触发 flush（亦即如果只有一个不可变 memtable，就会等待）。调大这个值的好处是可以使更多的更改在 flush 前就被合并，降低写放大，但同时又可能增加读放大，因为读取数据时要检查的 memtable 变多了。经测试，该参数设为2或3相对较好。

#### Tuning Block/Block Cache

block是sstable的基本存储单位。block cache 则扮演读缓存的角色，采用LRU算法存储最近使用的block，对读性能有较大的影响。

- `block_size` | `state.backend.rocksdb.block.blocksize`

block的大小，默认值为 4KB。在生产环境中总是会适当调大一些，一般 32KB 比较合适，对于机械硬盘可以再增大到128~256KB，充分利用其顺序读取能力。但是需要注意，如果 block 大小增大而 block cache 大小不变，那么缓存的 block 数量会减少，无形中会增加读放大。

- `block_cache_size` | `state.backend.rocksdb.block.cache-size`

block cache 的大小，默认为 8MB。由上文所述的读写流程可知，较大的 block cache 可以有效避免热数据的读请求落到sstable 上，所以若内存余量充足，建议设置到128MB甚至256MB，读性能会有非常明显的提升。

#### Tuning Compaction

compaction 在所有基于 LSM Tree 的存储引擎中都是开销最大的操作，弄不好的话会非常容易阻塞读写。

- `compaction_style` | `state.backend.rocksdb.compaction.style`

compaction算法，使用默认的LEVEL（即 leveled compaction）即可，下面的参数也是基于此。

- `target_file_size_base` | `state.backend.rocksdb.compaction.level.target-file-size-base`

L1层单个sstable文件的大小阈值，默认值为64MB。每向上提升一级，阈值会乘以因子`target_file_size_multiplier`（但默认为1，即每级 sstable 最大都是相同的）。显然，增大此值可以降低compaction 的频率，减少写放大，但是也会造成旧数据无法及时清理，从而增加读放大。此参数不太容易调整，一般不建议设为 256MB 以上。

- `max_bytes_for_level_base`  |  `state.backend.rocksdb.compaction.level.max-size-level-base`

L1层的数据总大小阈值，默认值为 256MB。每向上提升一级，阈值会乘以因子`max_bytes_for_level_multiplier`（默认值为10 ）。由于上层的大小阈值都是以它为基础推算出来的，所以要小心调整。建议设为`target_file_size_base`的倍数，且不能太小，例如 5~10 倍。

- `level_compaction_dynamic_level_bytes` | `state.backend.rocksdb.compaction.level.use-dynamic-size`

这个参数之前讲过。当开启之后，上述阈值的乘法因子会变成除法因子，能够动态调整每层的数据量阈值，使得较多的数据可以落在最高一层，能够减少空间放大，整个 LSM Tree 的结构也会更稳定。对于机械硬盘的环境，强烈建议开启。

#### Generic Parameters

- `max_open_files` | `state.backend.rocksdb.files.open`

顾名思义，是 RocksDB 实例能够打开的最大文件数，默认为-1，表示不限制。由于 sstable 的索引和布隆过滤器默认都会驻留内存，并占用文件描述符，所以如果此值太小，索引和布隆过滤器无法正常加载，就会严重拖累读取性能。

- `max_background_compactions`/`max_background_flushes` | `state.backend.rocksdb.thread.num`

后台负责 flush 和 compaction 的最大并发线程数，默认为1。注意Flink将这两个参数合二为一处理（对应DBOptions.setIncreaseParallelism()方法 ），鉴于 flush 和 compaction 都是相对重的操作，如果 CPU 余量比较充足，建议调大，在我们的实践中一般设为4。

最后总结如下，如果需要了解更详细内容请看[参考资料三](https://cloud.tencent.com/developer/article/1592441)

![](https://vendanner.github.io/img/Flink/Flink_RocksDB_Param.png)

### 调参

针对本例，我们调整以下参数

> state.backend.rocksdb.thread.num=4
>
> taskmanager.memory.managed.fraction=0.6
>
> taskmanager.network.netty.server.numThreads=2 

针对 RocksDB，我们调大状态内存比例(增大内存)并增加 flush 和 compaction 的线程数。



## 参考资料

[Flink 清理过期 Checkpoint 目录的正确姿势](https://mp.weixin.qq.com/s/oh53V_IQwgrD_GPRht1F5A)

[Flink RocksDB状态后端参数调优实践](https://www.jianshu.com/p/b337b693fb8d)

[Flink on RocksDB 参数调优指南](https://cloud.tencent.com/developer/article/1592441)

[flink 1.10 on yarn 内存超用，被kill](http://apache-flink.147419.n8.nabble.com/flink-1-10-on-yarn-kill-td4059.html)