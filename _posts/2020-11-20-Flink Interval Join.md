---
layout:     post
title:      Flink Interval Join
subtitle:   
date:       2020-11-20
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - bigdata
    - join
---

Flink 1.11

在流处理中，每条流到来的时间是不确定的，那如何才能保证流能 `join` 呢？答案是通过**缓存流的数据**，这样就能保证每条数据都能参与 `join`。但缓存每条流的数据的代价是很大，所以在 `state` 中有 `TTL`(数据过了生命周期后删除)的概念。一些特殊的场景：统计下单后一小时付款的订单的，其实我们只需要保留1小时的数据即可(但这里不能简单的定义 TTL 为1小时，因为两条流的时间有关联)。 Flink 中为我们提供 `IntervalJoin`：流的每一条数据会与另一条流上的**不同时间区域**的数据进行 JOIN，**过期的数据**会删除。

### Demo

```java
// 统计下单后一小时付款的订单的
ordersStream
  .keyBy("orderId")
  .intervalJoin(paymentStream.keyBy("orderId"))
  // 时间间隔,设定下界和上界
  // 下界: 当前 EventTime 时刻，上界: 当前EventTime时刻一小时
  .between(Time.seconds(0),Time.hours(1))
  // 自定义ProcessJoinFunction 处理Join到的元素
  .process(new ProcessJoinFunction<Order, Payment, String>() {
      @Override
      public void processElement(Order left, Payment right, Context ctx, Collector<String> out) throws Exception {
          out.collect(left +" =Interval Join=> "+right);
      }
  })
  .print();
```

Sql 语法相比更简单

```sql
SELECT
*
FROM Orders AS o 
JOIN Payment AS p 
ON o.orderId = p.orderId 
AND p.payTime BETWEEN orderTime AND orderTime + INTERVAL '1' HOUR
```

### 原理

会调 API 后，下面看看底层原理。

``` scala
// org.apache.flink.streaming.api.scala.KeyedStream
def intervalJoin[OTHER](otherStream: KeyedStream[OTHER, K]): IntervalJoin[T, OTHER, K] = {
  new IntervalJoin[T, OTHER, K](this, otherStream)
}

class IntervalJoin[IN1, IN2, KEY](val streamOne: KeyedStream[IN1, KEY],
                                  val streamTwo: KeyedStream[IN2, KEY]) {

  /**
    * Specifies the time boundaries over which the join operation works, so that
    * <pre>leftElement.timestamp + lowerBound <= rightElement.timestamp
    * <= leftElement.timestamp + upperBound</pre>
    * 默认是包含上界和下界。 This can be configured
    * with [[IntervalJoined.lowerBoundExclusive]] and
    * [[IntervalJoined.upperBoundExclusive]]
    */
  @PublicEvolving
  def between(lowerBound: Time, upperBound: Time): IntervalJoined[IN1, IN2, KEY] = {
    val lowerMillis = lowerBound.toMilliseconds
    val upperMillis = upperBound.toMilliseconds
    new IntervalJoined[IN1, IN2, KEY](streamOne, streamTwo, lowerMillis, upperMillis)
  }
}

/**
  * IntervalJoined is a container for two streams that have keys for both sides as well as
  * the time boundaries over which elements should be joined.
  */
@PublicEvolving
class IntervalJoined[IN1, IN2, KEY](private val firstStream: KeyedStream[IN1, KEY],
                                    private val secondStream: KeyedStream[IN2, KEY],
                                    private val lowerBound: Long,
                                    private val upperBound: Long) {
  private var lowerBoundInclusive = true
  private var upperBoundInclusive = true
  def lowerBoundExclusive(): IntervalJoined[IN1, IN2, KEY] = {
    this.lowerBoundInclusive = false
    this
  }
  def upperBoundExclusive(): IntervalJoined[IN1, IN2, KEY] = {
    this.upperBoundInclusive = false
    this
  }
  
  def process[OUT: TypeInformation](
    processJoinFunction: ProcessJoinFunction[IN1, IN2, OUT])
  : DataStream[OUT] = {

  val outType: TypeInformation[OUT] = implicitly[TypeInformation[OUT]]

  val javaJoined = new KeyedJavaStream.IntervalJoined[IN1, IN2, KEY](
    firstStream.javaStream.asInstanceOf[KeyedJavaStream[IN1, KEY]],
    secondStream.javaStream.asInstanceOf[KeyedJavaStream[IN2, KEY]],
    lowerBound,
    upperBound,
    lowerBoundInclusive,
    upperBoundInclusive)
  asScalaStream(javaJoined.process(processJoinFunction, outType))
}
// 以上构建 IntervalJoined 对象，接下来去构建 operater
```

#### ConnectedStreams

```java
// org.apache.flink.streaming.api.datastream.KeyedStream.IntervalJoined
public <OUT> SingleOutputStreamOperator<OUT> process(
    ProcessJoinFunction<IN1, IN2, OUT> processJoinFunction,
    TypeInformation<OUT> outputType) {
  ...
  final ProcessJoinFunction<IN1, IN2, OUT> cleanedUdf = left.getExecutionEnvironment().clean(processJoinFunction);
  // 构建 IntervalJoinOperator，重点
  final IntervalJoinOperator<KEY, IN1, IN2, OUT> operator =
    new IntervalJoinOperator<>(
      lowerBound,
      upperBound,
      lowerBoundInclusive,
      upperBoundInclusive,
      left.getType().createSerializer(left.getExecutionConfig()),
      right.getType().createSerializer(right.getExecutionConfig()),
      cleanedUdf
    );
  // 底层还是基于 connect 实现，ConnectedStreams
  // env 添加 transform，返回 SingleOutputStreamOperator
  return left
    .connect(right)
    .keyBy(keySelector1, keySelector2)
    .transform("Interval Join", outputType, operator);
}
```

底层还是基于 `ConnectedStreams` 实现

```java
// org.apache.flink.streaming.api.datastream.ConnectedStreams
// 整个函数就是构造 TwoInputTransformation，并添加到 env
public <R> SingleOutputStreamOperator<R> transform(String functionName,
    TypeInformation<R> outTypeInfo,
    TwoInputStreamOperator<IN1, IN2, R> operator) {
  ...
  TwoInputTransformation<IN1, IN2, R> transform = new TwoInputTransformation<>(
      inputStream1.getTransformation(),
      inputStream2.getTransformation(),
      functionName,
      operator,
      outTypeInfo,
      environment.getParallelism());
  // 必须是 KeyedStream
  if (inputStream1 instanceof KeyedStream && inputStream2 instanceof KeyedStream) {
    KeyedStream<IN1, ?> keyedInput1 = (KeyedStream<IN1, ?>) inputStream1;
    KeyedStream<IN2, ?> keyedInput2 = (KeyedStream<IN2, ?>) inputStream2;

    TypeInformation<?> keyType1 = keyedInput1.getKeyType();
    TypeInformation<?> keyType2 = keyedInput2.getKeyType();
    if (!(keyType1.canEqual(keyType2) && keyType1.equals(keyType2))) {
      throw new UnsupportedOperationException("Key types if input KeyedStreams " +
          "don't match: " + keyType1 + " and " + keyType2 + ".");
    }

    transform.setStateKeySelectors(keyedInput1.getKeySelector(), keyedInput2.getKeySelector());
    transform.setStateKeyType(keyType1);
  }
  SingleOutputStreamOperator<R> returnStream = new SingleOutputStreamOperator(environment, transform);
  getExecutionEnvironment().addOperator(transform);

  return returnStream;
}
```

#### IntervalJoinOperator

真正干活的是 `IntervalJoinOperator`，也是本文的重点。

```java
// org.apache.flink.streaming.api.operators.co.IntervalJoinOperator
// 有时间限制的 join
// 当流的数据到来时先缓存，然后到另一条流的缓存区匹配，结果传递给 ProcessJoinFunction
// 为避免缓存区无限增大，给 event 设置定时器，到期删除
public IntervalJoinOperator(
    long lowerBound,
    long upperBound,
    boolean lowerBoundInclusive,
    boolean upperBoundInclusive,
    TypeSerializer<T1> leftTypeSerializer,
    TypeSerializer<T2> rightTypeSerializer,
    ProcessJoinFunction<T1, T2, OUT> udf) {
  ...
  // 是否包含上下界
  this.lowerBound = (lowerBoundInclusive) ? lowerBound : lowerBound + 1L;
  this.upperBound = (upperBoundInclusive) ? upperBound : upperBound - 1L;
  ...
}
// 左右流 event 到来时，都会调用 processElement
public void processElement1(StreamRecord<T1> record) throws Exception {
  processElement(record, leftBuffer, rightBuffer, lowerBound, upperBound, true);
}
public void processElement2(StreamRecord<T2> record) throws Exception {
  processElement(record, rightBuffer, leftBuffer, -upperBound, -lowerBound, false);
}
// event 触发
private <THIS, OTHER> void processElement(
    final StreamRecord<THIS> record,
    final MapState<Long, List<IntervalJoinOperator.BufferEntry<THIS>>> ourBuffer,
    final MapState<Long, List<IntervalJoinOperator.BufferEntry<OTHER>>> otherBuffer,
    final long relativeLowerBound,
    final long relativeUpperBound,
    final boolean isLeft) throws Exception {

  final THIS ourValue = record.getValue();
  final long ourTimestamp = record.getTimestamp();
  ...
  // 迟到数据直接丢弃
  if (isLate(ourTimestamp)) {
    return;
  }
  // 将数据先缓存到当前侧的 state
  addToBuffer(ourBuffer, ourValue, ourTimestamp);
  // for 循环是 interval join 的关键
  // 寻找符合 left.time + lower < right.time < left.time + upper,并输出
  for (Map.Entry<Long, List<BufferEntry<OTHER>>> bucket: otherBuffer.entries()) {
    final long timestamp  = bucket.getKey();
    // 时间不匹配数据直接跳过
    if (timestamp < ourTimestamp + relativeLowerBound ||
        timestamp > ourTimestamp + relativeUpperBound) {
      continue;
    }
    // 时间匹配 event 输出到 udf (当前已经 keyby，表示 join key 值已相同)
    for (BufferEntry<OTHER> entry: bucket.getValue()) {
      if (isLeft) {
        collect((T1) ourValue, (T2) entry.element, ourTimestamp, timestamp);
      } else {
        collect((T1) entry.element, (T2) ourValue, timestamp, ourTimestamp);
      }
    }
  }
  // 注册清理过期数据的定时器
  // left：left.time + upper；这里很好理解
  // right: right.time - lower；下面分析为什么是这样
  long cleanupTime = (relativeUpperBound > 0L) ? ourTimestamp + relativeUpperBound : ourTimestamp;
  if (isLeft) {
    internalTimerService.registerEventTimeTimer(CLEANUP_NAMESPACE_LEFT, cleanupTime);
  } else {
    internalTimerService.registerEventTimeTimer(CLEANUP_NAMESPACE_RIGHT, cleanupTime);
  }
}
```

`processElement` 函数是流计算 `join` 的精髓：

- 数据到来先缓存，
- 去另一侧流匹配数据，
- 匹配的结果输出，
- **清理过期数据** 

#### State

`join` 的过程清楚，下面分析过期数据如何清除

``` java
// org.apache.flink.streaming.api.operators.co.IntervalJoinOperator
// 定时器触发；关于定时器，后续会有文章专门分析
public void onEventTime(InternalTimer<K, String> timer) throws Exception {

  long timerTimestamp = timer.getTimestamp();
  String namespace = timer.getNamespace();

  logger.trace("onEventTime @ {}", timerTimestamp);
  // 对照上面设定定时器的代码理解
  switch (namespace) {
    // left ：定义了 time + upper 定时器，到时间删除 time 时间的数据
    // leftBuffer 是 state；right 过期数据 = 当前时间戳 - upperBound(有正负)
    case CLEANUP_NAMESPACE_LEFT: {
      long timestamp = (upperBound <= 0L) ? timerTimestamp : timerTimestamp - upperBound;
      logger.trace("Removing from left buffer @ {}", timestamp);
      leftBuffer.remove(timestamp);
      break;
    }
    // right：定义了 time - low 定时器
    // 定时器是根据定义的事件时间或处理时间来计算，触发表示 min(left.time，right.time) = timerTimestamp
    // 假设 low = -10，那么触发此定时器时间 timerTimestamp = x+10
    // 删除 time = x  event；right 过期数据 = 当前时间戳 + lowerbound(有正负)
    case CLEANUP_NAMESPACE_RIGHT: {
      long timestamp = (lowerBound <= 0L) ? timerTimestamp + lowerBound : timerTimestamp;
      logger.trace("Removing from right buffer @ {}", timestamp);
      rightBuffer.remove(timestamp);
      break;
    }
    default:
      throw new RuntimeException("Invalid namespace " + namespace);
  }
}
```

定时器触发判断哪些数据是过期，然后从 `State` 中删除。借本案例，我们也可以学习 Flink 中 `State` 使用方法。

``` java
// org.apache.flink.streaming.api.operators.co.IntervalJoinOperator
private transient MapState<Long, List<BufferEntry<T1>>> leftBuffer;
private transient MapState<Long, List<BufferEntry<T2>>> rightBuffer;

// app 从 state 中恢复，会调用
@Override
public void initializeState(StateInitializationContext context) throws Exception {
  super.initializeState(context);
  // 每个 state 都必须有名字的，反序列化类，构建 StateDescriptor
  // context 根据 StateDescriptor 从 state 获取对应数据
  // 如果类没有 initializeState 方法，也可以放在 open 函数中使用
  this.leftBuffer = context.getKeyedStateStore().getMapState(new MapStateDescriptor<>(
    LEFT_BUFFER,
    LongSerializer.INSTANCE,
    new ListSerializer<>(new BufferEntrySerializer<>(leftTypeSerializer))
  ));

  this.rightBuffer = context.getKeyedStateStore().getMapState(new MapStateDescriptor<>(
    RIGHT_BUFFER,
    LongSerializer.INSTANCE,
    new ListSerializer<>(new BufferEntrySerializer<>(rightTypeSerializer))
  ));
}
```





## 参考资料

[Flink 流流关联( Interval Join)总结](https://www.jianshu.com/p/11b482394c73)

[Apache Flink 漫谈系列(12) - Time Interval(Time-windowed) JOIN](https://developer.aliyun.com/article/683681)

[Flink DataStream 基于Interval Join实时Join过去一段时间内的数据](https://mp.weixin.qq.com/s/2hnBWqgKBQoVHhNpyW4CZg)

[Flink intervalJoin 使用和原理分析](https://www.jianshu.com/p/d457a6dff349)