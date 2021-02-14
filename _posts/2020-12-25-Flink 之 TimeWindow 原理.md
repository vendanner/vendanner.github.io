---
layout:     post
title:      Flink 之 TimeWindow 原理
subtitle:   
date:       2020-12-25
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - Time
    - Window
---

Flink1.12

### 介绍

```java
// Keyed Windows
stream
       .keyBy(...)               <-  keyed versus non-keyed windows
       .window(...)              <-  required: "assigner"
      [.trigger(...)]            <-  optional: "trigger" (else default trigger)
      [.evictor(...)]            <-  optional: "evictor" (else no evictor)
      [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
      [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
       .reduce/aggregate/fold/apply()      <-  required: "function"
      [.getSideOutput(...)]      <-  optional: "output tag"
```

demo

```java
dataStream.flatMap(new FlatMapFunction<String, Tuple2<String, Long>>() {
    @Override
    public void flatMap(String s, Collector<Tuple2<String, Long>> collector) throws Exception {
        String[] fields = s.split(",");
        collector.collect(Tuple2.of(fields[0], Long.valueOf(fields[1])));
    }
    }).assignTimestampsAndWatermarks(
                    WatermarkStrategy
                .forGenerator((ctx) -> new PeriodicWatermarkGenerator())     // watermark
                .withTimestampAssigner((ctx) -> new TimeStampExtractor()))   // 指定时间字段
    .keyBy(tuple -> tuple.f0) //指定分组的字段
    // event time 滑动窗口
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .process(new SumProcessFunction()) // udf 计算结果
    .print();
```

demo 展示了基于事件时间的**滑动窗口**使用：**每隔5秒**计算**最近10秒**的单词次数。在分析窗口机制之前，先对 Flink 窗口有基本的认识。Flink窗口包含三个部件：**分配器**（assigner）、**触发器**（trigger）以及**驱逐器**（evictor）：

- 分配器用于为每个元素分配窗口（每个元素可以分配到一个或多个窗口）
- 触发器用于定义什么时候对窗口执行操作（比如对窗口进行计算、清理等，这取决于触发器触发后得到的触发结果）
- 驱逐器用于指定哪些元素需要从窗口中移除，其工作时机介于窗口函数之前或之后。

#### Assigner

分配器给元素分配一个或者多个窗口，实现的 `assignWindows` 方法为带有时间戳 timestamp 的 element 分配一个或多个窗口，并返回窗口集合。不同类型的窗口函数主要差别在于分配器的实现方式不同。

#### Trigger

窗口触发器定义了窗口何时被触发同时决定触发行为（比如：对窗口进行清理或者计算）。Trigger 提供了四个非常重要的方法，供具体的触发器根据自己的语义实现：

* onElement：每个元素触发的回调方法；
* onProcessingTime：基于处理时间触发的回调方法；
* onEventTime：基于事件时间触发的回调方法；
* onMerge：窗口在合并时触发的回调方法；

触发器方法返回的触发结果（TriggerResult）是一个枚举类型，它用于决定窗口在触发后的行为，枚举值如下：

* CONTINUE：不作任何处理；
* FIRE\_AND\_PURGE：触发窗口计算并输出结果同时清理并释放窗口（该值只会被清理触发器PurgingTrigger使用）；
* FIRE：触发窗口计算并输出结果，但窗口并没有被释放并且数据仍然保留；
* PURGE：不触发窗口计算，不输出结果，只清除窗口中的所有数据并释放窗口；

若用户无指定，算子提供默认的触发器。

#### Evictor

Evictor 用来从窗口中移除元素，可在窗口函数之前/之后触发，这对应着两个接口方法：evictBefore 和 evictAfter。

无默认值，一般不指定(除非有特殊实现)。

### 源码

在了解窗口函数的使用后，有必要深入理解底层的实现机制。

```java
// window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
// org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows
public static SlidingEventTimeWindows of(Time size, Time slide) {
    return new SlidingEventTimeWindows(size.toMilliseconds(), slide.toMilliseconds(), 0);
}
protected SlidingEventTimeWindows(long size, long slide, long offset) {
    ...
    this.size = size;
    this.slide = slide;
    // 默认是从 1970-01-01 00:00:00 为基准点，划分窗口
    // 此处可以设置偏移点，即不以 00 开始划分
    this.offset = offset;
}
```

#### SlidingEventTimeWindows

```java
// org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows
// 窗口分配器，输入 element，返回该元素需要分配的窗口集合
public Collection<TimeWindow> assignWindows(
        Object element, long timestamp, WindowAssignerContext context) {
    if (timestamp > Long.MIN_VALUE) {
        List<TimeWindow> windows = new ArrayList<>((int) (size / slide));
        // getWindowStartWithOffset，最后一个窗口开始的时间
        // timestamp 是 element 的时间戳
        long lastStart = TimeWindow.getWindowStartWithOffset(timestamp, offset, slide);
        for (long start = lastStart; start > timestamp - size; start -= slide) {
            // 滑动窗口中，一个元素可能分配到多个窗口中
            windows.add(new TimeWindow(start, start + size));
        }
        return windows;
    } 
    ...
}
// 默认触发器
public Trigger<Object, TimeWindow> getDefaultTrigger(StreamExecutionEnvironment env) {
    return EventTimeTrigger.create();
}
```

SlidingEventTimeWindows 默认指定 **EventTimeTrigger**。

#### EventTimeTrigger

```java
// org.apache.flink.streaming.api.windowing.triggers.EventTimeTrigger
public TriggerResult onElement(
        Object element, long timestamp, TimeWindow window, TriggerContext ctx)
        throws Exception {
    if (window.maxTimestamp() <= ctx.getCurrentWatermark()) {
        // 当水印大于窗口的结束时间，说明该窗口要进行计算了
        return TriggerResult.FIRE;
    } else {
        // 为窗口的结束时间注册定时器，到时间后进行窗口的计算
        ctx.registerEventTimeTimer(window.maxTimestamp());
        return TriggerResult.CONTINUE;
    }
}
// 事件时间定时器回调
public TriggerResult onEventTime(long time, TimeWindow window, TriggerContext ctx) {
    return time == window.maxTimestamp() ? TriggerResult.FIRE : TriggerResult.CONTINUE;
}
// 事件时间触发，处理时间不进行操作
public TriggerResult onProcessingTime(long time, TimeWindow window, TriggerContext ctx)
        throws Exception {
    return TriggerResult.CONTINUE;
}
// 窗口销毁时，删除之前注册的定时器
public void clear(TimeWindow window, TriggerContext ctx) throws Exception {
    ctx.deleteEventTimeTimer(window.maxTimestamp());
}
```

触发器依靠**定时器**来实现。

#### WindowOperator

定义好窗口后，那窗口数据如何存储，怎么计算呢？在 Flink 中，都是需要算子来实现的。这一小节来看看定义好的窗口是如何转化为一个算子。

```java
// .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
// org.apache.flink.streaming.api.datastream.KeyedStream
public <W extends Window> WindowedStream<T, KEY, W> window(
        WindowAssigner<? super T, W> assigner) {
    return new WindowedStream<>(this, assigner);
}
// org.apache.flink.streaming.api.datastream.WindowedStream
public WindowedStream(KeyedStream<T, K> input, WindowAssigner<? super T, W> windowAssigner) {
    this.input = input;
    // 构建 builder，可在 builder 上添加自己的触发器和驱逐器
    this.builder =
            new WindowOperatorBuilder<>(
                    windowAssigner,
                    windowAssigner.getDefaultTrigger(input.getExecutionEnvironment()),
                    input.getExecutionConfig(),
                    input.getType(),
                    input.getKeySelector(),
                    input.getKeyType());
}
```

窗口转化为算子

```java
// process( udf )
// org.apache.flink.streaming.api.datastream.WindowedStream
public <R> SingleOutputStreamOperator<R> process(ProcessWindowFunction<T, R, K, W> function) {
    TypeInformation<R> resultType =
            getProcessWindowFunctionReturnType(function, getInputType(), null);

    return process(function, resultType);
}
public <R> SingleOutputStreamOperator<R> process(
        ProcessWindowFunction<T, R, K, W> function, TypeInformation<R> resultType) {
    function = input.getExecutionEnvironment().clean(function);
    final String opName = builder.generateOperatorName(function, null);
    // 得到 WindowOperator
    OneInputStreamOperator<T, R> operator = builder.process(function);
    // DataStream 最终是收集 transform
    return input.transform(opName, resultType, operator);
}
// org.apache.flink.streaming.runtime.operators.windowing.WindowOperatorBuilder
public <R> WindowOperator<K, T, ?, R, W> process(ProcessWindowFunction<T, R, K, W> function) {
    Preconditions.checkNotNull(function, "ProcessWindowFunction cannot be null");
    return apply(new InternalIterableProcessWindowFunction<>(function));
}
private <R> WindowOperator<K, T, ?, R, W> apply(
        InternalWindowFunction<Iterable<T>, R, K, W> function) {
    if (evictor != null) {
        // 包含自定义的 evictor，暂不介绍
        return buildEvictingWindowOperator(function);
    } else {
        // 窗口中的数据如何保存？
        // ListState
        ListStateDescriptor<T> stateDesc =
                new ListStateDescriptor<>(
                        WINDOW_STATE_NAME, inputType.createSerializer(config));

        return buildWindowOperator(stateDesc, function);
    }
}
private <ACC, R> WindowOperator<K, T, ACC, R, W> buildWindowOperator(
        StateDescriptor<? extends AppendingState<T, ACC>, ?> stateDesc,
        InternalWindowFunction<ACC, R, K, W> function) {

    return new WindowOperator<>(
            windowAssigner,
            windowAssigner.getWindowSerializer(config),
            keySelector,
            keyType.createSerializer(config),
            stateDesc,
            function,
            trigger,
            allowedLateness,
            lateDataOutputTag);
}
```

#### 窗口执行流程

```java
// org.apache.flink.streaming.runtime.operators.windowing.WindowOperator
public void open() throws Exception {
    super.open();
    // 定时器服务
    internalTimerService = getInternalTimerService("window-timers", windowSerializer, this);
    // 触发器上下文
    triggerContext = new Context(null, null);
    processContext = new WindowContext(null);
    // 窗口分配器上下文
    windowAssignerContext =
            new WindowAssigner.WindowAssignerContext() {
                @Override
                public long getCurrentProcessingTime() {
                    return internalTimerService.currentProcessingTime();
                }
            };

    // state，每个窗口各自有自己的 namaspace
    if (windowStateDescriptor != null) {
        windowState =
                (InternalAppendingState<K, W, IN, ACC, ACC>)
                        getOrCreateKeyedState(windowSerializer, windowStateDescriptor);
    }
  ...
// 每条数据在窗口下的处理    
public void processElement(StreamRecord<IN> element) throws Exception {
    // windowAssigner 为 element 分配窗口
    final Collection<W> elementWindows =
            windowAssigner.assignWindows(
                    element.getValue(), element.getTimestamp(), windowAssignerContext);

    final K key = this.<K>getKeyedStateBackend().getCurrentKey();

    if (windowAssigner instanceof MergingWindowAssigner) {
        // 合并窗口(session) 情况暂不考虑
    } else {
        for (W window : elementWindows) {
            // 过期数据直接丢弃
            if (isWindowLate(window)) {
                continue;
            }
            isSkippedElement = false;
            // 保存当前 element 到 state
            windowState.setCurrentNamespace(window);
            // 若是增量计算，add 函数会直接调用 state reduceFunction
            // 此时相当于是来一个 element 计算一次，但不会向后发送
            windowState.add(element.getValue());
            // 调用触发器，默认 EventTimeTrigger
            triggerContext.key = key;
            triggerContext.window = window;
            TriggerResult triggerResult = triggerContext.onElement(element);
            // 触发返回计算，则调用 udf执行
            if (triggerResult.isFire()) {
                // 取出当前 windows 中 state 开始计算
                ACC contents = windowState.get();
                if (contents == null) {
                    continue;
                }
                // 执行 udf
                emitWindowContents(window, contents);
            }
            // 触发器返回清除数据，则 state 删除
            if (triggerResult.isPurge()) {
                windowState.clear();
            }
            registerCleanupTimer(window);
        }
    }

    // 过期数据侧流输出或记录
    if (isSkippedElement && isElementLate(element)) {
        if (lateDataOutputTag != null) {
            sideOutput(element);
        } else {
            this.numLateRecordsDropped.inc();
        }
    }
}
// 定时器服务回调 
public void onEventTime(InternalTimer<K, W> timer) throws Exception {
    triggerContext.key = timer.getKey();
    triggerContext.window = timer.getNamespace();

    MergingWindowSet<W> mergingWindows;

    if (windowAssigner instanceof MergingWindowAssigner) {
       // 暂不考虑合并的情况
    } else {
        windowState.setCurrentNamespace(triggerContext.window);
        mergingWindows = null;
    }
    // 调用触发器的 onTimer
    TriggerResult triggerResult = triggerContext.onEventTime(timer.getTimestamp());

    if (triggerResult.isFire()) {
        ACC contents = windowState.get();
        if (contents != null) {
            emitWindowContents(triggerContext.window, contents);
        }
    }
    // 当前窗口到期，销毁当前窗口所有资源
    if (windowAssigner.isEventTime()
            && isCleanupTime(triggerContext.window, timer.getTimestamp())) {
        clearAllState(triggerContext.window, windowState, mergingWindows);
    }
    if (triggerResult.isPurge()) {
        windowState.clear();
    }
}
```

![](https://vendanner.github.io/img/Flink/WindowOperator.jpg)

### 扩展

#### ContinuousEventTimeTrigger

现在我们应该算是对窗口有一定的了解。考虑这样一个需求：

> 每五分钟输出当天 PV

这个需求的特殊之处在于每5分钟就要触发而不是在窗口结束的时候触发，显然我们需要修改默认的触发器来支持。触发器中间隔5分钟注册一个定时器，定时器回调函数返回 `Fire` 来执行 UDF 。

```java
// org.apache.flink.streaming.api.windowing.triggers.ContinuousEventTimeTrigger
public TriggerResult onElement(Object element, long timestamp, W window, TriggerContext ctx)
        throws Exception {

    if (window.maxTimestamp() <= ctx.getCurrentWatermark()) {
        return TriggerResult.FIRE;
    } else {
        ctx.registerEventTimeTimer(window.maxTimestamp());
    }
    // 第一次：注册一个五分钟后的定时器，并将时间存入 state
    ReducingState<Long> fireTimestamp = ctx.getPartitionedState(stateDesc);
    if (fireTimestamp.get() == null) {
        long start = timestamp - (timestamp % interval);
        long nextFireTimestamp = start + interval;
        ctx.registerEventTimeTimer(nextFireTimestamp);
        fireTimestamp.add(nextFireTimestamp);
    }
    return TriggerResult.CONTINUE;
}
public TriggerResult onEventTime(long time, W window, TriggerContext ctx) throws Exception {
    // 窗口时间到了，触发
    if (time == window.maxTimestamp()) {
        return TriggerResult.FIRE;
    }
    // 到5五分钟，再次注册定时器
    // 并返回 FIRE，执行 UDF
    ReducingState<Long> fireTimestampState = ctx.getPartitionedState(stateDesc);
    Long fireTimestamp = fireTimestampState.get();
    if (fireTimestamp != null && fireTimestamp == time) {
        fireTimestampState.clear();
        fireTimestampState.add(time + interval);
        ctx.registerEventTimeTimer(time + interval);
        return TriggerResult.FIRE;
    }

    return TriggerResult.CONTINUE;
}
```

#### EvictingWindowOperator

当为窗口指定 `evictor` 时，生产就不是 **WindowOperator** 而是 **EvictingWindowOperator**。**EvictingWindowOperator** 在执行 UDF 前后多了一步  `evictor`  回调，它将删除一些无效的元素。使用 `evictor` 必须包含所有元素，所有不能使用增量计算的函数。

```java
// org.apache.flink.streaming.runtime.operators.windowing.EvictingWindowOperator
private void emitWindowContents{
    evictorContext.evictBefore(recordsWithTimestamp, Iterables.size(recordsWithTimestamp));
    userFunction.process;
    evictorContext.evictAfter(recordsWithTimestamp, Iterables.size(recordsWithTimestamp));
}
```





## 参考资料

[Flink滑动窗口原理与细粒度滑动窗口的性能问题](https://www.jianshu.com/p/45b03390b258)

[Apache Flink 零基础入门（六）：Flink Time & Window 解析](https://ververica.cn/developers/time-window/)