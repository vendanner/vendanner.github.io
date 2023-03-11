---
layout:     post
title:      StarRocks Optimizer：Extract Best Plan
subtitle:
date:       2023-03-11
author:     danner
header-img: img/bg.jpg
catalog: true
tags:
    - StarRocks
    - OLAP
    - Optimizer
---

在[memo](https://vendanner.github.io/2023/02/20/StarRocks-Optimizer-Memo/) 中，遍历出所有的Physical Expression 以及Cost，本节基于此**找出最优**(Cost 最低)的Physical Plan。

`memoOptimize` 执行后，`memo`中Physical Expression 如下图所示

![](https://vendanner.github.io/img/StarRocks/memo_plan.png)

- `lowestCostExpressions` : 左侧灰色方框
  - 代表每一个 Group 中，满足 **Required Property** 下的最佳 Expression，并记录相应的 `Cost`
  - Map<PhysicalPropertySet, Pair<Double, GroupExpression>>
- `Physical Expression` : 黄色背景图
- `lowestCostTable`：Physical Expression 下方的表格
  - 代表每一个 GroupExpression 中，满足 **Required Property** 的节点，它的**子节点**需要满足的 **Required Properties**以及Cost
  - Map<PhysicalPropertySet, Pair<Double, List<PhysicalPropertySet>>>  => list 是可能有多个input
- 每个Physical Expression 名字中斜杠后面代表`input group`

注：

- EnforceOperator 也是一个Physical Expression，只是没保存在 group的`physicalExpressions` 
- **EnforceOperator 的input group 是其自身所在的group**(符合搜索最优解代码逻辑)

### extractBestPlan

```java
// com.starrocks.sql.optimizer.Optimizer
// Extract the lowest cost physical operator tree from memo
private OptExpression extractBestPlan(PhysicalPropertySet requiredProperty,
                                    Group rootGroup) {
  // 获取group中满足requiredProperty 最低Cost的 groupExpression
  GroupExpression groupExpression = rootGroup.getBestExpression(requiredProperty);
  Preconditions.checkNotNull(groupExpression, "no executable plan for this sql");
  // 获取groupExpression 的input group 要满足的 requiredProperty
  List<PhysicalPropertySet> inputProperties = groupExpression.getInputProperties(requiredProperty);

  // 递归往下查找
  List<OptExpression> childPlans = Lists.newArrayList();
  for (int i = 0; i < groupExpression.arity(); ++i) {
      OptExpression childPlan = extractBestPlan(inputProperties.get(i), groupExpression.inputAt(i));
      childPlans.add(childPlan);
  }

  OptExpression expression = OptExpression.create(groupExpression.getOp(),
          childPlans);
  // record inputProperties at optExpression, used for planFragment builder to determine join type
  expression.setRequiredProperties(inputProperties);
  expression.setStatistics(groupExpression.getGroup().hasConfidenceStatistic(requiredProperty) ?
          groupExpression.getGroup().getConfidenceStatistic(requiredProperty) :
          groupExpression.getGroup().getStatistics());
  expression.setCost(groupExpression.getCost(requiredProperty));

  // When build plan fragment, we need the output column of logical property
  expression.setLogicalProperty(rootGroup.getLogicalProperty());
  return expression;
}
```

逻辑不复杂

- 使用`lowestCostExpressions` 找到当前group 满足`requiredProperty`最优的groupExpression
- 使用 `lowestCostTable` 得到input group 需要满足的`requiredProperty`
- 递归重复操作

最优Pyhsical plan 如下

![](https://vendanner.github.io/img/StarRocks/best_plan.png)



