---
layout:     post
title:      StarRocks 学习：AST2LogicalPLan
subtitle:
date:       2023-01-11
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - StarRocks
    - OLAP
---

> 从 SQL 文本到分布式物理执行计划, 在 StarRocks 中，需要经过以下 5 个步骤:
>
> 1、SQL Parse： 将 SQL 文本转换成一个 AST（抽象语法树）
>
> 2、SQL Analyze：基于 AST 进行语法和语义分析
>
> 3、SQL Logical Plan： 将 AST 转换成逻辑计划
>
> 4、SQL Optimize：基于关系代数、统计信息、Cost 模型，对逻辑计划进行重写、转换，选择出 Cost “最低” 的物理执行计
>
> 5、生成 Plan Fragment：将 Optimizer 选择的物理执行计划转换为 BE 可以直接执行的 Plan Fragment

本文分析的是步骤三：在StarRocks 中如何将AST转换成LogicalPlan，供后续优化器优化。

StarRocks 源码中有丰富的测试案例，选择案例中的一条SQL来分析：

- 源码目录：com.starrocks.sql.analyzer.AnalyzeSubqueryTest
- SQL：select k from (select v1 as k from t0) a

## AST

本例中的SQL 经Analyze 后生成的AST，如下所示
![](https://vendanner.github.io/img/StarRocks/AST.png)

上图对应下面几个行为

- `TableRelation` : 包含表t0 的基本信息
- `SelectRelation`：
  - **select 内容从哪里来**：relation => TableRelation
  - **select 哪些字段**：selectList
- `SubqueryRelation`: 
  - 别名： alias
  - query： select
- `SelectRelation`: 
  - **select 内容从哪里来**：relation => SubqueryRelation
  - **select 哪些字段**：selectList

通过AST，大致了解本SQL要做哪些事情

1. 获取表数据
2. 表中有三列，但只需要v1 这列即可
3. select 后得到一个子查询，并命名为 "a"
4. 从"a"表 中取 "k" 这列

AST 到这里为止，下面看看LogicalPlan。



## LogicalPlan

```java
com.starrocks.sql.optimizer.transformer.RelationTransformer#transformWithSelectLimit(Relation relation)
```

**transformWithSelectLimit**将AST 转换为LogicalPlan，本例中生成的LogicalPlan 如下所示

![](https://vendanner.github.io/img/logicalPlan.png)

下面分析**transformWithSelectLimit**逻辑

### visit

访问者模式应该并不陌生，它可以将数据结构定义和处理分开。在这里，**RelationTransformer**来就是负责处理，而本例中涉及的数据结构: SelectRelation、SubqueryRelation、TableRelation 都包含`accept` 函数供**RelationTransformer**调用。

LogicalPlan 是从下到上生成的，利用visit 进行递归。

![](https://vendanner.github.io/img/StarRocks/ATS_visit.png)



### 数据结构

在看LogicalPlan 如何生成前，先介绍下OptExprBuilder 和LogicalPlan 结构

#### OptExprBuilder

```java
// com.starrocks.sql.optimizer.transformer.OptExprBuilder
public class OptExprBuilder {
    private final Operator root;
    private final List<OptExprBuilder> inputs;
```

- root：定义当前OptExpression的 Operator
- inputs： 当前OptExpression的输入，可以构造一个OptExpression tree

#### LogicalPlan

```java
// com.starrocks.sql.optimizer.transformer.LogicalPlan
public class LogicalPlan {
    private final OptExprBuilder root;
    private final List<ColumnRefOperator> outputColumn;
```

由此可见，LogicalPlan 只是对OptExprBuilder 包了一层，真正的逻辑在OptExprBuilder。

### TableRelation

```java
// com.starrocks.sql.optimizer.transformer.RelationTransformer#visitTable
 public LogicalPlan visitTable(TableRelation node, ExpressionMapping context) {
    ...
    // 给每个表生成唯一的id
    // 给每个列都标上唯一id
    // 获取TableRelation 的列信息
    Map<Column, ColumnRefOperator> columnMetaToColRefMap = columnMetaToColRefMapBuilder.build();
    List<ColumnRefOperator> outputVariables = outputVariablesBuilder.build();
    LogicalScanOperator scanOperator;
    if (node.getTable().isNativeTable()) {          // 本地表，starrocks也能查外部表(当查询引擎使用，类比presto)
        ...
        // 获取表的分桶bucket 信息 => hashDistributionDesc
        if (node.isMetaQuery()) {
            scanOperator = new LogicalMetaScanOperator(node.getTable(), colRefToColumnMetaMapBuilder.build());
        } else {
            scanOperator = new LogicalOlapScanOperator(node.getTable(),
                    colRefToColumnMetaMapBuilder.build(),
                    columnMetaToColRefMap,
                    DistributionSpec.createHashDistributionSpec(hashDistributionDesc),
                    Operator.DEFAULT_LIMIT,
                    null,
                    ((OlapTable) node.getTable()).getBaseIndexId(),
                    null,
                    node.getPartitionNames(),
                    Lists.newArrayList(),
                    node.getTabletIds());
        }
    } 
   ...
    // 处理外部表的Operator：hive、huid、mysql、es等等
    OptExprBuilder scanBuilder = new OptExprBuilder(scanOperator, Collections.emptyList(),
            new ExpressionMapping(node.getScope(), outputVariables));
    LogicalProjectOperator projectOperator =
            new LogicalProjectOperator(outputVariables.stream().distinct()
                    .collect(Collectors.toMap(Function.identity(), Function.identity())));

    return new LogicalPlan(scanBuilder.withNewRoot(projectOperator), outputVariables, null);
}


// com.starrocks.sql.optimizer.operator.logical.LogicalOlapScanOperator#LogicalOlapScanOperator
public LogicalOlapScanOperator {
    super(OperatorType.LOGICAL_OLAP_SCAN, table, colRefToColumnMetaMap, columnMetaToColRefMap, limit, predicate,
            null);
}
```

生成的Operator 是`LogicalOlapScanOperator`，注意它的OperatorType => LOGICAL_OLAP_SCAN，在后续优化时会用到。值得注意的是，最后生成的LogicalProjectOperator：scan 表后，默认自动获取全部的列。

看最后的LogicalPlan 创建，对应的就是上面LogicalPlan图的最下面一个 OptExprBuilder。

### SelectRelation

```java
// com.starrocks.sql.optimizer.transformer.RelationTransformer#visitSelect
public LogicalPlan visitSelect(SelectRelation node, ExpressionMapping context) {
    return new QueryTransformer(columnRefFactory, session, cteContext).plan(node, outer);
}

// com.starrocks.sql.optimizer.transformer.QueryTransformer#plan
public LogicalPlan plan(SelectRelation queryBlock, ExpressionMapping outer) {
    // builder 就是TableRelation.accept 的返回
    OptExprBuilder builder = planFrom(queryBlock.getRelation(), cteContext);
    builder.setExpressionMapping(new ExpressionMapping(builder.getScope(), builder.getFieldMappings(), outer));
		// 每个操作如果满足，都会生成一个新的OptExprBuilder
    builder = filter(builder, queryBlock.getPredicate());
    builder = aggregate(builder, queryBlock.getGroupBy(), queryBlock.getAggregate(),
            queryBlock.getGroupingSetsList(), queryBlock.getGroupingFunctionCallExprs());
    builder = filter(builder, queryBlock.getHaving());

    List<AnalyticExpr> analyticExprList = new ArrayList<>(queryBlock.getOutputAnalytic());
    analyticExprList.addAll(queryBlock.getOrderByAnalytic());
    builder = window(builder, analyticExprList);
    ...
    builder = distinct(builder, queryBlock.isDistinct(), queryBlock.getOutputExpr());
    // 本例非常简单，只有select 只有project 操作
    builder = project(builder, Iterables.concat(queryBlock.getOrderByExpressions(), queryBlock.getOutputExpr()));
    List<ColumnRefOperator> orderByColumns = Lists.newArrayList();
    builder = sort(builder, queryBlock.getOrderBy(), orderByColumns);
    builder = limit(builder, queryBlock.getLimit());

    List<ColumnRefOperator> outputColumns = computeOutputs(builder, queryBlock.getOutputExpr(), columnRefFactory);

    // Add project operator to prune order by columns
    if (!orderByColumns.isEmpty() && !outputColumns.containsAll(orderByColumns)) {
        long limit = queryBlock.hasLimit() ? queryBlock.getLimit().getLimit() : -1;
        builder = project(builder, queryBlock.getOutputExpr(), limit);
    }

    return new LogicalPlan(builder, outputColumns, correlation);
}
```

plan 中还包含很多操作：filter、aggregate、distinct、sort、limit，每个操作都生成一个新的OptExprBuilder(老的OptExprBuilder 当作input)。`project` 函数做了表达式计算，新建一个LogicalProjectOperator，并将之前的OptExprBuilder 当成input。

### SubqueryRelation

```java
// com.starrocks.sql.optimizer.transformer.RelationTransformer#visitSubquery
public LogicalPlan visitSubquery(SubqueryRelation node, ExpressionMapping context) {
    LogicalPlan logicalPlan = transform(node.getQueryStatement().getQueryRelation());
    OptExprBuilder builder = new OptExprBuilder(
            logicalPlan.getRoot().getOp(),
            logicalPlan.getRootBuilder().getInputs(),
            new ExpressionMapping(node.getScope(), logicalPlan.getOutputColumn()));

    builder = addOrderByLimit(builder, node);
    return new LogicalPlan(builder, logicalPlan.getOutputColumn(), logicalPlan.getCorrelation());
}
```

visitSubquery 没有实质的逻辑(OrderByLimit先放一边，本例不涉及)，只是把下层的LogicalPlan 取出来重新构建一个新的LogicalPlan。



## 转换图

![](https://vendanner.github.io/img/StarRocks/ast2logicalPlan.png)



大声点

## 参考资料

[一条查询SQL的StarRocks之旅](https://zhuanlan.zhihu.com/p/550520456)