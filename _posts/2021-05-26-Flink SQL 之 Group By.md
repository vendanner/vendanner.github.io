---
layout:     post
title:      Flink SQL 之 Group By
subtitle:   
date:       2021-05-26
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - SQL
---

在之前的[Flink 之 Retract](https://vendanner.github.io/2020/06/02/Flink-%E4%B9%8B-Retract/) 介绍回撤的重要性(订正之前的错误状态)。本文结合 `group by` 再次回顾下回撤在 Flink 中的应用。

```sql
CREATE TABLE table1 (
 name STRING,
 cnt int
) WITH (
'connector' = 'kafka',
 'topic' = 'products_binlog',
 'properties.bootstrap.servers' = '127.0.0.1:9092',
 'properties.group.id' = 'testGroup',
 'scan.startup.mode' = 'earliest-offset',
 'format' = 'canal-json'
)
         
select name
sum(cnt),
max(cnt)
from table1
group by name
```

使用 `explainSql` 打印执行计划，结合之前 Flink SQL 翻译过程，找到 `group by ` 具体 ExecNode：`StreamExecGroupAggregate`。Transformation 有两种实现 `GroupAggFunction` 和 `MiniBatchGroupAggFunction`。

### 流程

以`GroupAggFunction` 为例

使用 group by 后按 key 分组存储数据(state)，新来一条数据时，经过 state 计算后

- acc 有值
  - 若之前无值(第一次计算)，直接输出 INSERT
  - 若之前有值
    - state 设置不清除且计算后的 state 与之前相同，无输出直接返回
    - state 有设置清楚，先输出一条 UPDATE_BEFORE，再输出一条 UPDATE_AFTER

- acc 无值，若之前是有值，则输出 DELETE(删除之前的数据)

state 如何计算呢？分为两种情况

- event 是回撤状态，state 做 `retract`
- 不是回撤状态， state 做 `acc`

计算的具体实现看下面分析。`group by` 的语义非常简单，这里的重点对 state 计算，state 是指那些数据呢？

### 代码生成

本案例中 state 是指 sum(cnt) 和max(cnt)，当做 acc/retract 时，由**一个聚合函数**完成所有操作。在未执行前，我们无法确定聚合函数具体的操作是 sum or max or count or 组合 ，在执行时由动态代码生成。

```java
public final class GroupAggsHandler implements org.apache.flink.table.runtime.generated.AggsHandleFunction {
    int agg0_sum;
    boolean agg0_sumIsNull;
    long agg0_count;
    boolean agg0_countIsNull;
    private transient org.apache.flink.table.planner.functions.aggfunctions.MaxWithRetractAggFunction function_org$apache$flink$table$planner$functions$aggfunctions$MaxWithRetractAggFunction$d78f624eeff2a86742b5f64899608448;
    private transient org.apache.flink.table.runtime.typeutils.ExternalSerializer externalSerializer$2;
    private transient org.apache.flink.table.runtime.typeutils.ExternalSerializer externalSerializer$3;
    private org.apache.flink.table.runtime.dataview.StateMapView agg1$map_dataview;
    private org.apache.flink.table.data.binary.BinaryRawValueData agg1$map_dataview_raw_value;
    private org.apache.flink.table.runtime.dataview.StateMapView agg1$map_dataview_backup;
    private org.apache.flink.table.data.binary.BinaryRawValueData agg1$map_dataview_backup_raw_value;
    long agg2_count1;
    boolean agg2_count1IsNull;
    private transient org.apache.flink.table.data.conversion.StructuredObjectConverter converter$4;
    org.apache.flink.table.data.GenericRowData acc$6 = new org.apache.flink.table.data.GenericRowData(4);
    org.apache.flink.table.data.GenericRowData acc$7 = new org.apache.flink.table.data.GenericRowData(4);
    org.apache.flink.table.data.UpdatableRowData field$11;
    private org.apache.flink.table.data.RowData agg1_acc_internal;
    private org.apache.flink.table.planner.functions.aggfunctions.MaxWithRetractAggFunction.MaxWithRetractAccumulator agg1_acc_external;
    org.apache.flink.table.data.GenericRowData aggValue$41 = new org.apache.flink.table.data.GenericRowData(2);
    private org.apache.flink.table.runtime.dataview.StateDataViewStore store;

    public GroupAggsHandler(java.lang.Object[] references) throws Exception {
        function_org$apache$flink$table$planner$functions$aggfunctions$MaxWithRetractAggFunction$d78f624eeff2a86742b5f64899608448 = (((org.apache.flink.table.planner.functions.aggfunctions.MaxWithRetractAggFunction) references[0]));
        externalSerializer$2 = (((org.apache.flink.table.runtime.typeutils.ExternalSerializer) references[1]));
        externalSerializer$3 = (((org.apache.flink.table.runtime.typeutils.ExternalSerializer) references[2]));
        converter$4 = (((org.apache.flink.table.data.conversion.StructuredObjectConverter) references[3]));
    }

    private org.apache.flink.api.common.functions.RuntimeContext getRuntimeContext() {
        return store.getRuntimeContext();
    }

    @Override
    public void open(org.apache.flink.table.runtime.dataview.StateDataViewStore store) throws Exception {
        this.store = store;

        function_org$apache$flink$table$planner$functions$aggfunctions$MaxWithRetractAggFunction$d78f624eeff2a86742b5f64899608448.open(new org.apache.flink.table.functions.FunctionContext(store.getRuntimeContext()));

        agg1$map_dataview = (org.apache.flink.table.runtime.dataview.StateMapView) store.getStateMapView("agg1$map", false, externalSerializer$2, externalSerializer$3);
        agg1$map_dataview_raw_value = org.apache.flink.table.data.binary.BinaryRawValueData.fromObject(agg1$map_dataview);

        agg1$map_dataview_backup = (org.apache.flink.table.runtime.dataview.StateMapView) store.getStateMapView("agg1$map", false, externalSerializer$2, externalSerializer$3);
        agg1$map_dataview_backup_raw_value = org.apache.flink.table.data.binary.BinaryRawValueData.fromObject(agg1$map_dataview_backup);

        converter$4.open(getRuntimeContext().getUserCodeClassLoader());

    }

    /**
     * 累加计算
     * @param accInput
     * @throws Exception
     */
    @Override
    public void accumulate(org.apache.flink.table.data.RowData accInput) throws Exception {

        int field$13;
        boolean isNull$13;
        boolean isNull$14;
        int result$15;
        boolean isNull$18;
        long result$19;
        boolean isNull$21;
        long result$22;
        isNull$13 = accInput.isNullAt(1);
        field$13 = -1;
        if (!isNull$13) {
            field$13 = accInput.getInt(1);
        }

        int result$17 = -1;
        boolean isNull$17;
        if (isNull$13) {

            isNull$17 = agg0_sumIsNull;
            if (!isNull$17) {
                result$17 = agg0_sum;
            }
        } else {
            int result$16 = -1;
            boolean isNull$16;
            if (agg0_sumIsNull) {
                // sum 之前为 null，input 直接赋值
                isNull$16 = isNull$13;
                if (!isNull$16) {
                    result$16 = field$13;
                }
            } else {
                // sum 之前不为 null，input + agg0_sum
                isNull$14 = agg0_sumIsNull || isNull$13;
                result$15 = -1;
                if (!isNull$14) {
                    result$15 = (int) (agg0_sum + field$13);
                }

                isNull$16 = isNull$14;
                if (!isNull$16) {
                    result$16 = result$15;
                }
            }
            // result$17 保存最终 sum 结果
            isNull$17 = isNull$16;
            if (!isNull$17) {
                result$17 = result$16;
            }

        }
        // 到此处 sum 计算结束
        agg0_sum = result$17;
        agg0_sumIsNull = isNull$17;

        long result$20 = -1L;
        boolean isNull$20;
        if (isNull$13) {

            isNull$20 = agg0_countIsNull;
            if (!isNull$20) {
                result$20 = agg0_count;
            }
        } else {
            isNull$18 = agg0_countIsNull || false;
            result$19 = -1L;
            if (!isNull$18) {
                result$19 = (long) (agg0_count + ((long) 1L));

            }
            isNull$20 = isNull$18;
            if (!isNull$20) {
                result$20 = result$19;
            }
        }
        // 计算 sum 已累加的个数
        agg0_count = result$20;
        agg0_countIsNull = isNull$20;
        // 调用 max的 accumulate
        function_org$apache$flink$table$planner$functions$aggfunctions$MaxWithRetractAggFunction$d78f624eeff2a86742b5f64899608448.accumulate(agg1_acc_external, isNull$13 ? null : ((java.lang.Integer) field$13));

        isNull$21 = agg2_count1IsNull || false;
        result$22 = -1L;
        if (!isNull$21) {
            result$22 = (long) (agg2_count1 + ((long) 1L));
        }

        agg2_count1 = result$22;
        agg2_count1IsNull = isNull$21;

    }

    /**
     * 回撤计算
     * @param retractInput
     * @throws Exception
     */
    @Override
    public void retract(org.apache.flink.table.data.RowData retractInput) throws Exception {

        int field$23;
        boolean isNull$23;
        boolean isNull$24;
        int result$25;
        boolean isNull$26;
        int result$27;
        boolean isNull$30;
        long result$31;
        boolean isNull$33;
        long result$34;
        isNull$23 = retractInput.isNullAt(1);
        field$23 = -1;
        if (!isNull$23) {
            field$23 = retractInput.getInt(1);
        }

        int result$29 = -1;
        boolean isNull$29;
        if (isNull$23) {
            isNull$29 = agg0_sumIsNull;
            if (!isNull$29) {
                result$29 = agg0_sum;
            }
        } else {
            int result$28 = -1;
            boolean isNull$28;
            if (agg0_sumIsNull) {
                isNull$24 = false || isNull$23;
                result$25 = -1;
                if (!isNull$24) {
                    result$25 = (int) (((int) 0) - field$23);
                }
                isNull$28 = isNull$24;
                if (!isNull$28) {
                    result$28 = result$25;
                }
            } else {
                isNull$26 = agg0_sumIsNull || isNull$23;
                result$27 = -1;
                if (!isNull$26) {
                    result$27 = (int) (agg0_sum - field$23);
                }
                isNull$28 = isNull$26;
                if (!isNull$28) {
                    result$28 = result$27;
                }
            }
            isNull$29 = isNull$28;
            if (!isNull$29) {
                result$29 = result$28;
            }
        }
        // agg_sum = 之前的agg0_sum - input 值
        agg0_sum = result$29;
        agg0_sumIsNull = isNull$29;

        long result$32 = -1L;
        boolean isNull$32;
        if (isNull$23) {
            isNull$32 = agg0_countIsNull;
            if (!isNull$32) {
                result$32 = agg0_count;
            }
        } else {
            isNull$30 = agg0_countIsNull || false;
            result$31 = -1L;
            if (!isNull$30) {
                result$31 = (long) (agg0_count - ((long) 1L));
            }
            isNull$32 = isNull$30;
            if (!isNull$32) {
                result$32 = result$31;
            }
        }
        // 若input 不为null，agg_count-1
        agg0_count = result$32;
        agg0_countIsNull = isNull$32;
        // max retract
        function_org$apache$flink$table$planner$functions$aggfunctions$MaxWithRetractAggFunction$d78f624eeff2a86742b5f64899608448.retract(agg1_acc_external, isNull$23 ? null : ((java.lang.Integer) field$23));

        isNull$33 = agg2_count1IsNull || false;
        result$34 = -1L;
        if (!isNull$33) {
            result$34 = (long) (agg2_count1 - ((long) 1L));
        }

        agg2_count1 = result$34;
        agg2_count1IsNull = isNull$33;
    }

    @Override
    public void merge(org.apache.flink.table.data.RowData otherAcc) throws Exception {
        throw new java.lang.RuntimeException("This function not require merge method, but the merge method is called.");
    }

    /**
     * acc 初始化 agg0_sum、agg0_count、agg1_acc_internal、agg2_count1
     * @param acc
     * @throws Exception
     */
    @Override
    public void setAccumulators(org.apache.flink.table.data.RowData acc) throws Exception {

        int field$8;
        boolean isNull$8;
        long field$9;
        boolean isNull$9;
        org.apache.flink.table.data.RowData field$10;
        boolean isNull$10;
        long field$12;
        boolean isNull$12;
        isNull$8 = acc.isNullAt(0);
        field$8 = -1;
        if (!isNull$8) {
            field$8 = acc.getInt(0);
        }
        isNull$9 = acc.isNullAt(1);
        field$9 = -1L;
        if (!isNull$9) {
            field$9 = acc.getLong(1);
        }
        isNull$12 = acc.isNullAt(3);
        field$12 = -1L;
        if (!isNull$12) {
            field$12 = acc.getLong(3);
        }

        isNull$10 = acc.isNullAt(2);
        field$10 = null;
        if (!isNull$10) {
            field$10 = acc.getRow(2, 3);
        }
        field$11 = null;
        if (!isNull$10) {
            field$11 = new org.apache.flink.table.data.UpdatableRowData(
                    field$10,
                    3);

            agg1$map_dataview_raw_value.setJavaObject(agg1$map_dataview);
            field$11.setField(2, agg1$map_dataview_raw_value);
        }

        agg0_sum = field$8;
        agg0_sumIsNull = isNull$8;

        agg0_count = field$9;
        agg0_countIsNull = isNull$9;

        agg1_acc_internal = field$11;
        agg1_acc_external = (org.apache.flink.table.planner.functions.aggfunctions.MaxWithRetractAggFunction.MaxWithRetractAccumulator) converter$4.toExternal((org.apache.flink.table.data.RowData) agg1_acc_internal);

        agg2_count1 = field$12;
        agg2_count1IsNull = isNull$12;
    }

    @Override
    public void resetAccumulators() throws Exception {

        agg0_sum = ((int) -1);
        agg0_sumIsNull = true;

        agg0_count = ((long) 0L);
        agg0_countIsNull = false;

        agg1_acc_external = (org.apache.flink.table.planner.functions.aggfunctions.MaxWithRetractAggFunction.MaxWithRetractAccumulator) function_org$apache$flink$table$planner$functions$aggfunctions$MaxWithRetractAggFunction$d78f624eeff2a86742b5f64899608448.createAccumulator();
        agg1_acc_internal = (org.apache.flink.table.data.RowData) converter$4.toInternalOrNull((org.apache.flink.table.planner.functions.aggfunctions.MaxWithRetractAggFunction.MaxWithRetractAccumulator) agg1_acc_external);

        agg2_count1 = ((long) 0L);
        agg2_count1IsNull = false;
    }

    /**
     * 返回 acc
     * @return
     * @throws Exception
     */
    @Override
    public org.apache.flink.table.data.RowData getAccumulators() throws Exception {

        acc$7 = new org.apache.flink.table.data.GenericRowData(4);

        if (agg0_sumIsNull) {
            acc$7.setField(0, null);
        } else {
            acc$7.setField(0, agg0_sum);
        }

        if (agg0_countIsNull) {
            acc$7.setField(1, null);
        } else {
            acc$7.setField(1, agg0_count);
        }

        agg1_acc_internal = (org.apache.flink.table.data.RowData) converter$4.toInternalOrNull((org.apache.flink.table.planner.functions.aggfunctions.MaxWithRetractAggFunction.MaxWithRetractAccumulator) agg1_acc_external);
        if (false) {
            acc$7.setField(2, null);
        } else {
            acc$7.setField(2, agg1_acc_internal);
        }

        if (agg2_count1IsNull) {
            acc$7.setField(3, null);
        } else {
            acc$7.setField(3, agg2_count1);
        }

        return acc$7;
    }

    /**
     * 创建
     *
     * @return
     * @throws Exception
     */
    @Override
    public org.apache.flink.table.data.RowData createAccumulators() throws Exception {

        acc$6 = new org.apache.flink.table.data.GenericRowData(4);

        if (true) {
            acc$6.setField(0, null);
        } else {
            acc$6.setField(0, ((int) -1));
        }

        if (false) {
            acc$6.setField(1, null);
        } else {
            acc$6.setField(1, ((long) 0L));
        }

        org.apache.flink.table.data.RowData acc_internal$5 = (org.apache.flink.table.data.RowData) (org.apache.flink.table.data.RowData) converter$4.toInternalOrNull((org.apache.flink.table.planner.functions.aggfunctions.MaxWithRetractAggFunction.MaxWithRetractAccumulator) function_org$apache$flink$table$planner$functions$aggfunctions$MaxWithRetractAggFunction$d78f624eeff2a86742b5f64899608448.createAccumulator());
        if (false) {
            acc$6.setField(2, null);
        } else {
            acc$6.setField(2, acc_internal$5);
        }

        if (false) {
            acc$6.setField(3, null);
        } else {
            acc$6.setField(3, ((long) 0L));
        }

        return acc$6;
    }

    /**
     * 获取 agg
     * @return
     * @throws Exception
     */
    @Override
    public org.apache.flink.table.data.RowData getValue() throws Exception {

        boolean isNull$35;
        boolean result$36;

        aggValue$41 = new org.apache.flink.table.data.GenericRowData(2);

        isNull$35 = agg0_countIsNull || false;
        result$36 = false;
        if (!isNull$35) {
            result$36 = agg0_count == ((long) 0L);
        }

        int result$37 = -1;
        boolean isNull$37;
        if (result$36) {
            isNull$37 = true;
            if (!isNull$37) {
                result$37 = ((int) -1);
            }
        } else {
            isNull$37 = agg0_sumIsNull;
            if (!isNull$37) {
                result$37 = agg0_sum;
            }
        }
        if (isNull$37) {
            aggValue$41.setField(0, null);
        } else {
            aggValue$41.setField(0, result$37);
        }

        java.lang.Integer value_external$38 = (java.lang.Integer)
                function_org$apache$flink$table$planner$functions$aggfunctions$MaxWithRetractAggFunction$d78f624eeff2a86742b5f64899608448.getValue(agg1_acc_external);
        java.lang.Integer value_internal$39 =
                value_external$38;
        boolean valueIsNull$40 = value_internal$39 == null;

        if (valueIsNull$40) {
            aggValue$41.setField(1, null);
        } else {
            aggValue$41.setField(1, value_internal$39);
        }
        return aggValue$41;
    }

    @Override
    public void cleanup() throws Exception {
        agg1$map_dataview.clear();
    }

    @Override
    public void close() throws Exception {
        function_org$apache$flink$table$planner$functions$aggfunctions$MaxWithRetractAggFunction$d78f624eeff2a86742b5f64899608448.close();
    }
}
```

- createAccumulators 函数：创建
  - group by 涉及到两个状态的维护：`max` 和 `sum`，合称为 aggState(内存中，每次调用都会初始化)
  - 创建包含四列的 RowData：每个状态维护2类值(具体作用下面介绍) ->  MaxWithRetractAggFunction 也会调用自己的createAccumulators
- setAccumulators 函数：初始化
  - 将**之前存在**或为null 的  accState(flink state保存) 设置进 `GroupAggsHandler`，初始化 Handle 中 acc 值
  - Sum、agg0_count(sum 已累加的个数)、max、count(参与计算的数据个数) 四个状态，后续 getValue 时会用到
- getValue 函数：获取之前的 agg 值(setAccumulators 刚初始化进去)
  - agg：Sum 和 max
- accumulate 函数，累加：input RowData RowKind 是正
  - sum：加上 input 中的值
  - count：input 不为null ，+1
  - max：MaxWithRetractAggFunction.accumulate
  - count：+1
- retract 函数：回撤操作，input RowData RowKind 是负
  - sum：减去 input 中的值
  - count：input 不为null ，-1
  - max：MaxWithRetractAggFunction.retract
  - count：-1
- getAccumulators 函数：返回 acc

```shell
流程如下：创建、初始化、获取之前状态、累加/回撤、获取当前状态
	createAccumulators
	-> setAccumulators
		-> getValue
			-> accumulate
			-> retract
				-> getValue
				 -> getAccumulators
```

注意这里的 MaxWithRetractAggFunction ，是Flink 内置函数。计算 state 生成的动态代码中，没有包含Max代码而是直接调用 MaxWithRetractAggFunction。动态代码做的事情就是把state 中的每个部分(sum/count) 都组合起，变成完整的 state 去计算。

### MiniBatch

Flink 是 event 触发，来一条计算一次，吞吐量肯定没有批处理好。Flink 提供 miniBatch 设置，将event 攒批后一起处理提升吞吐量(也提高了延迟)。

`MiniBatchGroupAggFunction` 相对于 `GroupAggFunction` 多了哪些操作呢？

```shell
"table.exec.mini-batch.enabled" = "true"         # 启用
"table.exec.mini-batch.allow-latency" = "5s"    # 缓存超时时长
"table.exec.mini-batch.size" = "5000"            # 缓存大小
```

以上是参数设置

#### Buffer

```java
// org.apache.flink.table.runtime.operators.aggregate.MiniBatchGroupAggFunction
@Override
public List<RowData> addInput(@Nullable List<RowData> value, RowData input) throws Exception {
  List<RowData> bufferedRows = value;
  if (value == null) {
    bufferedRows = new ArrayList<>();
  }
  // input 缓存到 bufferedRows
  // input row maybe reused, we need deep copy here
  bufferedRows.add(inputRowSerializer.copy(input));
  return bufferedRows;
}
```

mini batch 肯定是有攒批的过程，event 输入时先放到 buffer 中，然后一起处理。可以减少与 `State` 交互，本来是一个event一次，现在是一批一次。

```java
// org.apache.flink.table.runtime.operators.aggregate.MiniBatchGroupAggFunction#finishBundle
for (Map.Entry<RowData, List<RowData>> entry : buffer.entrySet()) {
  RowData currentKey = entry.getKey();
  List<RowData> inputRows = entry.getValue();
  ...
  ctx.setCurrentKey(currentKey);
  RowData acc = accState.value();
  if (acc == null) {
    Iterator<RowData> inputIter = inputRows.iterator();
    while (inputIter.hasNext()) {
      RowData current = inputIter.next();
      if (isRetractMsg(current)) {
        inputIter.remove(); // remove all the beginning retraction messages
      } else {
        break;
      }
    }
	...
  for (RowData input : inputRows) {
    if (isAccumulateMsg(input)) {
      function.accumulate(input);
    } else {
      function.retract(input);
    }
  }
```

`finishBundle` 去处理 buffer 中的 event，那什么时候触发呢？

#### Size

触发：满足大小/时间间隔

`MiniBatchGroupAggFunction` 函数是 `KeyedMapBundleOperator` 的 udf。

算子包含 function，相同的道理 `KeyedProcessOperator` 包含着 `GroupAggFunction`。

event -> Operator -> Function

```java
// org.apache.flink.table.runtime.operators.bundle.AbstractMapBundleOperator#processElement
public void processElement(StreamRecord<IN> element) throws Exception {
    // get the key and value for the map bundle
    final IN input = element.getValue();
    final K bundleKey = getKey(input);
    final V bundleValue = bundle.get(bundleKey);

    // get a new value after adding this element to bundle
    // 保存新的event 到buffer
    final V newBundleValue = function.addInput(bundleValue, input);

    // update to map bundle
    bundle.put(bundleKey, newBundleValue);
    // 计数
    numOfElements++;
    // 计数器是否满足条件可以触发，满足size 就会调用下面的finishBundle 函数
    // 详细原理查看 CountBundleTrigger
    bundleTrigger.onElement(input);
}


// org.apache.flink.table.runtime.operators.bundle.AbstractMapBundleOperator#finishBundle
public void finishBundle() throws Exception {
    if (!bundle.isEmpty()) {
        // 清除event 计数
        numOfElements = 0;
        // 调用finishBundle 处理events
        function.finishBundle(bundle, collector);
        // 清空event 缓存
        bundle.clear();
    }
    // 计时复位
    bundleTrigger.reset();
}
// watermark(时间间隔) 也会触发 finishBundle
public void processWatermark(Watermark mark) throws Exception {
    finishBundle();
    super.processWatermark(mark);
}
```

#### 间隔

按上面的案例，每5s 也会触发一次计算。

启用 miniBatch后，会在Source 后面多加一个 `MiniBatchAssigner` 算子，对应的物理节点 `StreamExecMiniBatchAssigner`。

```shell
{
    "id" : 2,
    "type" : "MiniBatchAssigner(interval=[5000ms], mode=[ProcTime])",
    "pact" : "Operator",
    "contents" : "MiniBatchAssigner(interval=[5000ms], mode=[ProcTime])",
    "parallelism" : 8,
    "predecessors" : [ {
      "id" : 1,
      "ship_strategy" : "FORWARD",
      "side" : "second"
    } ]
  }
```

Code：

```java
// org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecMiniBatchAssigner#translateToPlanInternal
protected Transformation<RowData> translateToPlanInternal(PlannerBase planner) {
    final Transformation<RowData> inputTransform =
            (Transformation<RowData>) getInputEdges().get(0).translateToPlan(planner);

    final OneInputStreamOperator<RowData, RowData> operator;
    if (miniBatchInterval.getMode() == MiniBatchMode.ProcTime) {
        // 处理时间语义，本案例是 ProcTime
        operator = new ProcTimeMiniBatchAssignerOperator(miniBatchInterval.getInterval());
    } else if (miniBatchInterval.getMode() == MiniBatchMode.RowTime) {
        // 事件时间语义
        operator = new RowTimeMiniBatchAssginerOperator(miniBatchInterval.getInterval());
    } else {
        throw new TableException(
                String.format(
                        "MiniBatchAssigner shouldn't be in %s mode this is a bug, please file an issue.",
                        miniBatchInterval.getMode()));
    }
    return new OneInputTransformation<>(
            inputTransform,
            getDescription(),
            operator,
            InternalTypeInfo.of(getOutputType()),
            inputTransform.getParallelism());
}

// org.apache.flink.table.runtime.operators.wmassigners.ProcTimeMiniBatchAssignerOperator
@Override
public void processElement(StreamRecord<RowData> element) throws Exception {
    long now = getProcessingTimeService().getCurrentProcessingTime();
    long currentBatch = now - now % intervalMs;
    if (currentBatch > currentWatermark) {
        currentWatermark = currentBatch;
        // emit
        output.emitWatermark(new Watermark(currentBatch));
    }
    output.collect(element);
}

@Override
public void onProcessingTime(long timestamp) throws Exception {
    long now = getProcessingTimeService().getCurrentProcessingTime();
    long currentBatch = now - now % intervalMs;
    if (currentBatch > currentWatermark) {
        currentWatermark = currentBatch;
        // emit
        output.emitWatermark(new Watermark(currentBatch));
    }
    getProcessingTimeService().registerTimer(currentBatch + intervalMs, this);
}
```

`intervalMs` 就往下游发送 watermark，下游收到 watermark 就会触发 `finishBundle` 计算。

watermark 是不区分处理时间、事件时间，只是一般情况下我们用 watermark 是为了处理乱序的数据，所以都是使用事件时间。

```java
// org.apache.flink.table.runtime.operators.wmassigners.RowTimeMiniBatchAssginerOperator
@Override
public void processWatermark(Watermark mark) throws Exception {
    // if we receive a Long.MAX_VALUE watermark we forward it since it is used
    // to signal the end of input and to not block watermark progress downstream
    if (mark.getTimestamp() == Long.MAX_VALUE && currentWatermark != Long.MAX_VALUE) {
        currentWatermark = Long.MAX_VALUE;
        output.emitWatermark(mark);
        return;
    }

    currentWatermark = Math.max(currentWatermark, mark.getTimestamp());
    if (currentWatermark >= nextWatermark) {
        advanceWatermark();
    }
}

private void advanceWatermark() {
    output.emitWatermark(new Watermark(currentWatermark));
    long start = getMiniBatchStart(currentWatermark, minibatchInterval);
    long end = start + minibatchInterval - 1;
    nextWatermark = end > currentWatermark ? end : end + minibatchInterval;
}
```

逻辑类似，上游传输的 watermark >= 当前的watermark + 时间间隔，就往下游发送新的 watermark。



