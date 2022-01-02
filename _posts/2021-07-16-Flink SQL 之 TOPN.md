---
layout:     post
title:      Flink SQL 之 TOPN
subtitle:   
date:       2021-07-16
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
---

```sql
    tEnv.executeSql(
      s"""
         |CREATE TABLE table1 (
         | name STRING,
         | cnt int,
         | procTime as proctime()
         |) WITH (
         |    'connector' = 'kafka',
         |    'properties.bootstrap.servers' = '127.0.0.1:9091',
         |    'scan.startup.mode' = 'latest-offset',
         |    'topic' = 'test',
         |    'properties.group.id' = 'test_group',
         |    'format' = 'json'
         |)
       """.stripMargin)

    tEnv.executeSql(
      s"""
         |CREATE TABLE mysql_table (
         | name STRING,
         | price int
         |) WITH (
         | 'connector' = 'jdbc',
         | 'url' = 'jdbc:mysql://localhost:3306/database',
         | 'username' = 'test',
         | 'password' = 'test',
         | 'table-name' = 'test'
         |)
       """.stripMargin)

    tEnv.executeSql(
      s"""
         |CREATE TABLE sink_table (
         | name STRING,
         | money int
         |) WITH (
         |'connector' = 'print'
         |)
       """.stripMargin)

    // topn
    println(tEnv.explainSql(
      s"""
         |select name,cnt
         |from (
         |select *,
         |ROW_NUMBER() OVER (PARTITION BY name ORDER BY cnt desc) as rk
         |from table1) as n
         |where rk <3
       """.stripMargin, ExplainDetail.JSON_EXECUTION_PLAN))
```

执行计划

```shell
== Abstract Syntax Tree ==
LogicalProject(name=[$0], cnt=[$1])
+- LogicalFilter(condition=[<($3, 3)])
   +- LogicalProject(name=[$0], cnt=[$1], procTime=[PROCTIME()], rk=[ROW_NUMBER() OVER (PARTITION BY $0 ORDER BY $1 DESC NULLS LAST)])
      +- LogicalTableScan(table=[[default_catalog, default_database, table1]])
      
# LogicalProject.rk 和 LogicalFilter.condition 触发合成 Rank
== Optimized Physical Plan ==
Rank(strategy=[AppendFastStrategy], rankType=[ROW_NUMBER], rankRange=[rankStart=1, rankEnd=2], partitionBy=[name], orderBy=[cnt DESC], select=[name, cnt])
+- Exchange(distribution=[hash[name]])
   +- TableSourceScan(table=[[default_catalog, default_database, table1]], fields=[name, cnt])

== Optimized Execution Plan ==
Rank(strategy=[AppendFastStrategy], rankType=[ROW_NUMBER], rankRange=[rankStart=1, rankEnd=2], partitionBy=[name], orderBy=[cnt DESC], select=[name, cnt])
+- Exchange(distribution=[hash[name]])
   +- TableSourceScan(table=[[default_catalog, default_database, table1]], fields=[name, cnt])
```

### TopN

#### rule

`StreamPhysicalRankRule`

规则匹配在<去重>讲解

`StreamExecRank` -> `StreamExecRank`

根据 'rankStrategy' 不同的到处理函数，相关代码可以查看 'org.apache.flink.table.planner.plan.utils.RankProcessStrategy#analyzeRankProcessStrategies'

- AppendOnlyTopNFunction：结果只追加，不更新
- RetractableTopNFunction：类似于回撤流，结果会更新，前提是输入数据没有主键，或者主键与partitionKey不同
- UpdatableTopNFunction：快速更新，前提是输入数据有主键，且结果单调递增/递减(数据有序)，还要求orderKey的排序规则与结果的单调性相反

##### AppendOnlyTopNFunction

```java
// org.apache.flink.table.runtime.operators.rank.AppendOnlyTopNFunction#processElement
public void processElement(RowData input, Context context, Collector<RowData> out)
        throws Exception {
    long currentTime = context.timerService().currentProcessingTime();
    // 注册清除状态的定时器，OnTimer 函数中清状态
    registerProcessingCleanupTimer(context, currentTime);
    // Map<PartitionKey, TopNBuffer>
    // TopNBuffer(TreeMap<sortKey, RowData>)：这是一个好的TopN类，自己项目使用到也可以用
    // 初始化PartitionKey，对应的TopNBuffer
    initHeapStates();
    // 确定 rankend，本例中是常量2
    initRankEnd(input);

    RowData sortKey = sortKeySelector.getKey(input);
    // 检测input 数据是否在 topn中，代码在org.apache.flink.table.runtime.operators.rank.TopNBuffer#checkSortKeyInBufferRange
    // 具体逻辑，查看input 与TreeMap值比较，是否在 topn个数内
    if (checkSortKeyInBufferRange(sortKey, buffer)) {
        // input 插入 TreeMap
        buffer.put(sortKey, inputRowSer.copy(input));
        Collection<RowData> inputs = buffer.get(sortKey);
        // update data state
        // copy a new collection to avoid mutating state values, see CopyOnWriteStateMap,
        // otherwise, the result might be corrupt.
        // don't need to perform a deep copy, because RowData elements will not be updated
        // 保存TreeMap 数据到状态
        dataState.put(sortKey, new ArrayList<>(inputs));
        if (outputRankNumber || hasOffset()) {
            // the without-number-algorithm can't handle topN with offset,
            // so use the with-number-algorithm to handle offset
            processElementWithRowNumber(sortKey, input, out);
        } else {
            processElementWithoutRowNumber(input, out);
        }
    }
    // 若不满足无输出也不会保存状态
}

// LRU，treeMap 初始化
private void initHeapStates() throws Exception {
    requestCount += 1;
    RowData currentKey = (RowData) keyContext.getCurrentKey();
    buffer = kvSortedMap.get(currentKey);
    if (buffer == null) {
        buffer = new TopNBuffer(sortKeyComparator, ArrayList::new);
        kvSortedMap.put(currentKey, buffer);
        // restore buffer
        // 恢复 buffer，为什么这里会有恢复操作？
        // kvSortedMap 是在内存中的LRU，size=默认1000，很久不用的partitionkey会被删除
        // 在这里从状态 dataState重新组装成 buffer(treeMap)
        Iterator<Map.Entry<RowData, List<RowData>>> iter = dataState.iterator();
        if (iter != null) {
            while (iter.hasNext()) {
                Map.Entry<RowData, List<RowData>> entry = iter.next();
                RowData sortKey = entry.getKey();
                List<RowData> values = entry.getValue();
                // the order is preserved
                buffer.putAll(sortKey, values);
            }
        }
    } else {
        hitCount += 1;
    }
}

// 定时清理数据
public void onTimer(long timestamp, OnTimerContext ctx, Collector<RowData> out)
        throws Exception {
    if (stateCleaningEnabled) {
        // cleanup cache
        kvSortedMap.remove(keyContext.getCurrentKey());
        cleanupState(dataState);
    }
}
```

- 数据
  - 状态：**dataState** <sortkey, list<Rowdata>>，当前key下**所有数据**，
  - kvSortedMap 保存每个key的 topn数据，是在**内存**中；但内存空间有限，使用`LRU` 算法，被删除的数据在使用时从状态 dataState 重新恢复
  - 定时器，清理kvSortedMap 和dataState
- 带rank 输出
  - input 输出
  - input 插入后导致之前的数据往后排序，产生回撤(撤回之前的rank，并输出新的rank)，有可能产生**大量回撤数据**
  - Buffer(treeMap) 和dataState **删除掉因input 插入导致在topn 后的数据**
- 不带 rank 输出
  - input 输出
  - 若input加入超出 topn
    - 回撤之前输出的数据，**只有一条**
    - Buffer(treeMap) 和dataState **删除掉因input 插入导致在topn 后的数据**



##### RetractableTopNFunction

```java
public void processElement(RowData input, Context ctx, Collector<RowData> out)
        throws Exception {
    long currentTime = ctx.timerService().currentProcessingTime();
    // register state-cleanup timer
    registerProcessingCleanupTimer(ctx, currentTime);
    initRankEnd(input);
    SortedMap<RowData, Long> sortedMap = treeMap.value();
    if (sortedMap == null) {
        sortedMap = new TreeMap<>(sortKeyComparator);
    }
    RowData sortKey = sortKeySelector.getKey(input);
    boolean isAccumulate = RowDataUtil.isAccumulateMsg(input);
    input.setRowKind(RowKind.INSERT); // erase row kind for further state accessing
    if (isAccumulate) {
        // update sortedMap
        if (sortedMap.containsKey(sortKey)) {
            sortedMap.put(sortKey, sortedMap.get(sortKey) + 1);
        } else {
            sortedMap.put(sortKey, 1L);
        }

        // emit
        if (outputRankNumber || hasOffset()) {
            // the without-number-algorithm can't handle topN with offset,
            // so use the with-number-algorithm to handle offset
            // rank 也输出
            emitRecordsWithRowNumber(sortedMap, sortKey, input, out);
        } else {
            emitRecordsWithoutRowNumber(sortedMap, sortKey, input, out);
        }
        // update data state
        List<RowData> inputs = dataState.get(sortKey);
        if (inputs == null) {
            // the sort key is never seen
            inputs = new ArrayList<>();
        }
        inputs.add(input);
        dataState.put(sortKey, inputs);
    } else {
        final boolean stateRemoved;
        // emit updates first
        if (outputRankNumber || hasOffset()) {
            // the without-number-algorithm can't handle topN with offset,
            // so use the with-number-algorithm to handle offset
            // rank 也输出
            stateRemoved = retractRecordWithRowNumber(sortedMap, sortKey, input, out);
        } else {
            stateRemoved = retractRecordWithoutRowNumber(sortedMap, sortKey, input, out);
        }

        // and then update sortedMap
        if (sortedMap.containsKey(sortKey)) {
            long count = sortedMap.get(sortKey) - 1;
            if (count == 0) {
                sortedMap.remove(sortKey);
            } else {
                sortedMap.put(sortKey, count);
            }
        } else {
            if (sortedMap.isEmpty()) {
                if (lenient) {
                    LOG.warn(STATE_CLEARED_WARN_MSG);
                } else {
                    throw new RuntimeException(STATE_CLEARED_WARN_MSG);
                }
            } else {
                throw new RuntimeException(
                        "Can not retract a non-existent record. This should never happen.");
            }
        }

        if (!stateRemoved) {
            // the input record has not been removed from state
            // should update the data state
            List<RowData> inputs = dataState.get(sortKey);
            if (inputs != null) {
                // comparing record by equaliser
                Iterator<RowData> inputsIter = inputs.iterator();
                while (inputsIter.hasNext()) {
                    if (equaliser.equals(inputsIter.next(), input)) {
                        inputsIter.remove();
                        break;
                    }
                }
                if (inputs.isEmpty()) {
                    dataState.remove(sortKey);
                } else {
                    dataState.put(sortKey, inputs);
                }
            }
        }
    }
    treeMap.update(sortedMap);
}
```

treeMap => SortedMap<sortkey, Long>，`状态`中

- 保存sortkey和此sortkey的个数，供排序使用

dataState => Map<sortkey, List<RowData>>，包含所有数据，状态一直累加

- insert
  - 带rank 输出
    - 根据treeMap 判断，新的input 是否在topn内
      - 若不在topn，无操作
      - 若在输出input，并回撤因为input而往后排的数据(数据会有**多条**)
  - 不带rank 输出
    - 根据treeMap 判断，新的input 是否在topn内
      - 若不在topn，无操作
      - 若在输出input，并回撤因为input而往后排的数据(只会有**一条**)
  - Input 数据加入到 `dataState`
- Retract
  - 带 rank输出
    - 若不在topn，无操作
    - 若在输出input，并回撤因为input而往后排的数据(数据会有**多条**)
  - 不带rank输出
    - 若不在topn，无操作
    - 若在输出input，并回撤因为input而往后排的数据(只会有**一条**)
  - SortedMap 更新(个数减一/删除此sortkey)，



##### UpdatableTopNFunction



#### 总结

- 不输出 rank ，可以减少很多数据输出
- AppendOnlyTopNFunction：排序key(treeMap)内存LRU方式，TreeMap 排序
- RetractableTopNFunction：排序(treeMap)状态里，只保存个数，每次操作都要从dataState 获取完整的RowData，TreeMap 排序
- UpdatableTopNFunction：



### 去重

当rk=1时，可以达到去重的效果。但当rk=1时，还用treemap来做排序太麻烦了，我们用dataStream API 一般都使用 valueState来保存。

#### rule

```java
//org.apache.flink.table.planner.plan.rules.physical.stream.StreamPhysicalDeduplicateRule#canConvertToDeduplicate
def canConvertToDeduplicate(rank: FlinkLogicalRank): Boolean = {
  val sortCollation = rank.orderKey
  val rankRange = rank.rankRange
  val isRowNumberType = rank.rankType == RankType.ROW_NUMBER
  val isLimit1 = rankRange match {
    case rankRange: ConstantRankRange =>
      rankRange.getRankStart == 1 && rankRange.getRankEnd == 1
    case _ => false
  }
  val inputRowType = rank.getInput.getRowType
  val isSortOnTimeAttribute = sortOnTimeAttribute(sortCollation, inputRowType)
  !rank.outputRankNumber && isLimit1 && isSortOnTimeAttribute && isRowNumberType
}
```

- rk=1
- **按处理时间/事件事件排序**
- 不输出rk值

满足以上三点，row_number就不是翻译成TopN而是`Deduplicate`

#### StreamExecDeduplicate

实现就是基于 ValueState，可以基于 minibatch 优化，但优化后还是每个event 都会单独往下游输出(没有减少数据量只减少访问state次数)。

