---
layout:     post
title:      Flume hdfsSink 数据延迟问题
subtitle:   
date:       2020-05-10
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flume
    - HDFS
    - 源码
---



Flume -> HDFS 时，我们一般会如下设置，将数据按时间落到**表分区**

``` shell
a1.sinks.s1.type = hdfs
a1.sinks.s1.hdfs.path = hdfs://xxx:8020/data/ds=%Y%m%d/
a1.sinks.s1.hdfs.writeFormat = Text
a1.sinks.s1.hdfs.fileType = DataStream
a1.sinks.s1.hdfs.rollSize = 0
a1.sinks.s1.hdfs.rollCount = 100000
a1.sinks.s1.rollInterval = 3600
a1.sinks.s1.hdfs.batchSize = 3000
a1.sinks.s1.hdfs.useLocalTimeStamp = true
```

​	按时间将数据分发到不同的目录下，对应着 `Hive` 的 分区。但这里会出现一个问题：假设数据生成时间 "2020-05-08 23:59:58"，数据包装成 event 落到 `Flume` 的时间是  "2020-05-09 00:00:5"；那么该数据就会落在 "ds=20200509" 这个分区，而不是我们想要的"ds=20200508"分区。那么是不是使数据到 Flume 时间和数据生成时间相同就可以解决这个问题呢？理想情况下是可以，但现实是无法保证保证所有数据都没有延迟。

### 数据时间

​	问题出现了，那么该如何解决呢？我们先来看看 Flume 中关于 hdfsSink 代码

``` java
// HDFSEventSink
public class HDFSEventSink extends AbstractSink implements Configurable, BatchSizeSupported {
	public Status process() throws EventDeliveryException {
  	for (txnEventCount = 0; txnEventCount < batchSize; txnEventCount++) {
        Event event = channel.take();
        if (event == null) {
          break;
        }
        // reconstruct the path name by substituting place holders
        String realPath = BucketPath.escapeString(filePath, event.getHeaders(),
            timeZone, needRounding, roundUnit, roundValue, useLocalTime);
        String realName = BucketPath.escapeString(fileName, event.getHeaders(),
            timeZone, needRounding, roundUnit, roundValue, useLocalTime);
				// lookupPath 文件落到 hdfs 路径和名称
        // 按最上面的 sink 设置，本文关注的是 realPath
        String lookupPath = realPath + DIRECTORY_DELIMITER + realName;
      ...
  }
}
  
  
// BucketPath
public static String escapeString(String in, Map<String, String> headers,
      TimeZone timeZone, boolean needRounding, int unit, int roundDown,
      boolean useLocalTimeStamp) {
   // flume 处理时间
	 long ts = clock.currentTimeMillis();
   while (matcher.find()) {
        String replacement = "";
        ...
        else {
          // The %x pattern.
          // Since we know the match is a single character, we can
          // switch on that rather than the string.
          Preconditions.checkState(matcher.group(1) != null
              && matcher.group(1).length() == 1,
              "Expected to match single character tag in string " + in);
          char c = matcher.group(1).charAt(0);
          replacement = replaceShorthand(c, headers, timeZone,
              needRounding, unit, roundDown, useLocalTimeStamp, ts);
        }
     ...
}
  
  protected static String replaceShorthand(char c, Map<String, String> headers,
      TimeZone timeZone, boolean needRounding, int unit, int roundDown,
      boolean useLocalTimestamp, long ts) {
  	 String timestampHeader = null;
    try {
      // useLocalTimestamp = false 时，
      // event header 必须有 timestamp 且会按此时间来落 hdfs
      if (!useLocalTimestamp) {
        timestampHeader = headers.get("timestamp");
        Preconditions.checkNotNull(timestampHeader, "Expected timestamp in " +
            "the Flume event headers, but it was null");
        ts = Long.valueOf(timestampHeader);
      } else {
        timestampHeader = String.valueOf(ts);
      ...
    SimpleDateFormat format = getSimpleDateFormat(formatString);
    if (timeZone != null) {
      format.setTimeZone(timeZone);
    } else {
      format.setTimeZone(TimeZone.getDefault());
    }
		
    Date date = new Date(ts);
    return format.format(date);
  }
```

看上面代码可知，只要设置 `useLocalTimeStamp = false`，Flume 就会使用 `header` 中的 `timestamp`。那么问题回到如何将 `header` 中的 `timestamp` 设置为 数据的生成时间。Flume 中 `Interceptor` 组件可实现这个功能，不复杂(详细看参考资料)。

### 文件关闭

上面的操作可以将数据落到正确的 hdfs 目录，但还有个问题：Flume 写 hdfs 文件时，自动加 "tmp" 字符串来标识正在写；当满足条件后，才会关闭文件流，文件名中的 "tmp" 才会被删除。本案例的条件

``` shell
# 满足任意即可
a1.sinks.s1.hdfs.rollCount = 100000
a1.sinks.s1.rollInterval = 3600
```

文件 event 超过 2万/文件创建时间超1小时，当前文件流才会关闭。很显然这些要求对于延迟的数据来说很严格：

- rollCount：延迟数据一般不会有这么多(就凌晨那几分钟产生的数据，如果你们公司有那更好)
- rollInterval：1小时后文件才关闭，那么我们 ETL 的整体时间就要往后延迟 1小时，显然这样不可取(文件都落地后才开始 ETL)。不要问为什么这里参数不调小，小文件了解下。

针对如上的问题，有个取巧的方案：

- 假定数据最大延迟五分钟
- 在 "00:00:00" - "00:05:00" 时间段中，强制设置 rollInterval 为10 分钟
- 那么要完整的收集到延迟数据，最晚在 "00:15:00" 即可开始 ETL 工作

当然上面不是一个完美的解决方案，只是提供一个可解决的方案(你还可以将数据的**时间戳设置成主键**，然后落到其他存储系统中，最后根据主键拉取数据到对应的 HDFS 目录)。

上面的方案该如何实现呢？继续看源码

``` java
// BucketWriter
// flume 打开 hdfs 文件
private void open() throws IOException, InterruptedException {
  // 文件名生成过程
	long counter = fileExtensionCounter.incrementAndGet();
  String fullFileName = fileName + "." + counter;
  if (fileSuffix != null && fileSuffix.length() > 0) {
    fullFileName += fileSuffix;
  } else if (codeC != null) {
    fullFileName += codeC.getDefaultExtension();
  }
  bucketPath = filePath + "/" + inUsePrefix
    + fullFileName + inUseSuffix;
  targetPath = filePath + "/" + fullFileName;
	...
   // if time-based rolling is enabled, schedule the roll
    if (rollInterval > 0) {
      Callable<Void> action = new Callable<Void>() {
        public Void call() throws Exception {
          LOG.debug("Rolling file ({}): Roll scheduled after {} sec elapsed.",
                    bucketPath, rollInterval);
          try {
            // Roll the file and remove reference from sfWriters map.
            close(true);
          } catch (Throwable t) {
            LOG.error("Unexpected error", t);
          }
          return null;
        }
      };
      // rollInterval 定时器关闭
      timedRollFuture = timedRollerPool.schedule(action, rollInterval,
                                                 TimeUnit.SECONDS);
    }
}
```

`open` 会注册定时器关闭文件，只要在创建定时器之前判断当前时间是否在 "00:00:00" - "00:05:00"，然后强制赋值 `rollInterval` 即可。



## 参考资料

[Apache Flume 如何解析消息中的事件时间](http://shzhangji.com/cnblogs/2017/08/06/how-to-extract-event-time-in-apache-flume/)

[通过源码分析Flume HDFSSink 写hdfs文件的过程](https://fangjian0423.github.io/2015/07/20/flume-hdfs-sink/)

[Flume-NG源码阅读之Interceptor(原创)](https://www.cnblogs.com/lxf20061900/p/3664602.html)

[修改Flume-NG的hdfs sink解析时间戳源码大幅提高写入性能](https://www.cnblogs.com/lxf20061900/p/4014281.html)

