---
layout:     post
title:      StarRocks Optimizer：RuleRewrite
subtitle:
date:       2023-02-10
author:     danner
header-img: img/bg.jpg
catalog: true
tags:
    - StarRocks
    - OLAP
    - Optimizer
---

根据规则对逻辑计划优化重写，案例分析：

- 源码目录：com.starrocks.sql.optimizer.OptimizerTaskTest#testFilterPushDownRule
- SQL：

```sql
select t1
from (
  select * 
  from (
    select t1, t2, t3 from table
  ) where t2 = 1
)
```

### 代码结构

#### logicalRuleRewrite

本函数中包含一些列的`RuleSet`，如果满足条件就会对逻辑计划重写。

```java
// com.starrocks.sql.optimizer.Optimizer#logicalRuleRewrite
ruleRewriteIterative(tree, rootTaskContext, RuleSetType.AGGREGATE_REWRITE);
ruleRewriteIterative(tree, rootTaskContext, RuleSetType.PUSH_DOWN_SUBQUERY);
ruleRewriteIterative(tree, rootTaskContext, RuleSetType.SUBQUERY_REWRITE_COMMON);
ruleRewriteIterative(tree, rootTaskContext, RuleSetType.SUBQUERY_REWRITE_TO_WINDOW);
ruleRewriteIterative(tree, rootTaskContext, RuleSetType.SUBQUERY_REWRITE_TO_JOIN);
ruleRewriteOnlyOnce(tree, rootTaskContext, new ApplyExceptionRule());
ruleRewriteIterative(tree, rootTaskContext, RuleSetType.PUSH_DOWN_PREDICATE);
ruleRewriteIterative(tree, rootTaskContext, new MergeTwoProjectRule());
ruleRewriteOnlyOnce(tree, rootTaskContext, new PushDownAggToMetaScanRule());
ruleRewriteOnlyOnce(tree, rootTaskContext, new PushDownPredicateRankingWindowRule());
ruleRewriteOnlyOnce(tree, rootTaskContext, new PushDownJoinOnExpressionToChildProject());
ruleRewriteOnlyOnce(tree, rootTaskContext, RuleSetType.PRUNE_COLUMNS);
......
```

- ruleRewriteIterative：一直迭代，直到当前逻辑计划不会变化(不再重写)
- ruleRewriteOnlyOnce： 只会调用一次，无论是否会优化重写

#### SeriallyTaskScheduler

每个`ruleRewrite`都是将task 包装成 `RewriteTreeTask` 供SeriallyTaskScheduler 调度执行(这里没看到啥调度? 看起来是单线程顺序执行 => 真正起作用是在`memo` `task`调度)。调试时需要注意的是`NEW_PLANNER_OPTIMIZER_TIMEOUT` 参数：整个Optimizer 过程不能超过这个时间，否则报错；调试时将此参数调大。

```java
// com.starrocks.sql.optimizer.task.SeriallyTaskScheduler#executeTasks
public void executeTasks(TaskContext context) {
    long timeout = context.getOptimizerContext().getSessionVariable().getOptimizerExecuteTimeout();
    Stopwatch watch = context.getOptimizerContext().getTraceInfo().getStopwatch();
    while (!tasks.empty()) {
        if (timeout > 0 && watch.elapsed(TimeUnit.MILLISECONDS) > timeout) {
            // Should have at least one valid plan
            // group will be null when in rewrite phase
            Group group = context.getOptimizerContext().getMemo().getRootGroup();
            if (group == null || !group.hasBestExpression(context.getRequiredProperty())) {
                throw new StarRocksPlannerException("StarRocks planner use long time " + timeout +
                        " ms in " + (group == null ? "logical" : "memo") + " phase, This probably because " +
                        "1. FE Full GC, " +
                        "2. Hive external table fetch metadata took a long time, " +
                        "3. The SQL is very complex. " +
                        "You could " +
                        "1. adjust FE JVM config, " +
                        "2. try query again, " +
                        "3. enlarge new_planner_optimize_timeout session variable",
                        ErrorType.INTERNAL_ERROR);
            }
            break;
        }
        OptimizerTask task = tasks.pop();
        context.getOptimizerContext().setTaskContext(context);
        // 任务开始执行
        task.execute();
    }
}
```

#### RewriteTreeTask

Rewrite 核心逻辑，包含本次任务的逻辑计划、一系列规则(相同规则类型)、是否只优化一次标识

```java
// com.starrocks.sql.optimizer.task.RewriteTreeTask
public void execute() {
    // first node must be RewriteAnchorNode
    rewrite(planTree, 0, planTree.getInputs().get(0));

    if (change > 0 && !onlyOnce) {
        // change => 从上到下Rewrite，规则有匹配到的次数
        // 若change条件不满足，表示上次Rewrite 未进行修改，那么无需再Rewrite
        pushTask(new RewriteTreeTask(context, planTree, rules, onlyOnce));
    }
}

private boolean match(Pattern pattern, OptExpression root) {
    // 当前node match
    if (!pattern.matchWithoutChild(root)) {
        return false;
    }

    int patternIndex = 0;
    int childIndex = 0;

    // 检查OptExpression 输入节点和 Pattern children 匹配
    while (patternIndex < pattern.children().size() && childIndex < root.getInputs().size()) {
        OptExpression child = root.getInputs().get(childIndex);
        Pattern childPattern = pattern.childAt(patternIndex);

        if (!match(childPattern, child)) {
            return false;
        }

        if (!(childPattern.isPatternMultiLeaf() && (root.getInputs().size() - childIndex) >
                (pattern.children().size() - patternIndex))) {
            patternIndex++;
        }

        childIndex++;
    }
    return true;
}

private void rewrite(OptExpression parent, int childIndex, OptExpression root) {
    SessionVariable sessionVariable = context.getOptimizerContext().getSessionVariable();

    for (Rule rule : rules) {
        if (!match(rule.getPattern(), root) || !rule.check(root, context.getOptimizerContext())) {
            continue;
        }
        // 当前表达式及其input node 与当前Rule 匹配后，优化重写
        List<OptExpression> result = rule.transform(root, context.getOptimizerContext());
        Preconditions.checkState(result.size() <= 1, "Rewrite rule should provide at most 1 expression");

        OptimizerTraceInfo traceInfo = context.getOptimizerContext().getTraceInfo();
        OptimizerTraceUtil.logApplyRule(sessionVariable, traceInfo, rule, root, result);

        if (result.isEmpty()) {
            continue;
        }

        // 优化后的OptExpression 设置为 input
        parent.getInputs().set(childIndex, result.get(0));
        // 优化后的OptExpression设为root(已重写)，继续rule 匹配
        root = result.get(0);
        // 执行到这里，说明规则是匹配 OptExpression，计数++
        change++;
        deriveLogicalProperty(root);
    }

    // prune cte column depend on prune right child first
    for (int i = root.getInputs().size() - 1; i >= 0; i--) {
        rewrite(root, i, root.getInputs().get(i));
    }
}
```

- `execute`：只尝试优化一次/一直优化直到没变化
- `match`：每个规则都有个`Pattern`，是否match 有两个条件
  - 当前Node 和 Pattern match
  - 当前Node 的子Node 和Pattern 的子Pattern match

- `rewrite`：
  - 优化路径是**TopDown** -> 先将规则作用root node，然后递归作用于root node 输入节点
  - 若logicalNode 满足当前规则(match)，调用`rule.transform` 重写(这是重点，下文会选几个rule 来分析如何重写)

#### Pattern

```java
// com.starrocks.sql.optimizer.operator.pattern.Pattern
public static Pattern create(OperatorType type, OperatorType... children) {
    Pattern p = new Pattern(type);
    for (OperatorType child : children) {
        p.addChildren(new Pattern(child));
    }
    return p;
}

public boolean matchWithoutChild(OptExpression expression) {
    if (expression == null) {
        return false;
    }

    // 表达式输入node 小于 当前pattern的children个数 且子children 不会去匹配多个node
    // 说明当前表达式与当前Pattern 不匹配
    if (expression.getInputs().size() < this.children().size()
            && children.stream().noneMatch(p -> OperatorType.PATTERN_MULTI_LEAF.equals(p.getOpType()))) {
        return false;
    }

    if (OperatorType.PATTERN_LEAF.equals(getOpType()) || OperatorType.PATTERN_MULTI_LEAF.equals(getOpType())) {
        return true;
    }

    if (isPatternScan() && scanTypes.contains(expression.getOp().getOpType())) {
        return true;
    }
    // Pattern的操作类型与 表达式的操作类型相同
    return getOpType().equals(expression.getOp().getOpType());
}
```

创建Pattern 时，若带多个OperatorType 则会自动创建对应的Children Pattern，这关系到Match 逻辑。

### Rules

前面介绍了代码的执行过程，接下来结合案例SQL 来分析下如何对logical node 重写。这里只输出有明显变化的重写过程，忽略一些不匹配/无重写过程。

```shell
2023-02-10 16:23:58 [main] INFO  OptimizerTraceUtil:138 - [TRACE QUERY null] origin logicOperatorTree:
LogicalProjectOperator {projection=[1: t1]}
->  LogicalFilterOperator {predicate=2: t2 = 1}
    ->  LogicalOlapScanOperator {table=0, selectedPartitionId=null, outputColumns=[1: t1, 2: t2, 3: t3], predicate=null, limit=-1}

# 谓词下推
2023-02-10 16:24:00 [main] INFO  OptimizerTraceUtil:156 - [TRACE QUERY null] APPLY RULE TF_PUSH_DOWN_PREDICATE_SCAN 25
Original Expression:
LogicalFilterOperator {predicate=2: t2 = 1}
->  LogicalOlapScanOperator {table=0, selectedPartitionId=null, outputColumns=[1: t1, 2: t2, 3: t3], predicate=null, limit=-1}
New Expression:
0:
LogicalProjectOperator {projection=[1: t1, 2: t2, 3: t3]}
->  LogicalOlapScanOperator {table=0, selectedPartitionId=null, outputColumns=[1: t1, 2: t2, 3: t3], predicate=2: t2 = 1, limit=-1}

# project 合并
2023-02-10 16:24:00 [main] INFO  OptimizerTraceUtil:156 - [TRACE QUERY null] APPLY RULE TF_MERGE_TWO_PROJECT 63
Original Expression:
LogicalProjectOperator {projection=[1: t1]}
->  LogicalProjectOperator {projection=[1: t1, 2: t2, 3: t3]}
    ->  LogicalOlapScanOperator {table=0, selectedPartitionId=null, outputColumns=[1: t1, 2: t2, 3: t3], predicate=2: t2 = 1, limit=-1}
New Expression:
0:
LogicalProjectOperator {projection=[1: t1]}
->  LogicalOlapScanOperator {table=0, selectedPartitionId=null, outputColumns=[1: t1, 2: t2, 3: t3], predicate=2: t2 = 1, limit=-1}

# olap scan 列裁剪
2023-02-10 16:24:00 [main] INFO  OptimizerTraceUtil:156 - [TRACE QUERY null] APPLY RULE TF_PRUNE_OLAP_SCAN_COLUMNS 45
Original Expression:
LogicalOlapScanOperator {table=0, selectedPartitionId=null, outputColumns=[1: t1, 2: t2, 3: t3], predicate=2: t2 = 1, limit=-1}
New Expression:
0:
LogicalOlapScanOperator {table=0, selectedPartitionId=null, outputColumns=[1: t1, 2: t2], predicate=2: t2 = 1, limit=-1}

# 合并project 及其child
2023-02-10 16:24:00 [main] INFO  OptimizerTraceUtil:156 - [TRACE QUERY null] APPLY RULE TF_MERGE_PROJECT_WITH_CHILD 93
Original Expression:
LogicalProjectOperator {projection=[1: t1]}
->  LogicalOlapScanOperator {table=0, selectedPartitionId=[], outputColumns=[1: t1, 2: t2], predicate=2: t2 = 1, limit=-1}
New Expression:
0:
LogicalOlapScanOperator {table=0, selectedPartitionId=[], outputColumns=[1: t1, 2: t2], predicate=2: t2 = 1, limit=-1}
```

- 谓词下推
- project 合并
- olap scan 列裁剪
- 合并project 及其child

RuleReWrite 后，原本Project + Filter + Scan 变成只需Scan 算子。

#### PushDownPredicateScanRule

```java
// com.starrocks.sql.optimizer.rule.transformation.PushDownPredicateScanRule
public static final PushDownPredicateScanRule OLAP_SCAN =
    new PushDownPredicateScanRule(OperatorType.LOGICAL_OLAP_SCAN);

public PushDownPredicateScanRule(OperatorType type) {
    super(RuleType.TF_PUSH_DOWN_PREDICATE_SCAN, Pattern.create(OperatorType.LOGICAL_FILTER, type));
}

// Filter + Scan 算子优化
// 将Filter 和Scan 的predicate AND 操作 -> newPredicate
// newPredicate 优化 -> rewriteOnlyColumn + DEFAULT_REWRITE_SCAN_PREDICATE_RULES
// 优化后的newPredicate 替换原先Scan 的predicate
@Override
public List<OptExpression> transform(OptExpression input, OptimizerContext context) {
    LogicalFilterOperator lfo = (LogicalFilterOperator) input.getOp();

    OptExpression scan = input.getInputs().get(0);
    LogicalScanOperator logicalScanOperator = (LogicalScanOperator) scan.getOp();

    ScalarOperatorRewriter scalarOperatorRewriter = new ScalarOperatorRewriter();
    // lfo.getPredicate()-> PredicateOp, logicalScanOperator.getPredicate()-> LogicalScanOperator.predicate
    // 组装条件在一个Op 中，操作符是 and
    // where 条件优化
    ScalarOperator predicates = Utils.compoundAnd(lfo.getPredicate(), logicalScanOperator.getPredicate());
    ScalarRangePredicateExtractor rangeExtractor = new ScalarRangePredicateExtractor();
    predicates = rangeExtractor.rewriteOnlyColumn(Utils.compoundAnd(Utils.extractConjuncts(predicates)
            .stream().map(rangeExtractor::rewriteOnlyColumn).collect(Collectors.toList())));
    Preconditions.checkState(predicates != null);
    predicates = scalarOperatorRewriter.rewrite(predicates,
            ScalarOperatorRewriter.DEFAULT_REWRITE_SCAN_PREDICATE_RULES);
    predicates = Utils.transTrue2Null(predicates);

    // clone a new scan operator and rewrite predicate.
    Operator.Builder builder = OperatorBuilderFactory.build(logicalScanOperator);
    LogicalScanOperator newScanOperator = (LogicalScanOperator) builder.withOperator(logicalScanOperator)
            .setPredicate(predicates)
            .build();
    newScanOperator.buildColumnFilters(predicates);
    Map<ColumnRefOperator, ScalarOperator> projectMap =
            newScanOperator.getOutputColumns().stream()
                    .collect(Collectors.toMap(Function.identity(), Function.identity()));
    LogicalProjectOperator logicalProjectOperator = new LogicalProjectOperator(projectMap);
    // filter op直接被删了 -> project + scan(predicate)
    OptExpression project = OptExpression.create(logicalProjectOperator, OptExpression.create(newScanOperator));
    return Lists.newArrayList(project);
}
```

整体逻辑不复杂，主要功能就是把Filter 的`predicate` 塞给 Scan 算子，Filter + scan => project + scan(predicate)

```shell
LogicalFilterOperator {predicate=2: t2 = 1}
->  LogicalOlapScanOperator {table=0, selectedPartitionId=null, outputColumns=[1: t1, 2: t2, 3: t3], predicate=null, limit=-1}
New Expression:
0:
LogicalProjectOperator {projection=[1: t1, 2: t2, 3: t3]}
->  LogicalOlapScanOperator {table=0, selectedPartitionId=null, outputColumns=[1: t1, 2: t2, 3: t3], predicate=2: t2 = 1, limit=-1}
```

Logical node 和Pattern 是否match，答案是显而易见

- 算子
  - LogicalFilterOperator
    - LogicalOlapScanOperator
- Pattern
  - LOGICAL_FILTER
    - LOGICAL_OLAP_SCAN

#### MergeTwoProjectRule

```java
// com.starrocks.sql.optimizer.rule.transformation.MergeTwoProjectRule

// 上下层级相邻的两个Project 行为合并为一个Project，保留上层字段和下层的CallOperator<br>
// LogicalProjectOperator<br>
//  LogicalProjectOperator
// ->
// LogicalProjectOperator
@Override
public List<OptExpression> transform(OptExpression input, OptimizerContext context) {
    LogicalProjectOperator firstProject = (LogicalProjectOperator) input.getOp();
    LogicalProjectOperator secondProject = (LogicalProjectOperator) input.getInputs().get(0).getOp();

    ReplaceColumnRefRewriter rewriter = new ReplaceColumnRefRewriter(secondProject.getColumnRefMap());
    Map<ColumnRefOperator, ScalarOperator> resultMap = Maps.newHashMap();
    for (Map.Entry<ColumnRefOperator, ScalarOperator> entry : firstProject.getColumnRefMap().entrySet()) {
        resultMap.put(entry.getKey(), rewriter.rewrite(entry.getValue()));
    }

    // ASSERT_TRUE must be executed in the runtime, so it should be kept anyway.
    for (Map.Entry<ColumnRefOperator, ScalarOperator> entry : secondProject.getColumnRefMap().entrySet()) {
        if (entry.getValue() instanceof CallOperator) {
            CallOperator callOp = entry.getValue().cast();
            if (FunctionSet.ASSERT_TRUE.equals(callOp.getFnName())) {
                resultMap.put(entry.getKey(), entry.getValue());
            }
        }
    }

    OptExpression optExpression = new OptExpression(
            new LogicalProjectOperator(resultMap, Math.min(firstProject.getLimit(), secondProject.getLimit())));
    optExpression.getInputs().addAll(input.getInputs().get(0).getInputs());
    return Lists.newArrayList(optExpression);
}
```

连续两个project 是没必要的，可以合为一个project。

```shell
LogicalProjectOperator {projection=[1: t1]}
->  LogicalProjectOperator {projection=[1: t1, 2: t2, 3: t3]}
    ->  LogicalOlapScanOperator {table=0, selectedPartitionId=null, outputColumns=[1: t1, 2: t2, 3: t3], predicate=2: t2 = 1, limit=-1}
New Expression:
0:
LogicalProjectOperator {projection=[1: t1]}
->  LogicalOlapScanOperator {table=0, selectedPartitionId=null, outputColumns=[1: t1, 2: t2, 3: t3], predicate=2: t2 = 1, limit=-1}
```

#### PruneScanColumnRule

```java
// com.starrocks.sql.optimizer.rule.transformation.PruneScanColumnRule
//
// scan 时列裁剪，只scan 真正要的列
//      outputColumns = requiredOutputColumns + Predicate
@Override
public List<OptExpression> transform(OptExpression input, OptimizerContext context) {
    LogicalScanOperator scanOperator = (LogicalScanOperator) input.getOp();
    ColumnRefSet requiredOutputColumns = context.getTaskContext().getRequiredColumns();

    // The `outputColumns`s are some columns required but not specified by `requiredOutputColumns`.
    // including columns in predicate or some specialized columns defined by scan operator.
            Set<ColumnRefOperator> outputColumns =
            scanOperator.getColRefToColumnMetaMap().keySet().stream().filter(requiredOutputColumns::contains)
                    .collect(Collectors.toSet());
    outputColumns.addAll(Utils.extractColumnRef(scanOperator.getPredicate()));

    if (outputColumns.size() == 0) {
        outputColumns.add(Utils.findSmallestColumnRef(
                new ArrayList<>(scanOperator.getColRefToColumnMetaMap().keySet())));
    }

    // 已经是最优了，不用再继续裁剪
    if (scanOperator.getColRefToColumnMetaMap().keySet().equals(outputColumns)) {
        return Collections.emptyList();
    } else {
        Map<ColumnRefOperator, Column> newColumnRefMap = outputColumns.stream()
                .collect(Collectors.toMap(identity(), scanOperator.getColRefToColumnMetaMap()::get));
        if (scanOperator instanceof LogicalOlapScanOperator) {
            LogicalOlapScanOperator olapScanOperator = (LogicalOlapScanOperator) scanOperator;
            // 新建scanOP，与之前OP不同之处在于，只scan 需要的列
            LogicalOlapScanOperator newScanOperator = new LogicalOlapScanOperator(
                    olapScanOperator.getTable(),
                    newColumnRefMap,
                    olapScanOperator.getColumnMetaToColRefMap(),
                    olapScanOperator.getDistributionSpec(),
                    olapScanOperator.getLimit(),
                    olapScanOperator.getPredicate(),
                    olapScanOperator.getSelectedIndexId(),
                    olapScanOperator.getSelectedPartitionId(),
                    olapScanOperator.getPartitionNames(),
                    olapScanOperator.getSelectedTabletId(),
                    olapScanOperator.getHintsTabletIds());

            return Lists.newArrayList(new OptExpression(newScanOperator));
        }
        ......
    }
}
```

scan 不需要把所有字段都获取，只获取整个sql 需要的列(优化时收集好的) 和 predicate 列

```shell
LogicalOlapScanOperator {table=0, selectedPartitionId=null, outputColumns=[1: t1, 2: t2, 3: t3], predicate=2: t2 = 1, limit=-1}
New Expression:
0:
LogicalOlapScanOperator {table=0, selectedPartitionId=null, outputColumns=[1: t1, 2: t2], predicate=2: t2 = 1, limit=-1}
```

#### MergeProjectWithChildRule

```java
// com.starrocks.sql.optimizer.rule.transformation.MergeProjectWithChildRule
// 合并Project+Scan => Scan
//  将Project 直接塞进 Scan.setProjection(Project)
@Override
public List<OptExpression> transform(OptExpression input, OptimizerContext context) {
    LogicalProjectOperator logicalProjectOperator = (LogicalProjectOperator) input.getOp();

    if (logicalProjectOperator.getColumnRefMap().isEmpty()) {
        return Lists.newArrayList(input.getInputs().get(0));
    }
    LogicalOperator child = (LogicalOperator) input.inputAt(0).getOp();

    ColumnRefSet projectColumns = logicalProjectOperator.getOutputColumns(
            new ExpressionContext(input));
    ColumnRefSet childOutputColumns = child.getOutputColumns(new ExpressionContext(input.inputAt(0)));
    if (projectColumns.equals(childOutputColumns)) {
        return input.getInputs();
    }

    Operator.Builder builder = OperatorBuilderFactory.build(child);
    builder.withOperator(child).setProjection(new Projection(logicalProjectOperator.getColumnRefMap(),
            Maps.newHashMap()));

    if (logicalProjectOperator.hasLimit()) {
        builder.setLimit(Math.min(logicalProjectOperator.getLimit(), child.getLimit()));
    } else {
        builder.setLimit(child.getLimit());
    }

    return Lists.newArrayList(OptExpression.create(builder.build(), input.inputAt(0).getInputs()));
}
```

最终，sql :

```sql
select t1
from (
  select * 
  from (
    select t1, t2, t3 from table
  ) where t2 = 1
)
```

优化后只剩

```shell
LogicalOlapScanOperator {table=0, selectedPartitionId=[], outputColumns=[1: t1, 2: t2], predicate=2: t2 = 1, limit=-1}
```







