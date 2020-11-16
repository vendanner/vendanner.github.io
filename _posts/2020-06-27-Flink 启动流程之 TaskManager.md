---
layout:     post
title:      Flink 启动流程之 TaskManager
subtitle:   
date:       2020-06-27
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - bigdata
---

Flink 1.10 

### TaskManagerRunner

之前分析的 `start-cluster` 脚本中，启动TaskManager 是去执行 `org.apache.flink.runtime.taskexecutor.TaskManagerRunner`

```java
// org.apache.flink.runtime.taskexecutor.TaskManagerRunner
/**
 * This class is the executable entry point for the task manager in yarn or standalone mode.
 * It constructs the related components (network, I/O manager, memory manager, RPC service, HA service)
 * and starts them.
 */
public class TaskManagerRunner implements FatalErrorHandler, AutoCloseableAsync {
	//  Static entry point
	public static void main(String[] args) throws Exception {
		// startup checks and logging
		EnvironmentInformation.logEnvironmentInfo(LOG, "TaskManager", args);
		SignalHandler.register(LOG);
		JvmShutdownSafeguard.installAsShutdownHook(LOG);
		long maxOpenFileHandles = EnvironmentInformation.getOpenFileHandlesLimit();
		if (maxOpenFileHandles != -1L) {
			LOG.info("Maximum number of open file descriptors is {}.", maxOpenFileHandles);
		} else {
			LOG.info("Cannot determine the maximum number of open file descriptors");
		}
		runTaskManagerSecurely(args, ResourceID.generate());
	}
  public static void runTaskManagerSecurely(String[] args, ResourceID resourceID) {
    try {
      final Configuration configuration = loadConfiguration(args);
      FileSystem.initialize(configuration, PluginUtils.createPluginManagerFromRootFolder(configuration));
      SecurityUtils.install(new SecurityConfiguration(configuration));
      SecurityUtils.getInstalledContext().runSecured(() -> {
        runTaskManager(configuration, resourceID);
        return null;
      });
    } 
    ...
  }
  public static void runTaskManagerSecurely(String[] args, ResourceID resourceID) {
    try {
      final Configuration configuration = loadConfiguration(args);
      FileSystem.initialize(configuration, PluginUtils.createPluginManagerFromRootFolder(configuration));

      SecurityUtils.install(new SecurityConfiguration(configuration));

      SecurityUtils.getInstalledContext().runSecured(() -> {
        runTaskManager(configuration, resourceID);
        return null;
      });
    } 
    ...
  }
```

 流程类似 `JobManager` 

- 读取配置文件
- 初始化配置⽂文件中的共享⽂文件设置
- 获取 `Hadoop security context`

```java
// org.apache.flink.runtime.taskexecutor.TaskManagerRunner
public static void runTaskManager(Configuration configuration, ResourceID resourceId) throws Exception {
  final TaskManagerRunner taskManagerRunner = new TaskManagerRunner(configuration, resourceId);
  taskManagerRunner.start();
}
// TaskManagerRunner 初始化
public TaskManagerRunner(Configuration configuration, ResourceID resourceId) throws Exception {
  this.configuration = checkNotNull(configuration);
  this.resourceId = checkNotNull(resourceId);

  timeout = AkkaUtils.getTimeoutAsTime(configuration);

  this.executor = java.util.concurrent.Executors.newScheduledThreadPool(
    Hardware.getNumberCPUCores(),
    new ExecutorThreadFactory("taskmanager-future"));
  // 老套路，创建一堆 service：ha，rpc，metric，blobCache，心跳
  ...
  // 创建 TaskExecutor
  taskManager = startTaskManager(
    this.configuration,
    this.resourceId,
    rpcService,
    highAvailabilityServices,
    heartbeatServices,
    metricRegistry,
    blobCacheService,
    false,
    this);
	...
}
public static TaskExecutor startTaskManager(
			Configuration configuration,
			ResourceID resourceID,
			RpcService rpcService,
			HighAvailabilityServices highAvailabilityServices,
			HeartbeatServices heartbeatServices,
			MetricRegistry metricRegistry,
			BlobCacheService blobCacheService,
			boolean localCommunicationOnly,
			FatalErrorHandler fatalErrorHandler) throws Exception {
    ...
		InetAddress remoteAddress = InetAddress.getByName(rpcService.getAddress());
    // TaskManager 资源:CPU,MEMORY,NETWORK
		final TaskExecutorResourceSpec taskExecutorResourceSpec = TaskExecutorResourceUtils.resourceSpecFromConfig(configuration);

		TaskManagerServicesConfiguration taskManagerServicesConfiguration =
			TaskManagerServicesConfiguration.fromConfiguration(
				configuration,
				resourceID,
				remoteAddress,
				localCommunicationOnly,
				taskExecutorResourceSpec);
   // metric
		Tuple2<TaskManagerMetricGroup, MetricGroup> taskManagerMetricGroup = MetricUtils.instantiateTaskManagerMetricGroup(
			metricRegistry,
			TaskManagerLocation.getHostName(remoteAddress),
			resourceID,
			taskManagerServicesConfiguration.getSystemResourceMetricsProbingInterval());
    // service:taskSlotTable,broadcastVariableManager,taskStateManager,JobManagerTable ...
		TaskManagerServices taskManagerServices = TaskManagerServices.fromConfiguration(
			taskManagerServicesConfiguration,
			taskManagerMetricGroup.f1,
			rpcService.getExecutor()); 
		TaskManagerConfiguration taskManagerConfiguration =
			TaskManagerConfiguration.fromConfiguration(configuration, taskExecutorResourceSpec);

		String metricQueryServiceAddress = metricRegistry.getMetricQueryServiceGatewayRpcAddress();

		return new TaskExecutor(
			rpcService,
			taskManagerConfiguration,
			highAvailabilityServices,
			taskManagerServices,
			heartbeatServices,
			taskManagerMetricGroup.f0,
			metricQueryServiceAddress,
			blobCacheService,
			fatalErrorHandler,
			new TaskExecutorPartitionTrackerImpl(taskManagerServices.getShuffleEnvironment()),
      // 反压
			createBackPressureSampleService(configuration, rpcService.getScheduledExecutor()));
	}
```

> 创建 `TaskExector`  所需的各种服务和组件

### TaskExecutor

```java
// org.apache.flink.runtime.taskexecutor.TaskExecutor
/**
 * TaskExecutor implementation. The task executor is responsible for the execution of multiple
 */
public class TaskExecutor extends RpcEndpoint implements TaskExecutorGateway {
	/**
	 * Triggers start of the rpc endpoint. This tells the underlying rpc server that the rpc endpoint is ready
	 * to process remote procedure calls.
	 *
	 * @throws Exception indicating that something went wrong while starting the RPC endpoint
	 */
  // org.apache.flink.runtime.rpc.RpcEndpoint
  // rpc 端已准备完毕，执行结束后会触发 onStart 方法
	public final void start() {
		rpcServer.start();
	}
  
  @Override
	public void onStart() throws Exception {
		try {
			startTaskExecutorServices();
		}
    ...
		startRegistrationTimeout();
	}
  private void startTaskExecutorServices() throws Exception {
		try {
			// 注册到 ResourceManager
			resourceManagerLeaderRetriever.start(new ResourceManagerLeaderListener());

			// tell the task slot table who's responsible for the task slot actions
			taskSlotTable.start(new SlotActionsImpl(), getMainThreadExecutor());

			// start the job leader service
			jobLeaderService.start(getAddress(), getRpcService(), haServices, new JobLeaderListenerImpl());

			fileCache = new FileCache(taskManagerConfiguration.getTmpDirectories(), blobCacheService.getPermanentBlobService());
		} catch (Exception e) {
			handleStartTaskExecutorServicesException(e);
		}
	}
}
```

#### ResourceManagerLeaderRetriever

```java
// org.apache.flink.runtime.leaderretrieval.StandaloneLeaderRetrievalService
public void start(LeaderRetrievalListener listener) {
		synchronized (startStopLock) {
			checkState(!started, "StandaloneLeaderRetrievalService can only be started once.");
			started = true;

			// directly notify the listener, because we already know the leading JobManager's address
			listener.notifyLeaderAddress(leaderAddress, leaderId);
		}
	}
// org.apache.flink.runtime.taskexecutor.TaskExecutor
private final class ResourceManagerLeaderListener implements LeaderRetrievalListener {
  @Override
  public void notifyLeaderAddress(final String leaderAddress, final UUID leaderSessionID) {
    runAsync(
   // This method is called by the {@link LeaderRetrievalService} when a new leader is elected
      () -> notifyOfNewResourceManagerLeader(
        leaderAddress,
        ResourceManagerId.fromUuidOrNull(leaderSessionID)));
  }
  ...
}
private void notifyOfNewResourceManagerLeader(String newLeaderAddress, ResourceManagerId newResourceManagerId) {
  resourceManagerAddress = createResourceManagerAddress(newLeaderAddress, newResourceManagerId);
  reconnectToResourceManager(new FlinkException(String.format("ResourceManager leader changed to new address %s", resourceManagerAddress)));
}
```

> 创建 ResourceManagerLeaderListener 实例，该类中实现 LeaderRetrievalListener 接口的 notifyLeaderAddress ⽅法，⼀旦有新的 leader 选举时，LeaderRetrievalService 将调⽤用此⽅方法。重写后的 notifyLeaderAddress 方法里面会通知新的资源管理器 leader。

#### TaskSlotTable

```java
// org.apache.flink.runtime.taskexecutor.TaskExecutor
private class SlotActionsImpl implements SlotActions {
  /**
	 * Free the task slot with the given allocation id.
	 * @param allocationId to identify the slot to be freed
	 */
  @Override
  public void freeSlot(final AllocationID allocationId) {
    runAsync(() ->
             freeSlotInternal(
               allocationId,
               new FlinkException("TaskSlotTable requested freeing the TaskSlot " + allocationId + '.')));
  }
 ...
}
// 释放 slot 逻辑
private void freeSlotInternal(AllocationID allocationId, Throwable cause) {
	...
  try {
    final JobID jobId = taskSlotTable.getOwningJob(allocationId);
    final int slotIndex = taskSlotTable.freeSlot(allocationId, cause);
    if (slotIndex != -1) {
      if (isConnectedToResourceManager()) {
        // the slot was freed. Tell the RM about it
        ResourceManagerGateway resourceManagerGateway = establishedResourceManagerConnection.getResourceManagerGateway();
        resourceManagerGateway.notifySlotAvailable(
          establishedResourceManagerConnection.getTaskExecutorRegistrationId(),
          new SlotID(getResourceID(), slotIndex),
          allocationId);
      }
      ...
}
// org.apache.flink.runtime.taskexecutor.slot.TaskSlotTableImpl
public void start(SlotActions initialSlotActions, ComponentMainThreadExecutor mainThreadExecutor) {
  Preconditions.checkState(
    state == State.CREATED,
    "The %s has to be just created before starting",
    TaskSlotTableImpl.class.getSimpleName());
  this.slotActions = Preconditions.checkNotNull(initialSlotActions);
  this.mainThreadExecutor = Preconditions.checkNotNull(mainThreadExecutor);
  timerService.start(this);
  state = State.RUNNING;
}
private CompletableFuture<Void> freeSlotInternal(TaskSlot<T> taskSlot, Throwable cause) {
  AllocationID allocationId = taskSlot.getAllocationId();
	...
  if (taskSlot.isEmpty()) {
    // remove the allocation id to task slot mapping
    allocatedSlots.remove(allocationId);
    // unregister a potential timeout
    timerService.unregisterTimeout(allocationId);
    JobID jobId = taskSlot.getJobId();
    Set<AllocationID> slots = slotsPerJob.get(jobId);
    ...
    slots.remove(allocationId);

    if (slots.isEmpty()) {
      slotsPerJob.remove(jobId);
    }

    taskSlots.remove(taskSlot.getIndex());
    budgetManager.release(taskSlot.getResourceProfile());
  }
  return taskSlot.closeAsync(cause);
}
```

`SlotActionsImpl` 实现 `SlotActions` ，重写 `freeSlot` 逻辑。在剖析释放流程之前，先看看 `AllocationID`

> Unique identifier for a physical slot allocated by a JobManager via the ResourceManager
> from a TaskManager. The ID is assigned once the JobManager (or its SlotPool) first
> requests the slot and is constant across retries.
>
> This ID is used by the TaskManager and ResourceManager to track and synchronize which
> slots are allocated to which JobManager and which are free.
>
> In contrast to this AllocationID, the {@link org.apache.flink.runtime.jobmaster.SlotRequestId}
> is used when a task requests a logical slot from the SlotPool. Multiple logical slot requests
> can map to one physical slot request (due to slot sharing).
>
> 大意就是物理 slot 的标识符，在第一次分配的时候就确定了。

- 通过 `AllocationID` 得到 `TaskSlot`（任务槽，flink 优化后可能多个 task 都在一个 solt），然后再得到 `JobID`
- `TaskSlotTableImpl.freeSlotInternal` 中 将 此 `allocationId` 从 allocatedSlots、timerService、slots 移除，并把对应的 `taskSlot` 从 `taskSlots` 移除
- slot 释放后，通知 `RM` ；

#### jobLeaderService

建立 `JobManager`  连接

```java
// org.apache.flink.runtime.taskexecutor.JobLeaderService
public void start(
  final String initialOwnerAddress,
  final RpcService initialRpcService,
  final HighAvailabilityServices initialHighAvailabilityServices,
  final JobLeaderListener initialJobLeaderListener) {
   ...
    LOG.info("Start job leader service.");

    this.ownerAddress = Preconditions.checkNotNull(initialOwnerAddress);
    this.rpcService = Preconditions.checkNotNull(initialRpcService);
    this.highAvailabilityServices = Preconditions.checkNotNull(initialHighAvailabilityServices);
    this.jobLeaderListener = Preconditions.checkNotNull(initialJobLeaderListener);
    state = JobLeaderService.State.STARTED;
  }
}
// org.apache.flink.runtime.taskexecutor.TaskExecutor
// The listener is notified whenever a job manager gained leadership for a registered job and the service could establish a connection to it
private final class JobLeaderListenerImpl implements JobLeaderListener {
  @Override
  public void jobManagerGainedLeadership(
    final JobID jobId,
    final JobMasterGateway jobManagerGateway,
    final JMTMRegistrationSuccess registrationMessage) {
    runAsync(
      () ->
      establishJobManagerConnection(
        jobId,
        jobManagerGateway,
        registrationMessage));
   ...
  }
```

#### FileCache

```java
// org.apache.flink.runtime.filecache.FileCache
// The FileCache is used to access registered cache files when a task is deployed
public FileCache(String[] tempDirectories, PermanentBlobService blobService) throws IOException {
  this (tempDirectories, blobService, Executors.newScheduledThreadPool(10,
       new ExecutorThreadFactory("flink-file-cache")), 5000);
}
@VisibleForTesting
FileCache(String[] tempDirectories, PermanentBlobService blobService,
          ScheduledExecutorService executorService, long cleanupInterval) throws IOException {
  Preconditions.checkNotNull(tempDirectories);
  this.cleanupInterval = cleanupInterval;
  storageDirectories = new File[tempDirectories.length];
  for (int i = 0; i < tempDirectories.length; i++) {
    String cacheDirName = "flink-dist-cache-" + UUID.randomUUID().toString();
    storageDirectories[i] = new File(tempDirectories[i], cacheDirName);
    String path = storageDirectories[i].getAbsolutePath();

    if (storageDirectories[i].mkdirs()) {
      LOG.info("User file cache uses directory " + path);
    } 
    ...
}
```

新建⼀个 FileCache 的实例，当部署任务时 FileCache ⽤于为已注册的缓存文件创建本地文件。





## 参考资料

[Flink 源码阅读笔记（5）- 集群启动流程](https://blog.jrwang.me/2019/flink-source-code-bootstarp/#taskmanager-的启动)