---
layout:     post
title:      Executor Locality
subtitle:   
date:       2021-04-03
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Locality
    - Alluxio
---

翻译 https://www.alluxio.io/blog/top-10-tips-for-making-the-spark-alluxio-stack-blazing-fast/



Apache Spark+Alluxio 技术栈非常流行，特别是在跨S3和HDFS统一数据访问方面。此外，计算和存储分离是趋势，导致查询的延迟越来越大。Alluxio 被用作计算端虚拟存储以提高性能。但要获得最佳性能，就像任何技术堆栈一样，您需要遵循最佳实践。本文提供了在Alluxio 上运行Spark时为实际工作负载进行性能调整的前10个技巧(只翻译前4个关于数据本地化的内容)，其中**数据本地化**提供了最大的好处。

高数据本地化可以极大地提高Spark 作业的性能。当数据本地化时，Spark tasks可以以内存速度（在配置ramdisk时）从本地Alluxio Worker读取Alluxio 数据，而不是通过网络传输数据。

##### 检查是否实现数据本地化

当Spark workers 与Alluxio workers 在同一机器运行Spark tasks 并执行短路读写时(无需通过网络拉数据，数据在同台机器)，Alluxio 可以提供最佳性能。有多种方法可以检查I/O请求是否由短路读/写提供服务。

运行Spark作业时，在Alluxio metrics UI页面上监控“短路读取”和“来自远程实例”的指标；或者监控`cluster.BytesReadAlluxioThroughput` and `cluster.BytesReadLocalThroughput` 指标。如果本地吞吐量为零或明显低于总吞吐量，则此作业可能不是与本地Alluxio 进行交互。利用`dstat`等工具检测网络流量，这将允许我们查看是否发生了短路读取，以及网络流量吞吐量是什么样子。有Yarn 权限的用户可以**/var/log/hadoop-yarn/userlogs **查看是否有短路读：

```shell
INFO ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms
```

你可以跑 Spark 任务来收集和处理这些日志统计和分析短路读情况。

如果Spark作业只有一小部分由Alluxio 提供的短路读取或写入，请阅读以下几个技巧以改进数据本地化。

##### 确保 Spark Executor 本地化

Spark 有两种不同级别的数据本地化实现。

- Executor Locality

YARN 调度

Spark 在运行时，由资源调度器(Mesos/Yarn)将Spark 任务分配到有 Alluxio 运行的节点上。如果未实现本地化，那么Spark 不可能在本地机器读取到 Alluxio上的数据。相比于 Spark Standalone 集群，由资源管理调度的Spark 任务，本地化问题更多。

Spark 最新版本支持在Mesos/Yarn 运行的程序在调度时考虑数据本地化，但还有其他方式也可以实现：一个简单的策略是在每个节点上启动一个 executor，因此至少在所需的节点上总会有一个 executor。实际上，由于资源限制，在生产环境中的每个节点上部署 Alluxio worker可能不适用。可以利用资源管理框架的特性：`Yarn node label`，将Alluxio worker与计算节点一起部署。有运行 Alluxio的NodeManager 打上 'alluxio' 标签，提交Spark 任务时选择  'alluxio' 标签机器，那么 Spark executor 和 Alluxio 在同一台机器上。

- Task Locality

Spark task 调度

Spark task scheduler 首先从Alluxio收集所有数据位置，作为Alluxio Worker的主机名列表，然后尝试匹配调度的执行器主机名。

- Spark 调度器本地化的优先级
  - `spark.locality.wait`：spark 调度的task 本地化的等待时间。在数据机器上启动后续的 spark task容许等待的时间，本地化等级分为: 进程内、相同节点、相同机架。Spark 切分DAG 后有shuffle ，遵循移动计算而不是移动数据原则，计算最快的方式是在当前机器执行后续task。但当前节点此时可能没有资源，那么Spark 调度器允许等待 `spark.locality.wait`，若时间段内该节点有空闲资源，就在此节点启动新task否则移动数据到其他节点开始计算。
  - `spark.locality.wait.node`：为每个节点设置等待时间



