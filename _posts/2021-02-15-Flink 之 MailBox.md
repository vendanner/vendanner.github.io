---
layout:     post
title:      Flink 之 MailBox
subtitle:   
date:       2021-02-15
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - MailBox
---

在Flip-27 提出了类似于基于`MailBox` 单线程的执行模型，取代现有的多线程模型(1.10之前的版本)。所有的并发操作都通过队列(mailbox)进行排队，单线程依次处理，这样避免了并发操作。

`StreamTask` 是上面提及的单线程执行体。线程中会调用 MailboxProcessor 一直处理 mail(其他要执行task、event)。

```java
// org.apache.flink.streaming.runtime.tasks.mailbox.MailboxProcessor#runMailboxLoop
public void runMailboxLoop() throws Exception {
    suspended = !mailboxLoopRunning;

    final TaskMailbox localMailbox = mailbox;

    checkState(
            localMailbox.isMailboxThread(),
            "Method must be executed by declared mailbox thread!");

    assert localMailbox.getState() == TaskMailbox.State.OPEN : "Mailbox must be opened!";

    final MailboxController defaultActionContext = new MailboxController(this);
		
    while (isNextLoopPossible()) { // !suspended
        // The blocking `processMail` call will not return until default action is available.
        processMail(localMailbox, false); //检查Mailbox是否有mail，并处理
        if (isNextLoopPossible()) {
            // event 处理 -> StreamTask.processInput
            mailboxDefaultAction.runDefaultAction(
                    defaultActionContext); // lock is acquired inside default action as needed
        }
    }
}
```

整体流程变得非常简单，主线程不断获取mail处理即可。在checkpoint章节，会具体分析如何将消息传入 mailbox。

既然是单线程，那么 event的处理也都是在此线程中处理。以event的流转来分析整个处理流程。

![](https://vendanner.github.io/img/Flink/generateDAG.jpg)

在进入分析之前，先熟悉下几个术语

- InputGate：输入网关，与Edge一一对应，包含一/多个 InputChannel
  - InputChannel：对应**当前subTask 并行度**，有两种实现
    - LocalInputChannel：上下游在同个TaskManager
    - RemoteInputChannel：上下游不在同个TaskManager，需要跨网络
- ResultPartition：结果分区，与InputGate 一一对应，包含一/多个ResultSubPartition
  - ResultSubPartition：结果子分区，对应**下游subTask 并行度**，有两种实现
    - PipelinedSubpartition：流模式，纯内存，一次性，默认100ms/满足大小后往下游发送
    - BoundedBlockingSubpartition：批处理，内存/文件，需要等待上游所有的数据处理完毕

event 输入、处理、输出，这三个流程分析

### 输入

```java
// org.apache.flink.streaming.runtime.tasks.StreamTask#processInput
protected void processInput(MailboxDefaultAction.Controller controller) throws Exception {
    // 处理 event
  	InputStatus status = inputProcessor.processInput();
    if (status == InputStatus.MORE_AVAILABLE && recordWriter.isAvailable()) {
        // MORE_AVAILABLE（表示还有可用的数据等待处理）并且 recordWriter 可用（之前的异步操作已经处理完成）
        return;
    }
    if (status == InputStatus.END_OF_INPUT) {
        // END_OF_INPUT，它表示数据处理完成，这里就会告诉 MailBox 数据已经处理完成了
        controller.allActionsCompleted();
        return;
    }
    
    TaskIOMetricGroup ioMetrics = getEnvironment().getMetricGroup().getIOMetricGroup();
    TimerGauge timer;
    CompletableFuture<?> resumeFuture;
    if (!recordWriter.isAvailable()) {
        // 反压时间，recordWriter 无法继续写入，写出buffer 满了
        timer = ioMetrics.getBackPressuredTimePerSecond();
        resumeFuture = recordWriter.getAvailableFuture();
    } else {
        // 等待时间，等到有数据进 inputGate(buffer)
        timer = ioMetrics.getIdleTimeMsPerSecond();
        resumeFuture = inputProcessor.getAvailableFuture();
    }
    // 阻塞，直到有可用的数据到/recordWriter 可用
    // resumeFuture完成后，异步操作
    assertNoException(
            resumeFuture.thenRun(
                    new ResumeWrapper(controller.suspendDefaultAction(timer), timer)));
}
```

![](https://vendanner.github.io/img/Flink/StreamInputProcessor.png)

#### StreamInputProcessor

基本实现类 `StreamOneInputProcessor`/`StreamTwoInputProcessor`

```java
// org.apache.flink.streaming.runtime.io.StreamOneInputProcessor#processInput
public InputStatus processInput() throws Exception {
    InputStatus status = input.emitNext(output); // StreamTaskInput 处理

    if (status == InputStatus.END_OF_INPUT) {
        endOfInputAware.endInput(input.getInputIndex() + 1);
    } else if (status == InputStatus.END_OF_RECOVERY) {
        if (input instanceof RecoverableStreamTaskInput) {
            input = ((RecoverableStreamTaskInput<IN>) input).finishRecovery();
        }
        return InputStatus.MORE_AVAILABLE;
    }

    return status;
}
```

看上图，StreamInputProcessor 只是做了封装，逻辑在StreamTaskInput和StreamTaskNetworkOutput中。

#### InputGate

```java
inputProcessor.getAvailableFuture()
-> org.apache.flink.streaming.runtime.io.StreamOneInputProcessor#getAvailableFuture
  -> org.apache.flink.streaming.runtime.io.AbstractStreamTaskNetworkInput#getAvailableFuture
   -> org.apache.flink.streaming.runtime.io.checkpointing.CheckpointedInputGate#getAvailableFuture
    -> org.apache.flink.runtime.io.network.partition.consumer.InputGate#getAvailableFuture
// org.apache.flink.runtime.io.AvailabilityProvider.AvailabilityHelper#getAvailableFuture
public CompletableFuture<?> getAvailableFuture() {
    return availableFuture;
}
```

获取输入event时，会一直**阻塞**在InputGate，直到 Future.complete。

那什么时候会完成呢？任意一个 InputChannel 有数据(在输出时会分析)

#### StreamTaskInput

数据输入的抽象，根据数据从哪里读取，分为两类：

- `StreamTaskNetworkInput`：从上游task获取数据(没有chain)，InputGate
- `StreamTaskSourceInput`：外部数据源获取数据，SourceFunction

```java
// org.apache.flink.streaming.runtime.io.AbstractStreamTaskNetworkInput#emitNext
public InputStatus emitNext(DataOutput<T> output) throws Exception {

    while (true) {
        // get the stream element from the deserializer
        if (currentRecordDeserializer != null) {
            RecordDeserializer.DeserializationResult result;
            try {
                // 反序列化，得到element，有好几种类型
                result = currentRecordDeserializer.getNextRecord(deserializationDelegate);
            } catch (IOException e) {
                throw new IOException(
                        String.format("Can't get next record for channel %s", lastChannel), e);
            }
            if (result.isBufferConsumed()) {
                currentRecordDeserializer = null;
            }

            if (result.isFullRecord()) {
                // 真正去处理数据
                processElement(deserializationDelegate.getInstance(), output);
                return InputStatus.MORE_AVAILABLE;
            }
        }
        // 继续获取下一个buffer，底层是从InputChannel获取
        Optional<BufferOrEvent> bufferOrEvent = checkpointedInputGate.pollNext();
        if (bufferOrEvent.isPresent()) {
            // return to the mailbox after receiving a checkpoint barrier to avoid processing of
            // data after the barrier before checkpoint is performed for unaligned checkpoint
            // mode
            if (bufferOrEvent.get().isBuffer()) {
                processBuffer(bufferOrEvent.get());
            } else {
                return processEvent(bufferOrEvent.get());
            }
        } else {
            if (checkpointedInputGate.isFinished()) {
                checkState(
                        checkpointedInputGate.getAvailableFuture().isDone(),
                        "Finished BarrierHandler should be available");
                return InputStatus.END_OF_INPUT;
            }
            return InputStatus.NOTHING_AVAILABLE;
        }
    }
}

// org.apache.flink.streaming.runtime.io.AbstractStreamTaskNetworkInput#processElement
private void processElement(StreamElement recordOrMark, DataOutput<T> output) throws Exception {
    // 处理四类数据
    if (recordOrMark.isRecord()) {
        // 数据，本文分析这里
        output.emitRecord(recordOrMark.asRecord());
    } else if (recordOrMark.isWatermark()) {
        // watermark
        statusWatermarkValve.inputWatermark(
                recordOrMark.asWatermark(), flattenedChannelIndices.get(lastChannel), output);
    } else if (recordOrMark.isLatencyMarker()) {
        // Latency
        output.emitLatencyMarker(recordOrMark.asLatencyMarker());
    } else if (recordOrMark.isStreamStatus()) {
        // StreamStatus
        statusWatermarkValve.inputStreamStatus(
                recordOrMark.asStreamStatus(),
                flattenedChannelIndices.get(lastChannel),
                output);
    } else {
        throw new UnsupportedOperationException("Unknown type of StreamElement");
    }
}
```

#### 序列化

从buffer 获取到的是二进制

#### StreamTaskNetworkOutput



### 处理

### 输出





## 参考资料

[重磅！Flink 将重构其核心线程模型](https://mp.weixin.qq.com/s/MrQZS7-dEuNr442lzw3xYg)

[Flink 基于 MailBox 实现的 StreamTask 线程模型](http://matt33.com/2020/03/20/flink-task-mailbox/)

