---
layout:     post
title:      Flink 源码之构建 ExecutionGraph
subtitle:   
date:       2020-11-05
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - bigdata
---

Flink 1.11

>Flink 任务在运行之前会经历以下几个阶段：
>
>Program -> StreamGraph -> JobGraph -> ExecutionGraph -> 物理执行计划

![](https://vendanner.github.io/img/Flink/generateDAG.jpg)

`JobGraph` 生成 `ExecutionGraph`

> `JobVertex` DAG **提交任务**以后(JobManager 生成)，从 Source 节点开始排序，根据 JobVertex 生成 `ExecutionJobVertex`，根据 `jobVertex`的 `IntermediateDataSet` 构建 `IntermediateResult`，然后 `IntermediateResult` 构建上下游的依赖关系，形成 `ExecutionJobVertex` 层面的 DAG 即 `ExecutionGraph`。

[JobManager](https://vendanner.github.io/2020/06/04/Flink-%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E4%B9%8B-JobManager/) 在会启动一系列服务，其中包含 **Dispatcher**

``` java
// org.apache.flink.runtime.entrypoint.component.DefaultDispatcherResourceManagerComponentFactory
dispatcherRunner = dispatcherRunnerFactory.createDispatcherRunner(...)
// org.apache.flink.runtime.dispatcher.runner.DefaultDispatcherRunnerFactory
public DispatcherRunner createDispatcherRunner(
    LeaderElectionService leaderElectionService,
    FatalErrorHandler fatalErrorHandler,
    JobGraphStoreFactory jobGraphStoreFactory,
    Executor ioExecutor,
    RpcService rpcService,
    PartialDispatcherServices partialDispatcherServices) throws Exception {

  final DispatcherLeaderProcessFactory dispatcherLeaderProcessFactory = dispatcherLeaderProcessFactoryFactory.createFactory(
    jobGraphStoreFactory,
    ioExecutor,
    rpcService,
    partialDispatcherServices,
    fatalErrorHandler);

  return DefaultDispatcherRunner.create(
    leaderElectionService,
    fatalErrorHandler,
    dispatcherLeaderProcessFactory);
}
// org.apache.flink.runtime.dispatcher.runner.JobDispatcherLeaderProcessFactoryFactory
public DispatcherLeaderProcessFactory createFactory(
    JobGraphStoreFactory jobGraphStoreFactory,
    Executor ioExecutor,
    RpcService rpcService,
    PartialDispatcherServices partialDispatcherServices,
    FatalErrorHandler fatalErrorHandler) {

  final JobGraph jobGraph;

  try {
    jobGraph = jobGraphRetriever.retrieveJobGraph(partialDispatcherServices.getConfiguration());
  } catch (FlinkException e) {
    throw new FlinkRuntimeException("Could not retrieve the JobGraph.", e);
  }

  final DefaultDispatcherGatewayServiceFactory defaultDispatcherServiceFactory = new DefaultDispatcherGatewayServiceFactory(
    JobDispatcherFactory.INSTANCE,
    rpcService,
    partialDispatcherServices);

  return new JobDispatcherLeaderProcessFactory(
    defaultDispatcherServiceFactory,
    jobGraph,
    fatalErrorHandler);
}
// org.apache.flink.runtime.dispatcher.runner.JobDispatcherLeaderProcessFactory
JobDispatcherLeaderProcessFactory(
    AbstractDispatcherLeaderProcess.DispatcherGatewayServiceFactory dispatcherGatewayServiceFactory,
    JobGraph jobGraph,
    FatalErrorHandler fatalErrorHandler) {
  this.dispatcherGatewayServiceFactory = dispatcherGatewayServiceFactory;
  this.jobGraph = jobGraph;
  this.fatalErrorHandler = fatalErrorHandler;
}
// 以上已取出 JobGraph
// 接下里开始 转换
public DispatcherLeaderProcess create(UUID leaderSessionID) {
  return new JobDispatcherLeaderProcess(leaderSessionID, dispatcherGatewayServiceFactory, jobGraph, fatalErrorHandler);
}
// JM 是高可用的，当为主时会调用 Dispatcher.grantLeadership 
// DefaultDispatcherRunner.grantLeadership 
// -> DefaultDispatcherRunner.startNewDispatcherLeaderProcess
// -> DefaultDispatcherRunner.createNewDispatcherLeaderProcess
// -> JobDispatcherLeaderProcessFactory.create
// -> new JobDispatcherLeaderProcess
// -> MiniDispatcher.submitJob

```









## 参考资料

[Flink 源码阅读笔记（3）- ExecutionGraph 的生成](https://blog.jrwang.me/2019/flink-source-code-executiongraph/)

[Flink 四层转化流程](https://ververica.cn/developers/advanced-tutorial-2-flink-job-execution-depth-analysis/)

[Flink原理与实现：如何生成ExecutionGraph及物理执行图](https://blog.csdn.net/weixin_33724659/article/details/90334123)

[Flink 如何生成 ExecutionGraph](http://matt33.com/2019/12/20/flink-execution-graph-4/)

[作业调度](https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/internals/job_scheduling.html)

