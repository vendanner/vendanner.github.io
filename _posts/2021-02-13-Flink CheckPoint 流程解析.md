---
layout:     post
title:      Flink CheckPoint 流程解析
subtitle:   
date:       2021-02-13
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - CheckPoint
    - 源码
---

### CheckPoint 准备

```java
// org.apache.flink.runtime.checkpoint.CheckpointCoordinator
private ScheduledFuture<?> scheduleTriggerWithDelay(long initDelay) {
  // 周期性触发
  return timer.scheduleAtFixedRate(
          new ScheduledTrigger(), initDelay, baseInterval, TimeUnit.MILLISECONDS);
}
private final class ScheduledTrigger implements Runnable {
  public void run() {
      try {
          // 触发 CheckPoint
          triggerCheckpoint(true);
      } catch (Exception e) {
          LOG.error("Exception while triggering checkpoint for job {}.", job, e);
      }
  }
}
public CompletableFuture<CompletedCheckpoint> triggerCheckpoint(
      CheckpointProperties props,
      @Nullable String externalSavepointLocation,
      boolean isPeriodic,
      boolean advanceToEndOfTime) {
  ...
  // externalSavepointLocation savepoint 触发也是这个入口
  CheckpointTriggerRequest request =
          new CheckpointTriggerRequest(
                  props, externalSavepointLocation, isPeriodic, advanceToEndOfTime);
  // 先判断当前 checkpointRequest 是否允许被发送
  // 满足再 startTriggeringCheckpoint
  chooseRequestToExecute(request).ifPresent(this::startTriggeringCheckpoint);
  return request.onCompletionPromise;
}
```

#### CheckPoint 间隔

两个 checkpoint 间隔要满足配置

```java
// org.apache.flink.runtime.checkpoint.CheckpointCoordinator
// -> chooseRequestToExecute
// org.apache.flink.runtime.checkpoint.CheckpointRequestDecider
// -> chooseRequestToExecute
private Optional<CheckpointTriggerRequest> chooseRequestToExecute(
      boolean isTriggering, long lastCompletionMs) {
  ...
  // queuedRequests 存放要进行 checkpoint 的请求
  CheckpointTriggerRequest first = queuedRequests.first();
  if (!first.isForce() && first.isPeriodic) {
      // 是否满足 checkpoint 间隔，异常输出
      long nextTriggerDelayMillis = nextTriggerDelayMillis(lastCompletionMs);
      if (nextTriggerDelayMillis > 0) {
          queuedRequests
                  .pollFirst()
                  .completeExceptionally(
                          new CheckpointException(MINIMUM_TIME_BETWEEN_CHECKPOINTS));
          rescheduleTrigger.accept(nextTriggerDelayMillis);
          return Optional.empty();
      }
  }
  // 返回第一个 checkpoint
  return Optional.of(queuedRequests.pollFirst());
}
// 最后一个 (checkpoint 时间 + 间隔时间) - 当前时间
private long nextTriggerDelayMillis(long lastCheckpointCompletionRelativeTime) {
  return lastCheckpointCompletionRelativeTime
          - clock.relativeTimeMillis()
          + minPauseBetweenCheckpoints;
}
```

#### Task 状态检查

所有 Task 状态都必须是 RUNNING

```java
// org.apache.flink.runtime.checkpoint.CheckpointCoordinator
private void startTriggeringCheckpoint(CheckpointTriggerRequest request) {
  try {
      // 检查 task 状态都是运行，否则报错
      final Execution[] executions = getTriggerExecutions();
      // ack task 状态检查，否则报错
      final Map<ExecutionAttemptID, ExecutionVertex> ackTasks = getAckTasks();
      // 开始触发 checkpoint，isTriggering 置位，checkpoint 结束后失效
      isTriggering = true;

      final long timestamp = System.currentTimeMillis();
      final CompletableFuture<PendingCheckpoint> pendingCheckpointCompletableFuture =
              // checkpointid 自增
              initializeCheckpoint(request.props, request.externalSavepointLocation)
                      .thenApplyAsync(
                              (checkpointIdAndStorageLocation) ->
                                      // 创建检查点：PendingCheckpoint
                                      // 并设置超时定时器，checkpointTimeOut
                                      createPendingCheckpoint(
                                              timestamp,
                                              request.props,
                                              ackTasks,
                                              request.isPeriodic,
                                              checkpointIdAndStorageLocation.checkpointId,
                                              checkpointIdAndStorageLocation
                                                      .checkpointStorageLocation,
                                              request.getOnCompletionFuture()),
                              timer);

      final CompletableFuture<?> coordinatorCheckpointsComplete =
              pendingCheckpointCompletableFuture.thenComposeAsync(
                      (pendingCheckpoint) ->
                              OperatorCoordinatorCheckpoints
                                      .triggerAndAcknowledgeAllCoordinatorCheckpointsWithCompletion(
                                              coordinatorsToCheckpoint,
                                              pendingCheckpoint,
                                              timer),
                      timer);
      final CompletableFuture<?> masterStatesComplete =
              coordinatorCheckpointsComplete.thenComposeAsync(
                      ignored -> {
                          PendingCheckpoint checkpoint =
                                  FutureUtils.getWithoutException(
                                          pendingCheckpointCompletableFuture);
                          return snapshotMasterState(checkpoint);
                      },
                      timer);

      FutureUtils.assertNoException(
              CompletableFuture.allOf(masterStatesComplete, coordinatorCheckpointsComplete)
                      .handleAsync(
                              (ignored, throwable) -> {
                                  final PendingCheckpoint checkpoint =
                                          FutureUtils.getWithoutException(
                                                  pendingCheckpointCompletableFuture);

                                  Preconditions.checkState(
                                          checkpoint != null || throwable != null,
                                          "Either the pending checkpoint needs to be created or an error must have been occurred.");

                                  if (throwable != null) {
                                      // the initialization might not be finished yet
                                      if (checkpoint == null) {
                                          onTriggerFailure(request, throwable);
                                      } else {
                                          onTriggerFailure(checkpoint, throwable);
                                      }
                                  } else {
                                      if (checkpoint.isDisposed()) {
                                          onTriggerFailure(
                                                  checkpoint,
                                                  new CheckpointException(
                                                          CheckpointFailureReason
                                                                  .TRIGGER_CHECKPOINT_FAILURE,
                                                          checkpoint.getFailureCause()));
                                      } else {
                                          // no exception, no discarding, everything is OK
                                          final long checkpointId =
                                                  checkpoint.getCheckpointId();
                                         // 开始出发算子的 checkpoint 
                                          snapshotTaskState(
                                                  timestamp,
                                                  checkpointId,
                                                  checkpoint.getCheckpointStorageLocation(),
                                                  request.props,
                                                  executions,
                                                  request.advanceToEndOfTime);

                                          coordinatorsToCheckpoint.forEach(
                                                  (ctx) ->
                                                          ctx.afterSourceBarrierInjection(
                                                                  checkpointId));
                                          // It is possible that the tasks has finished
                                          // checkpointing at this point.
                                          // So we need to complete this pending checkpoint.
                                          if (!maybeCompleteCheckpoint(checkpoint)) {
                                              return null;
                                          }
                                          onTriggerSuccess();
                                      }
                                  }
                                  return null;
                              },
                              timer)
                      .exceptionally(
                           ......
}

// task 必须都是 runnning，否则 checkpoint 停止
private Execution[] getTriggerExecutions() throws CheckpointException {
  Execution[] executions = new Execution[tasksToTrigger.length];
  for (int i = 0; i < tasksToTrigger.length; i++) {
      Execution ee = tasksToTrigger[i].getCurrentExecutionAttempt();
      if (ee == null) {
          LOG.info(
                  "Checkpoint triggering task {} of job {} is not being executed at the moment. Aborting checkpoint.",
                  tasksToTrigger[i].getTaskNameWithSubtaskIndex(),
                  job);
          throw new CheckpointException(
                  CheckpointFailureReason.NOT_ALL_REQUIRED_TASKS_RUNNING);
      } else if (ee.getState() == ExecutionState.RUNNING) {
          // 检查状态是 running
          executions[i] = ee;
      } else {
          LOG.info(
                  "Checkpoint triggering task {} of job {} is not in state {} but {} instead. Aborting checkpoint.",
                  tasksToTrigger[i].getTaskNameWithSubtaskIndex(),
                  job,
                  ExecutionState.RUNNING,
                  ee.getState());
          throw new CheckpointException(
                  CheckpointFailureReason.NOT_ALL_REQUIRED_TASKS_RUNNING);
      }
  }
  return executions;
}
```

#### ack task 检查

```java
// 所有 ack task 必须 running ，否则 checkpoint 停止
private Map<ExecutionAttemptID, ExecutionVertex> getAckTasks() throws CheckpointException {
  Map<ExecutionAttemptID, ExecutionVertex> ackTasks = new HashMap<>(tasksToWaitFor.length);
  for (ExecutionVertex ev : tasksToWaitFor) {
      // 应答点 task running
      Execution ee = ev.getCurrentExecutionAttempt();
      if (ee != null) {
          ackTasks.put(ee.getAttemptId(), ev);
      } else {
          LOG.info(
                  "Checkpoint acknowledging task {} of job {} is not being executed at the moment. Aborting checkpoint.",
                  ev.getTaskNameWithSubtaskIndex(),
                  job);
          throw new CheckpointException(
                  CheckpointFailureReason.NOT_ALL_REQUIRED_TASKS_RUNNING);
      }
  }
  return ackTasks;
}
```

#### PendingCheckpoint

checkpointid 自增得到新的 checkpoint id，并创建新的 PendingCheckpoint：启动的检查点，但还没有被确认，等所有任务都确认好本次检查点后，就会转化为 Completed-checkpoint。

```java
// org.apache.flink.runtime.checkpoint.CheckpointCoordinator#initializeCheckpoint
// 在高可用模式下 Flink 会基于 Zookeeper 的分布式计数器来生成检查点编号
long checkpointID = checkpointIdCounter.getAndIncrement();
// org.apache.flink.runtime.checkpoint.CheckpointCoordinator#createPendingCheckpoint
private PendingCheckpoint createPendingCheckpoint {
  ...
  // 新建检查点
  final PendingCheckpoint checkpoint =
          new PendingCheckpoint(
                  job,
                  checkpointID,
                  timestamp,
                  ackTasks,
                  OperatorInfo.getIds(coordinatorsToCheckpoint),
                  masterHooks.keySet(),
                  props,
                  checkpointStorageLocation,
                  onCompletionPromise);

  if (statsTracker != null) {
      PendingCheckpointStats callback =
              statsTracker.reportPendingCheckpoint(checkpointID, timestamp, props);

      checkpoint.setStatsCallback(callback);
  }
  // checkpoint timeout 处理：CheckpointCanceller，停止 checkpoint 并取消
  synchronized (lock) {
      pendingCheckpoints.put(checkpointID, checkpoint);

      ScheduledFuture<?> cancellerHandle =
              timer.schedule(
                      new CheckpointCanceller(checkpoint),
                      checkpointTimeout,
                      TimeUnit.MILLISECONDS);

      if (!checkpoint.setCancellerHandle(cancellerHandle)) {
          // checkpoint is already disposed!
          cancellerHandle.cancel(false);
      }
  }
  ...
  return checkpoint;
}
```

#### 触发

JobMaster 开始向 Source 节点发送 触发 checkpoint 命令：triggerSynchronousSavepoint/triggerCheckpoint

```java
// org.apache.flink.runtime.checkpoint.CheckpointCoordinator#snapshotTaskState
private void snapshotTaskState {
  final CheckpointOptions checkpointOptions =
          CheckpointOptions.forConfig(
                  props.getCheckpointType(),
                  checkpointStorageLocation.getLocationReference(),
                  isExactlyOnceMode,
                  unalignedCheckpointsEnabled,
                  alignmentTimeout);
  for (Execution execution : executions) {
      if (props.isSynchronous()) {
          // savepoint 同步
          execution.triggerSynchronousSavepoint(
                  checkpointID, timestamp, checkpointOptions, advanceToEndOfTime);
      } else {
          // checkpoint 异步
          execution.triggerCheckpoint(checkpointID, timestamp, checkpointOptions);
      }
  }
}
// org.apache.flink.runtime.executiongraph.Execution#triggerCheckpointHelper
private void triggerCheckpointHelper{
  final CheckpointType checkpointType = checkpointOptions.getCheckpointType();
  ...
  final LogicalSlot slot = assignedResource;

  if (slot != null) {
      final TaskManagerGateway taskManagerGateway = slot.getTaskManagerGateway();
      taskManagerGateway.triggerCheckpoint(
              attemptId,
              getVertex().getJobId(),
              checkpointId,
              timestamp,
              checkpointOptions,
              advanceToEndOfEventTime);
  } 
  ...
```

`Execution` 与 task 是一一对应的，通过成员变量 `assignedResource` (slot) 获取当前 `Execution` 所在 TaskManager，然后通过 `taskManagerGateway` 向对应的 TaskManager 发送触发事件。

- 先检查 checkpoint 间隔
- 确保 task 都是活着的
- 自增 checkpoint 并新建 pendingCheckpoint
- pendingCheckpoint 注册 timeout 回调，超时停止 checkpoint
- 携带 executionAttemptID 给 TaskExecutor 触发 Checkpoint

### Checkpoint

#### TaskExecutor

TaskManagerGateway.triggerCheckpoint 命令时，底层时通过 RPC 调用 TaskExecutor.triggerCheckpoint 函数，每个算子 checkpoint 都是通过这种方式。

```java
// org.apache.flink.runtime.taskexecutor.TaskExecutor#triggerCheckpoint
public CompletableFuture<Acknowledge> triggerCheckpoint{
  ...
  final CheckpointType checkpointType = checkpointOptions.getCheckpointType();
  ...
  final Task task = taskSlotTable.getTask(executionAttemptID);
  // task 接收到 checkpoint 事件，后续正式执行 checkpoint
  if (task != null) {
      task.triggerCheckpointBarrier(
              checkpointId, checkpointTimestamp, checkpointOptions, advanceToEndOfEventTime);
      return CompletableFuture.completedFuture(Acknowledge.get());
  } 
  ...
}
```

#### Task

`task.triggerCheckpointBarrier` 触发 Task 层的 checkpoint。

```java
// org.apache.flink.runtime.taskmanager.Task#triggerCheckpointBarrier
public void triggerCheckpointBarrier(
      final long checkpointID,
      final long checkpointTimestamp,
      final CheckpointOptions checkpointOptions,
      final boolean advanceToEndOfEventTime) {
  // invokable = StreamTask
  final AbstractInvokable invokable = this.invokable;
  final CheckpointMetaData checkpointMetaData =
          new CheckpointMetaData(checkpointID, checkpointTimestamp);

  if (executionState == ExecutionState.RUNNING && invokable != null) {
      try {
          // StreamTask 层触发 Checkpoint
          invokable.triggerCheckpointAsync(
                  checkpointMetaData, checkpointOptions, advanceToEndOfEventTime);
```

#### StreamTask

Task 将 Checkpoint 委托到了 StreamTask。StreamTask 是算子 chain 后形成的 task，里面包含一个或多个算子。此时分为两种情况：

- SourceStreamTask，**产生 CheckpointBarrier 向下游广播**

```java
// org.apache.flink.streaming.runtime.tasks.StreamTask#triggerCheckpointAsync
triggerCheckpointAsync -> 
  -> triggerCheckpoint     // source 节点触发 checkpoint
    -> performCheckpoint
```

- 普通 StreamTask 根据 CheckpointBarrier 触发检查点

```java
// org.apache.flink.streaming.runtime.tasks.StreamTask#triggerCheckpointOnBarrier
public void triggerCheckpointOnBarrier {
  try {
      if (performCheckpoint(
              checkpointMetaData, checkpointOptions, checkpointMetrics, false)) {
```

最终两者都是调用 `performCheckpoint`。

```java
// org.apache.flink.streaming.runtime.tasks.StreamTask#performCheckpoint -> 
performCheckpoint {
   if (isRunning) {
     checkpointState
   } else {
     // 非 running，向下游发送 CancelCheckpointMarker 取消本次 checkpoint
   }
} 
// org.apache.flink.streaming.runtime.tasks.SubtaskCheckpointCoordinatorImpl#checkpointState
public void checkpointState
      throws Exception {

  // Step (0): Record the last triggered checkpointId and abort the sync phase of checkpoint
  // if necessary.
  lastCheckpointId = metadata.getCheckpointId();
  if (checkAndClearAbortedStatus(metadata.getCheckpointId())) {
      // broadcast cancel checkpoint marker to avoid downstream back-pressure due to
      // checkpoint barrier align.
      operatorChain.broadcastEvent(new CancelCheckpointMarker(metadata.getCheckpointId()));
      LOG.info(
              "Checkpoint {} has been notified as aborted, would not trigger any checkpoint.",
              metadata.getCheckpointId());
      return;
  }
  // 步骤一：
  // 为开始 checkpoint 做准备工作，回调算子的 prepareSnapshotPreBarrier
  // 该方法执行事件要尽可能短
  operatorChain.prepareSnapshotPreBarrier(metadata.getCheckpointId());
  // 步骤二：向下游发送 checkpoint barrier
  operatorChain.broadcastEvent(
          new CheckpointBarrier(metadata.getCheckpointId(), metadata.getTimestamp(), options),
          options.isUnalignedCheckpoint());
  // 步骤三： checkpiont 非对齐模式，input/output buffer 准备保存
  if (options.isUnalignedCheckpoint()) {
      // output data already written while broadcasting event
      channelStateWriter.finishOutput(metadata.getCheckpointId());
  }
  // 步骤四: 开始 checkpioint
  Map<OperatorID, OperatorSnapshotFutures> snapshotFutures =
          new HashMap<>(operatorChain.getNumberOfOperators());
  try {
      // 开始 checkpoint，这是同步操作，执行
      if (takeSnapshotSync(
              snapshotFutures, metadata, metrics, options, operatorChain, isCanceled)) {
          // task checkpoint 结束，异步给 JobMaster 发送通知 
          finishAndReportAsync(snapshotFutures, metadata, metrics, options);
      } 
  ...
}
```

- prepareSnapshotPreBarrier：算子回调，该函数体执行时长要短
- 向下游发送 checkpoint barrier
- checkpiont 非对齐模式，input/output buffer 准备保存
- checkpioint
  - checkpoint 同步操作
  - checkpoint 结束，异步给 JobMaster 发送通知

##### checkpoint 同步

```java
// org.apache.flink.streaming.runtime.tasks.SubtaskCheckpointCoordinatorImpl#takeSnapshotSync
-> takeSnapshotSync
  -> buildOperatorSnapshotFutures
    -> checkpointStreamOperator
// org.apache.flink.streaming.runtime.tasks.SubtaskCheckpointCoordinatorImpl#checkpointStreamOperator
private static OperatorSnapshotFutures checkpointStreamOperator{
  try {
      // 回调算子 snapshotState
      return op.snapshotState(
              checkpointMetaData.getCheckpointId(),
              checkpointMetaData.getTimestamp(),
              checkpointOptions,
              storageLocation);
  } 
  ...
}
```

回调算子 snapshotState 函数

## 参考资料

[Checkpoint barrier 对齐源码解析](https://mp.weixin.qq.com/s/5qJ7ass3YdajXup2uQEN5A)

[Checkpoint对齐机制源码分析](https://mp.weixin.qq.com/s/oYn0mlxSaC1OFpgG8zcJpg)

[Flink rocksdb如何做checkpoint](https://cloud.tencent.com/developer/article/1560091)

[Flink源码阅读（四）--- checkpoint制作](https://www.jianshu.com/p/539dbda544b0)