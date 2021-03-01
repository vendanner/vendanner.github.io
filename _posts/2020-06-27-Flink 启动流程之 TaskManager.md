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

TaskExecutor 才是真正的执行器。

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
      // zk HA 模式下，从 zk 上的jobid 目录下获取 resourceManager 地址
      // ResourceManagerLeaderListener 获取地址的监听器
			resourceManagerLeaderRetriever.start(new ResourceManagerLeaderListener());

			// taskSlotTable 是 taskSlot 的管理模块
      // 记录 TaskExecutor 上的 slot 状态
			taskSlotTable.start(new SlotActionsImpl(), getMainThreadExecutor());

			// 监控 JobMaster 
			jobLeaderService.start(getAddress(), getRpcService(), haServices, new JobLeaderListenerImpl());
      // 启动 FileCache 服务
			fileCache = new FileCache(taskManagerConfiguration.getTmpDirectories(), blobCacheService.getPermanentBlobService());
		} catch (Exception e) {
			handleStartTaskExecutorServicesException(e);
		}
	}
}
```

### ResourceManagerLeaderRetriever

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

> 创建 ResourceManagerLeaderListener 实例，该类中实现 LeaderRetrievalListener 接口的 notifyLeaderAddress ⽅法，⼀旦有新的 ResourceManager，LeaderRetrievalService 将调⽤用此⽅方法。重写后的 notifyLeaderAddress 方法里面会通知新的资源管理器 leader。

#### 注册

不管是第一次获取到 resourceManager 还是运行任务过程中获取到 resourceManager 地址(这种情况是resourceManager 挂了重启)，都要先注册到新的 ResourceManager。

```java
// org.apache.flink.runtime.taskexecutor.TaskExecutor
private final class ResourceManagerLeaderListener implements LeaderRetrievalListener {
  public void notifyLeaderAddress(final String leaderAddress, final UUID leaderSessionID) {
    runAsync(
      () -> notifyOfNewResourceManagerLeader(
        leaderAddress,
        ResourceManagerId.fromUuidOrNull(leaderSessionID)));
  }
  // 异常处理
}
-> notifyOfNewResourceManagerLeader
  -> reconnectToResourceManager
   -> tryConnectToResourceManager
    -> connectToResourceManager
private void connectToResourceManager() {
  log.info("Connecting to ResourceManager {}.", resourceManagerAddress);
  // TaskExecutor 注册时向 ResourceManager 发送的信息包含：
  //   地址，resourceID，slot 信息
  final TaskExecutorRegistration taskExecutorRegistration = new TaskExecutorRegistration(
    getAddress(),
    getResourceID(),
    taskManagerLocation.dataPort(),
    hardwareDescription,
    taskManagerConfiguration.getDefaultSlotResourceProfile(),
    taskManagerConfiguration.getTotalResourceProfile()
  );
  // 
  resourceManagerConnection =
    new TaskExecutorToResourceManagerConnection(
      log,
      getRpcService(),
      taskManagerConfiguration.getRetryingRegistrationConfiguration(),
      resourceManagerAddress.getAddress(),
      resourceManagerAddress.getResourceManagerId(),
      getMainThreadExecutor(),
      new ResourceManagerRegistrationListener(),
      taskExecutorRegistration);
  // 开始注册
  resourceManagerConnection.start();
}
// RPC 
-> org.apache.flink.runtime.registration.RegisteredRpcConnection#start    // 注册成功回调 org.apache.flink.runtime.registration.RegisteredRpcConnection#onRegistrationSuccess
  -> org.apache.flink.runtime.registration.RetryingRegistration#startRegistration
   -> org.apache.flink.runtime.registration.RetryingRegistration#register
    -> org.apache.flink.runtime.taskexecutor.TaskExecutorToResourceManagerConnection.ResourceManagerRegistration#invokeRegistration
     -> org.apache.flink.runtime.resourcemanager.ResourceManager#registerTaskExecutor
      -> org.apache.flink.runtime.resourcemanager.ResourceManager#registerTaskExecutorInternal
// ResourceManager 上注册一个 TaskExecutor
private RegistrationResponse registerTaskExecutorInternal(
    TaskExecutorGateway taskExecutorGateway,
    TaskExecutorRegistration taskExecutorRegistration) {
  ...
  String taskExecutorAddress = taskExecutorRegistration.getTaskExecutorAddress();
  if (newWorker == null) {
    log.warn("Discard registration from TaskExecutor {} at ({}) because the framework did " +
      "not recognize it", taskExecutorResourceId, taskExecutorAddress);
    return new RegistrationResponse.Decline("unrecognized TaskExecutor");
  } else {
    WorkerRegistration<WorkerType> registration = new WorkerRegistration<>(
      taskExecutorGateway,
      newWorker,
      taskExecutorRegistration.getDataPort(),
      taskExecutorRegistration.getHardwareDescription());

    log.info("Registering TaskManager with ResourceID {} ({}) at ResourceManager", taskExecutorResourceId, taskExecutorAddress);
    // 增加 taskExecutor 信息，代表注册成功
    taskExecutors.put(taskExecutorResourceId, registration);
    // 维持心跳
    taskManagerHeartbeatManager.monitorTarget(taskExecutorResourceId, new HeartbeatTarget<Void>() {
      public void receiveHeartbeat(ResourceID resourceID, Void payload) {
        // the ResourceManager will always send heartbeat requests to the
        // TaskManager
      }
      // 向 TaskManager 发送心跳
      public void requestHeartbeat(ResourceID resourceID, Void payload) {
        taskExecutorGateway.heartbeatFromResourceManager(resourceID);
      }
    });
    // 给 TaskExecutor 返回注册成功
    return new TaskExecutorRegistrationSuccess(
      registration.getInstanceID(),
      resourceId,
      clusterInformation);
  }
}

// TaskExecutor
// 注册成功回调 org.apache.flink.runtime.registration.RegisteredRpcConnection#onRegistrationSuccess
-> org.apache.flink.runtime.taskexecutor.TaskExecutor.ResourceManagerRegistrationListener#onRegistrationSuccess
  -> org.apache.flink.runtime.taskexecutor.TaskExecutor#establishResourceManagerConnection
private void establishResourceManagerConnection {
  // 注册成功后，首先先 Resource 回报 SlotReport
  final CompletableFuture<Acknowledge> slotReportResponseFuture = resourceManagerGateway.sendSlotReport(
    getResourceID(),
    taskExecutorRegistrationId,
    // 汇报 slot 状态：
	  // slotId，ResourceProfile，JobId，AllocationId；后两个有值表示当前 slot 已分配
    taskSlotTable.createSlotReport(getResourceID()),
    taskManagerConfiguration.getTimeout());
  ...
  // 维持和 ResourceManager 心跳
  resourceManagerHeartbeatManager.monitorTarget(resourceManagerResourceId, new HeartbeatTarget<TaskExecutorHeartbeatPayload>() {
    @Override
    public void receiveHeartbeat(ResourceID resourceID, TaskExecutorHeartbeatPayload heartbeatPayload) {
      resourceManagerGateway.heartbeatFromTaskManager(resourceID, heartbeatPayload);
    }
    public void requestHeartbeat(ResourceID resourceID, TaskExecutorHeartbeatPayload heartbeatPayload) {
      // the TaskManager won't send heartbeat requests to the ResourceManager
    }
  });
  ...
  // 停止注册超时服务：注册时启动启动超时服务，到时间没有关闭就会启用超时
  stopRegistrationTimeout();
}
```

TaskExecutor 向 ResourceManager 注册成功后，先向其发送 `SlotReport`。`SlotReport` 包含当前所有 slot 的信息。

#### 心跳

在注册成功时，ResourceManager 会与注册成功的 TaskExecutor 维持心跳；TaskExecutor 接收注册成功后，先发送 slotReport ，然后也会与 ResourceManager 维持心跳。在 Flink 中，ResourceManager、JobMaster、TaskExecutor 两两之间都会维持心跳来感知彼此的状态，若心跳超时表示服务异常会进入**容错**程序。

```java
// org.apache.flink.runtime.resourcemanager.ResourceManager#registerTaskExecutorInternal
taskManagerHeartbeatManager.monitorTarget(taskExecutorResourceId, new HeartbeatTarget<Void>() {
      public void receiveHeartbeat(ResourceID resourceID, Void payload) {
        // the ResourceManager will always send heartbeat requests to the
        // TaskManager
      }
      // 向 TaskManager 发送心跳，只携带 ResourceID，无其他信息
      public void requestHeartbeat(ResourceID resourceID, Void payload) {
        taskExecutorGateway.heartbeatFromResourceManager(resourceID);
      }
    });
```

以上代码会将 TaskExecutor 信息封装成 `HeartbeatMonitor`，并加入到 ResourceManager 的 `heartbeatTargets`。在 ResourceManager 会启动两个心跳服务：taskManagerHeartbeatManager、jobManagerHeartbeatManager。本例使用的是 TaskManagerHeartbeatManager，它会周期性(默认5s) 向 `heartbeatTargets` 中这些 target(taskExecutor) 发送心跳。

```java
// org.apache.flink.runtime.taskexecutor.TaskExecutor#establishResourceManagerConnection
resourceManagerHeartbeatManager.monitorTarget(resourceManagerResourceId, new HeartbeatTarget<TaskExecutorHeartbeatPayload>() {
    // 收到 ResourceManager 心跳返回
    public void receiveHeartbeat(ResourceID resourceID, TaskExecutorHeartbeatPayload heartbeatPayload) {
      resourceManagerGateway.heartbeatFromTaskManager(resourceID, heartbeatPayload);
    }
    public void requestHeartbeat(ResourceID resourceID, TaskExecutorHeartbeatPayload heartbeatPayload) {
      // the TaskManager won't send heartbeat requests to the ResourceManager
    }
  });
```

同理在 TaskExecutor 端也是将 ResourceManager 信息封装成 `HeartbeatMonitor` 并存入 `heartbeatTargets`。当收到 ResourceManager 后根据 ResourceID(每个资源都有唯一标识) 从 `heartbeatTargets` 取出并调用 `receiveHeartbeat`。ResourceManager 与 TaskExecutor 的心跳流向：

> ResourceManager -> resourceManagerHeartbeatManager -> HeartbeatManagerImpl -> receiveHeartbeat

```java
// org.apache.flink.runtime.heartbeat.HeartbeatManagerImpl
public void requestHeartbeat(final ResourceID requestOrigin, I heartbeatPayload) {
  if (!stopped) {
    log.debug("Received heartbeat request from {}.", requestOrigin);
    final HeartbeatTarget<O> heartbeatTarget = reportHeartbeat(requestOrigin);
    if (heartbeatTarget != null) {
      if (heartbeatPayload != null) {
        heartbeatListener.reportPayload(requestOrigin, heartbeatPayload);
      }
      // 调用 resourceManagerHeartbeatManager.monitorTarget 注册的 receiveHeartbeat
      // 返回的信息由 heartbeatListener (ResourceManagerHeartbeatListener) 提供
      heartbeatTarget.receiveHeartbeat(getOwnResourceID(), heartbeatListener.retrievePayload(requestOrigin));
    }
  }
}
// org.apache.flink.runtime.taskexecutor.TaskExecutor.ResourceManagerHeartbeatListener
private class ResourceManagerHeartbeatListener implements HeartbeatListener<Void, TaskExecutorHeartbeatPayload> {
  // 心跳超时，表示 ResourceManager 异常，TaskExecutor 向 ResourceManager 重新连接
  public void notifyHeartbeatTimeout(final ResourceID resourceId) {
    validateRunsInMainThread();
    // first check whether the timeout is still valid
    if (establishedResourceManagerConnection != null && establishedResourceManagerConnection.getResourceManagerResourceId().equals(resourceId)) {
      log.info("The heartbeat of ResourceManager with id {} timed out.", resourceId);

      reconnectToResourceManager(new TaskManagerException(
        String.format("The heartbeat of ResourceManager with id %s timed out.", resourceId)));
    } else {
      log.debug("Received heartbeat timeout for outdated ResourceManager id {}. Ignoring the timeout.", resourceId);
    }
  }
  public void reportPayload(ResourceID resourceID, Void payload) {
    // nothing to do since the payload is of type Void
  }
  // TaskExecutor 向 ResourceManager 返回的信息
  // SlotReport
  public TaskExecutorHeartbeatPayload retrievePayload(ResourceID resourceID) {
    validateRunsInMainThread();
    return new TaskExecutorHeartbeatPayload(taskSlotTable.createSlotReport(getResourceID()), partitionTracker.createClusterPartitionReport());
  }
}
```

由上可知，ResourceManager 向 TaskExecutor 发送空的心跳包，TaskExecutor 向 ResourceManager 返回 `slotReport`。

![](https://vendanner.github.io/img/Flink/TaskManager2ResourceManager.jpg)

#### 超时

TaskManger 和 ResourceManager 是通过心跳感知，若一方挂了如何检测呢？

```java
// org.apache.flink.runtime.heartbeat.HeartbeatMonitorImpl
public void reportHeartbeat() {
  // 收到心跳，重置超时线程
  lastHeartbeat = System.currentTimeMillis();
  resetHeartbeatTimeout(heartbeatTimeoutIntervalMs);
}
// 周期性检查 timeout
void resetHeartbeatTimeout(long heartbeatTimeout) {
  if (state.get() == State.RUNNING) {
    cancelTimeout();
    futureTimeout = scheduledExecutor.schedule(this, heartbeatTimeout, TimeUnit.MILLISECONDS);
    // Double check for concurrent accesses (e.g. a firing of the scheduled future)
    if (state.get() != State.RUNNING) {
      cancelTimeout();
    }
  }
}

public void run() {
  // The heartbeat has timed out if we're in state running
  // 开启检查心跳定时器线程，收到心跳重置定时器
  // 执行到本函数说明心跳超时了，通知接口
  if (state.compareAndSet(State.RUNNING, State.TIMEOUT)) {
    // ResourceManagerHeartbeatListener.notifyHeartbeatTimeout
    // TaskManager 准备重新向 ResourceManager 注册
    heartbeatListener.notifyHeartbeatTimeout(resourceID);
  }
}
```

### TaskSlotTable

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

### jobLeaderService

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

### FileCache

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