---
layout:     post
title:      Hive on Spark[译]
subtitle:   
date:       2020-04-05
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Spark
    - Hive
---

Hive 默认的是 `MR` 引擎，但只要设置

> ```shell
> set hive.execution.engine=spark;
> ```

执行引擎就变成 `Spark`。但 Hive 和 Spark 版本之间是存在一些兼容星的问题

| Hive Version | --    |
| ------------ | ----- |
| master       | 2.3.0 |
| 3.0.x        | 2.3.0 |
| 2.3.x        | 2.0.0 |
| 2.2.x        | 1.6.0 |
| 2.1.x        | 1.6.0 |
| 2.0.x        | 1.5.0 |
| 1.2.x        | 1.3.1 |
| 1.1.x        | 1.2.0 |

### 配置

#### YARN

调度器设置为**公平调度**

> yarn.resourcemanager.scheduler.class=org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler

#### Spark

- 处理好兼容问题
- 运行模式为 `YARN`
- 启动 Spark 集群

#### Hive

- Hive 2.2.0 开始，要把以下 jar 拷贝到 `HIVE_HOME/lib`

  - Scala-library
  - Spark-core
  - Spark-network-common

- 参数

  - 设置执行引擎为 Spark
  - 其他参数 [Spark section of Hive Configuration Properties](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties#ConfigurationProperties-Spark) 

- Spark 配置

  - Spark 的配置：`spark-defaults.conf` 移到 Hive 下或者以下参数设置到 `hive-site.xml`

    > ```shell
    > set spark.master=
    > set spark.eventLog.enabled=``true``;
    > set spark.eventLog.dir=
    > set spark.executor.memory=512m;       
    > set spark.serializer=org.apache.spark.serializer.KryoSerializer;
    > ```

  - 加速 Spark 启动：把 SPARK_HOME/jars 下所有 jar 都上传到 HDFS 上，然后在 `hive-site.xml` 配置

    > ```xml
    > <property>
    >   <name>spark.yarn.jars</name>
    >   <value>hdfs://xxxx:8020/spark-jars/*</value>
    > </property>
    > ```

    ​		Spark 每次启动时都会将 jar 上传一遍到 HDFS (yarn 模式下)，详细参考 [Spark-on-YARN-加速启动](https://vendanner.github.io/2018/09/18/Spark-on-YARN-加速启动/)

### 调优

#### spark.executor.cores

推荐个数为 `5/6/7`，取决于 vcores 资源浪费情况。假设 `yarn.nodemanager.resource.cpu-vcores` = 19，那么推荐6，月因为 5/7的话各自有4/5 个vcore 浪费。当然如果是 vcores = 20，最好的选择是 5

#### spark.executor.memory

Executor 内存通过计算 vcores 占比

> (spark.executor.cores / yarn.nodemanager.resource.cpu-vcores) * yarn.nodemanager.resource.memory-mb

#### spark.yarn.executor.memoryOverhead

Spark executor 堆外内存，在 Spark 中默认是堆内内存的 10% 且不小于 384M。Hive on Spark ，推荐占比是 15%-20%。

#### spark.executor.instances

根据内存可以计算出每个 Hadoop node 可容许**几个** Executor ，但作业要具体要几个 Executor 要具体情况具体分析

### 问题集

[Green are resolved,will be removed from this list](https://cwiki.apache.org/confluence/display/Hive/Hive+on+Spark:+Getting+Started#HiveonSpark:GettingStarted-CommonIssues)

### 推荐配置

```shell
mapreduce.input.fileinputformat.split.maxsize=750000000
hive.vectorized.execution.enabled=true

hive.cbo.enable=true
hive.optimize.reducededuplication.min.reducer=4
hive.optimize.reducededuplication=true
hive.orc.splits.include.file.footer=false
hive.merge.mapfiles=true
hive.merge.sparkfiles=false
hive.merge.smallfiles.avgsize=16000000
hive.merge.size.per.task=256000000
hive.merge.orcfile.stripe.level=true
hive.auto.convert.join=true
hive.auto.convert.join.noconditionaltask=true
hive.auto.convert.join.noconditionaltask.size=894435328
hive.optimize.bucketmapjoin.sortedmerge=false
hive.map.aggr.hash.percentmemory=0.5
hive.map.aggr=true
hive.optimize.sort.dynamic.partition=false
hive.stats.autogather=true
hive.stats.fetch.column.stats=true
hive.vectorized.execution.reduce.enabled=false
hive.vectorized.groupby.checkinterval=4096
hive.vectorized.groupby.flush.percent=0.1
hive.compute.query.using.stats=true
hive.limit.pushdown.memory.usage=0.4
hive.optimize.index.filter=true
hive.exec.reducers.bytes.per.reducer=67108864
hive.smbjoin.cache.rows=10000
hive.exec.orc.default.stripe.size=67108864
hive.fetch.task.conversion=more
hive.fetch.task.conversion.threshold=1073741824
hive.fetch.task.aggr=false
mapreduce.input.fileinputformat.list-status.num-threads=5
spark.kryo.referenceTracking=false
spark.kryo.classesToRegister=org.apache.hadoop.hive.ql.io.HiveKey,org.apache.hadoop.io.BytesWritable,org.apache.hadoop.hive.ql.exec.vector.VectorizedRowBatch
```



## 参考资料

[Hive on Spark: Getting Started](https://cwiki.apache.org/confluence/display/Hive/Hive+on+Spark%3A+Getting+Started)