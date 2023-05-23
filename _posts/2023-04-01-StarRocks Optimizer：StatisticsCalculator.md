---
layout:     post
title:      StarRocks Optimizer：StatisticsCalculator
subtitle:
date:       2023-04-01
author:     danner
header-img: img/bg.jpg
catalog: true
tags:
    - StarRocks
    - OLAP
    - Optimizer
---

Cost 里最重要的是**基数估计**：输出多少row，决定 `Cost`。

### Scan

从最基础的scan 开始

如何计算当前scan 会多少数据量？

- 元数据中获取 rowsize
- 谓词裁剪，一般情况不可能是全表扫描
  - 分区，只获取某几个/范围内的分区数据
  - 普通字段过滤

```java
// com.starrocks.sql.optimizer.statistics.StatisticsCalculator#computeOlapScanNode
private Void computeOlapScanNode(Operator node, ExpressionContext context, Table table,
																	Collection<Long> selectedPartitionIds,
																	Map<ColumnRefOperator, Column> colRefToColumnMetaMap) {
		Preconditions.checkState(context.arity() == 0);
		// 1. get table row count => 只获取选中的分区数据
		long tableRowCount = getTableRowCount(table, node);
		// 2. get required columns statistics
		Statistics.Builder builder = estimateScanColumns(table, colRefToColumnMetaMap);
		if (tableRowCount <= 1) {
				builder.setTableRowCountMayInaccurate(true);
		}
		// 3. deal with column statistics for partition prune
		// 分区字段的统计信息需要调整(adjustPartitionStatistic)，因为有分区裁剪的，只要在范围内的分区里找即可
		OlapTable olapTable = (OlapTable) table;
		ColumnStatistic partitionStatistic = adjustPartitionStatistic(selectedPartitionIds, olapTable);
		if (partitionStatistic != null) {
				String partitionColumnName = olapTable.getPartitionColumnNames().get(0);
				Optional<Map.Entry<ColumnRefOperator, Column>> partitionColumnEntry =
								colRefToColumnMetaMap.entrySet().stream().
												filter(column -> column.getValue().getName().equalsIgnoreCase(partitionColumnName))
												.findAny();
				// partition prune maybe because partition has none data
				partitionColumnEntry.ifPresent(entry -> builder.addColumnStatistic(entry.getKey(), partitionStatistic));
		}

		builder.setOutputRowCount(tableRowCount);
		// 4. estimate cardinality
		context.setStatistics(builder.build());
		return visitOperator(node, context);
}
```

- getTableRowCount 已包含分区裁剪，获取到截此为止要 SCAN 的数据量

#### visitOperator

```java
// com.starrocks.sql.optimizer.statistics.StatisticsCalculator#visitOperator
public Void visitOperator(Operator node, ExpressionContext context) {
	ScalarOperator predicate = null;
	long limit = Operator.DEFAULT_LIMIT;

	if (node instanceof LogicalOperator) {
			LogicalOperator logical = (LogicalOperator) node;
			predicate = logical.getPredicate();
			limit = logical.getLimit();
	} else if (node instanceof PhysicalOperator) {
			PhysicalOperator physical = (PhysicalOperator) node;
			predicate = physical.getPredicate();
			limit = physical.getLimit();
	}

	Statistics statistics = context.getStatistics();
	if (null != predicate) {
			// 谓词比较会减少 row count 以及更新predicate 中对应的 ColumnStatistics
			statistics = estimateStatistics(ImmutableList.of(predicate), statistics);
	}

	Statistics.Builder statisticsBuilder = Statistics.buildFrom(statistics);
	if (limit != Operator.DEFAULT_LIMIT && limit < statistics.getOutputRowCount()) {
			statisticsBuilder.setOutputRowCount(limit);
	}
  ......
```

- 谓词下推: `estimateStatistics`
  - `PredicateStatisticsCalculator.statisticsCalculate`
    - `PredicateStatisticsCalculator.BaseCalculatingVisitor#visitBinaryPredicate`
      - `BinaryPredicateStatisticCalculator.estimateColumnToConstantComparison`

以 id_date >= '2021-05-01' 为例(PredicateStatisticsCalculatorTest.testDateBinaryPredicate)

```java
// com.starrocks.sql.optimizer.statistics.BinaryPredicateStatisticCalculator#estimateColumnGreaterThanConstant
// columnRefOperator >= constant(id_date >= '2021-05-01')
// constant 构造成 StatisticRangeValues 去比较
private static Statistics estimateColumnGreaterThanConstant(Optional<ColumnRefOperator> columnRefOperator,
		ColumnStatistic columnStatistic,
		Optional<ConstantOperator> constant,
		Statistics statistics,
		BinaryPredicateOperator.BinaryType binaryType) {
	if (columnStatistic.getHistogram() == null || !constant.isPresent()) {
			StatisticRangeValues predicateRange;
			if (constant.isPresent()) {
					Optional<Double> d = StatisticUtils.convertStatisticsToDouble(
									constant.get().getType(), constant.get().toString());
					if (d.isPresent()) {
							predicateRange = new StatisticRangeValues(d.get(), POSITIVE_INFINITY, NaN);
					} else {
							predicateRange = new StatisticRangeValues(NEGATIVE_INFINITY, POSITIVE_INFINITY, NaN);
					}
			} else {
					predicateRange = new StatisticRangeValues(NEGATIVE_INFINITY, POSITIVE_INFINITY, NaN);
			}
			// 为啥StatisticRangeValues 的high 设置为POSITIVE_INFINITY(无穷大)？
			// 这里 >= 比较，那么常量的这边的取值范围 => (常量，∞)
			return estimatePredicateRange(columnRefOperator, columnStatistic, predicateRange, statistics);
	......
  
 // com.starrocks.sql.optimizer.statistics.BinaryPredicateStatisticCalculator#estimatePredicateRange
 // 如何得到两个Operator 比较后的统计信息？
// 一、取交集 intersectRange
// 二、得到 列选中的因子 predicateFactor = intersectRange/columnRange
// 三、更新row count = 当前row count * predicateFactor
// 四、得到新的 列统计信息 newEstimateColumnStatistics(更新最大、最小值)
public static Statistics estimatePredicateRange(Optional<ColumnRefOperator> columnRefOperator,
																								ColumnStatistic columnStatistic,
																								StatisticRangeValues predicateRange,
																								Statistics statistics) {
		StatisticRangeValues columnRange = StatisticRangeValues.from(columnStatistic);
		StatisticRangeValues intersectRange = columnRange.intersect(predicateRange);

		double predicateFactor = columnRange.overlapPercentWith(intersectRange); // overlapPercentWith => 正常情况(intersectRange/distinct_val)
		double rowCount = statistics.getOutputRowCount() * (1 - columnStatistic.getNullsFraction()) * predicateFactor;

		ColumnStatistic newEstimateColumnStatistics = ColumnStatistic.builder().
						setAverageRowSize(columnStatistic.getAverageRowSize()).
						setMaxValue(intersectRange.getHigh()).
						setMinValue(intersectRange.getLow()).
						setNullsFraction(0).
						setDistinctValuesCount(columnStatistic.getDistinctValuesCount()).
						setType(columnStatistic.getType()).
						build();
		return columnRefOperator.map(operator -> Statistics.buildFrom(statistics).setOutputRowCount(rowCount).
										addColumnStatistic(operator, newEstimateColumnStatistics).build()).
						orElseGet(() -> Statistics.buildFrom(statistics).setOutputRowCount(rowCount).build());
}
```

这里更新的 `rowCount` 是不准确的，这里假设**每个列都不相关**  => 真实世界的数据分布、字段的独立性(不相关->范式？) 很难用模型去拟合。

#### CostEstimator

统计信息计算好了，对`Cost` 是如何影响呢？

```java
// com.starrocks.sql.optimizer.cost.CostModel.CostEstimator#visitPhysicalOlapScan
public CostEstimate visitPhysicalOlapScan(PhysicalOlapScanOperator node, ExpressionContext context) {
		Statistics statistics = context.getStatistics();
		Preconditions.checkNotNull(statistics);
		return CostEstimate.of(statistics.getComputeSize(), 0, 0);
}

// com.starrocks.sql.optimizer.statistics.Statistics#getComputeSize
public double getComputeSize() {
		// Make it at least 1 byte, otherwise the cost model would propagate estimate error
		return Math.max(1.0, this.columnStatistics.values().stream().map(ColumnStatistic::getAverageRowSize).
						reduce(0.0, Double::sum)) * outputRowCount;
}
```

由此可见，对于Scan Cost 最重要的是 row count；由上面代码分析可知，对 row count 影响最大的是 partition keys，当然列裁剪也有影响。 

### Agg

```java
// com.starrocks.sql.optimizer.statistics.StatisticsCalculator#computeAggregateNode
private Void computeAggregateNode(Operator node, ExpressionContext context, List<ColumnRefOperator> groupBys,
																	Map<ColumnRefOperator, CallOperator> aggregations) {
		Preconditions.checkState(context.arity() == 1);
		Statistics.Builder builder = Statistics.builder();
		Statistics inputStatistics = context.getChildStatistics(0);

		//Update the statistics of the GroupBy column
		Map<ColumnRefOperator, ColumnStatistic> groupStatisticsMap = new HashMap<>();
		// GroupBy 计算row count，并从inputStatistics 获取groupby key ColumnStatistic
		double rowCount = computeGroupByStatistics(groupBys, inputStatistics, groupStatisticsMap);

		//Update Node Statistics
		builder.addColumnStatistics(groupStatisticsMap);
		rowCount = min(inputStatistics.getOutputRowCount(), rowCount);
		builder.setOutputRowCount(rowCount);
		// use inputStatistics and aggregateNode cardinality to estimate aggregate call operator column statistics.
		// because of we need cardinality to estimate count function.
		double estimateCount = rowCount;
    // 先不分析，agg cost 只与 row count 相关(当然getAverageRowSize 也有影响，但这个没办法优化)
		aggregations.forEach((key, value) -> builder
						.addColumnStatistic(key,
										ExpressionStatisticCalculator.calculate(value, inputStatistics, estimateCount)));

		context.setStatistics(builder.build());
		return visitOperator(node, context);
}

// com.starrocks.sql.optimizer.statistics.StatisticsCalculator#computeGroupByStatistics
// 计算GroupBy row count
// 1、从inputStatistics 获取groupby keys 的ColumnStatistic
// 2、计算group by的 row count
//    inputStatistics.getOutputRowCount() * 0.5 * 1.05 * 1.05 ...
//    or groupByColumnStatics.getDistinctValuesCount() * Math.max(1, Math.pow(0.75, index))
//    无论哪种方式，计算后的 row count 必须 <= inputStatistics.getOutputRowCount
public static double computeGroupByStatistics(List<ColumnRefOperator> groupBys, Statistics inputStatistics,
																							Map<ColumnRefOperator, ColumnStatistic> groupStatisticsMap) {
		for (ColumnRefOperator groupByColumn : groupBys) {
				ColumnStatistic groupByColumnStatics = inputStatistics.getColumnStatistic(groupByColumn);
				ColumnStatistic.Builder statsBuilder = buildFrom(groupByColumnStatics);
				if (groupByColumnStatics.getNullsFraction() == 0) {
						statsBuilder.setNullsFraction(0);
				} else {
						statsBuilder.setNullsFraction(1 / (groupByColumnStatics.getDistinctValuesCount() + 1));
				}

				groupStatisticsMap.put(groupByColumn, statsBuilder.build());
		}

		//Update the number of output rows of AggregateNode
		double rowCount = 1;
		if (groupStatisticsMap.values().stream().anyMatch(ColumnStatistic::isUnknown)) {
				// estimate with default column statistics
				for (int groupByIndex = 0; groupByIndex < groupBys.size(); ++groupByIndex) {
						if (groupByIndex == 0) {
								rowCount = inputStatistics.getOutputRowCount() *
												StatisticsEstimateCoefficient.DEFAULT_GROUP_BY_CORRELATION_COEFFICIENT;
						} else {
								rowCount *= StatisticsEstimateCoefficient.DEFAULT_GROUP_BY_EXPAND_COEFFICIENT;
								if (rowCount > inputStatistics.getOutputRowCount()) {
										rowCount = inputStatistics.getOutputRowCount();
										break;
								}
						}
				}
		} else {
				for (int groupByIndex = 0; groupByIndex < groupBys.size(); ++groupByIndex) {
						ColumnRefOperator groupByColumn = groupBys.get(groupByIndex);
						ColumnStatistic groupByColumnStatics = inputStatistics.getColumnStatistic(groupByColumn);
						double cardinality = groupByColumnStatics.getDistinctValuesCount() +
										((groupByColumnStatics.getNullsFraction() == 0.0) ? 0 : 1);
						if (groupByIndex == 0) {
								rowCount *= cardinality;
						} else {
								rowCount *= Math.max(1, cardinality * Math.pow(
												StatisticsEstimateCoefficient.UNKNOWN_GROUP_BY_CORRELATION_COEFFICIENT, groupByIndex + 1D));
								if (rowCount > inputStatistics.getOutputRowCount()) {
										rowCount = inputStatistics.getOutputRowCount();
										break;
								}
						}
				}
		}
		return Math.min(Math.max(1, rowCount), inputStatistics.getOutputRowCount());
}
```

statistics 计算逻辑很简单，重要的是几个`系数`的取值

```java
// estimate aggregates row count with default group by columns statistics
public static final double DEFAULT_GROUP_BY_CORRELATION_COEFFICIENT = 0.5;
// expand estimate aggregates row count with default group by columns statistics
public static final double DEFAULT_GROUP_BY_EXPAND_COEFFICIENT = 1.05;
// Group by columns correlation in estimate aggregates row count
public static final double UNKNOWN_GROUP_BY_CORRELATION_COEFFICIENT = 0.75;
```

Agg row count 有两种计算方式

- 存在`ColumnStatistic::isUnknown` ：**与groupby keys 没有任何关系**，只需 `inputStatistics.getOutputRowCount()` 乘以系数即可
-  **与groupby keys 相关**，不考虑列相关性的话，应该是每个groupby key的 `cardinality` 乘积，但为了准确一点，这里带上列相关系数

由此可见，对于agg row count ，重要的是 groupby key 的 `DistinctValuesCount`。

#### CostEstimator

```java
// com.starrocks.sql.optimizer.cost.CostModel.CostEstimator#visitPhysicalHashAggregate
public CostEstimate visitPhysicalHashAggregate(PhysicalHashAggregateOperator node, ExpressionContext context) {
		if (!needGenerateOneStageAggNode(context) && node.getDistinctColumnDataSkew() == null && !node.isSplit() &&
						node.getType().isGlobal()) {
				return CostEstimate.infinite();
		}

		Statistics statistics = context.getStatistics();
		Statistics inputStatistics = context.getChildStatistics(0);
		double penalty = 1.0;
		if (node.getDistinctColumnDataSkew() != null) {
				penalty = computeDataSkewPenaltyOfGroupByCountDistinct(node, inputStatistics);
		}

		return CostEstimate.of(inputStatistics.getComputeSize() * penalty, statistics.getComputeSize() * penalty,
						0);
}
```

不考虑数据倾斜的情况下，Cost 只和 `input row count` 和当前 `row count` (计算方式看上面)相关。

### Join

com.starrocks.sql.optimizer.statistics.StatisticsCalculator#computeJoinNode

- 得到一个笛卡尔积 `crossRowCount` => 左右表的 row count 乘积
- join on 中**等值条件** 裁剪得到新的 `innerJoinStats`

```java
// com.starrocks.sql.optimizer.statistics.StatisticsCalculator#estimateInnerJoinStatistics
public Statistics estimateInnerJoinStatistics(Statistics statistics, List<BinaryPredicateOperator> eqOnPredicates) {
		if (eqOnPredicates.isEmpty()) {
				return statistics;
		}
		if (ConnectContext.get().getSessionVariable().isUseCorrelatedJoinEstimate()) {
				// row count * 1/distinct value * 0.9、更新 column statistics
				return estimatedInnerJoinStatisticsAssumeCorrelated(statistics, eqOnPredicates);
		} else {
				return Statistics.buildFrom(statistics)
								.setOutputRowCount(estimateInnerRowCountMiddleGround(statistics, eqOnPredicates)).build();
		}
}
private Statistics estimatedInnerJoinStatisticsAssumeCorrelated(Statistics statistics,
																																List<BinaryPredicateOperator> eqOnPredicates) {
		Queue<BinaryPredicateOperator> remainingEqOnPredicates = new LinkedList<>(eqOnPredicates);
		BinaryPredicateOperator drivingPredicate = remainingEqOnPredicates.poll();
		Statistics result = statistics;
		for (int i = 0; i < eqOnPredicates.size(); ++i) {
				Statistics estimateStatistics =
								estimateByEqOnPredicates(statistics, drivingPredicate, remainingEqOnPredicates);
				if (estimateStatistics.getOutputRowCount() < result.getOutputRowCount()) {
						result = estimateStatistics;
				}
				remainingEqOnPredicates.add(drivingPredicate);
				drivingPredicate = remainingEqOnPredicates.poll();
		}
		return result;
}
public Statistics estimateByEqOnPredicates(Statistics statistics, BinaryPredicateOperator divingPredicate,
																						Collection<BinaryPredicateOperator> remainingEqOnPredicate) {
		// 谓词下推裁剪，on 条件 => 会更新 row count 以及column statistics
		Statistics estimateStatistics = estimateStatistics(ImmutableList.of(divingPredicate), statistics);
		for (BinaryPredicateOperator ignored : remainingEqOnPredicate) {
				estimateStatistics = estimateByAuxiliaryPredicates(estimateStatistics);
		}
		return estimateStatistics;
}
public Statistics estimateByAuxiliaryPredicates(Statistics estimateStatistics) {
		double rowCount = estimateStatistics.getOutputRowCount() *
						StatisticsEstimateCoefficient.UNKNOWN_AUXILIARY_FILTER_COEFFICIENT;
		return Statistics.buildFrom(estimateStatistics).setOutputRowCount(rowCount).build();
}

// com.starrocks.sql.optimizer.statistics.BinaryPredicateStatisticCalculator#estimateColumnEqualToColumn
// 两个列 Equal 比较 -  基数估计
// 1、更新 row count：row count * selectivity(1/x=DistinctValuesCount) => equal 只有一个值
// 2、新建 column Statistics 最大、小值 => 两个列交集
// 3、更新 Statistics row count 以及column Statistics(步骤2生成的Statistics)
public static Statistics estimateColumnEqualToColumn(ScalarOperator leftColumn,
																											ColumnStatistic leftColumnStatistic,
																											ScalarOperator rightColumn,
																											ColumnStatistic rightColumnStatistic,
																											Statistics statistics,
																											boolean isEqualForNull) {
		// 更新 row count：row count * selectivity(1/x=DistinctValuesCount)																										
		double leftDistinctValuesCount = leftColumnStatistic.getDistinctValuesCount();
		double rightDistinctValuesCount = rightColumnStatistic.getDistinctValuesCount();
		double selectivity = 1.0 / Math.max(1, Math.max(leftDistinctValuesCount, rightDistinctValuesCount));
		double rowCount = statistics.getOutputRowCount() * selectivity *
						(isEqualForNull ? 1 :
										(1 - leftColumnStatistic.getNullsFraction()) * (1 - rightColumnStatistic.getNullsFraction()));
    // 新建 column Statistics 最大、小值 => 两个列交集
		StatisticRangeValues intersect = StatisticRangeValues.from(leftColumnStatistic)
						.intersect(StatisticRangeValues.from(rightColumnStatistic));
		ColumnStatistic.Builder newEstimateColumnStatistics = ColumnStatistic.builder().
						setMaxValue(intersect.getHigh()).
						setMinValue(intersect.getLow()).
						setDistinctValuesCount(intersect.getDistinctValues());

		ColumnStatistic newLeftStatistic;
		ColumnStatistic newRightStatistic;
		if (!isEqualForNull) {
				newEstimateColumnStatistics.setNullsFraction(0);
				newLeftStatistic = newEstimateColumnStatistics
								.setAverageRowSize(leftColumnStatistic.getAverageRowSize()).build();
				newRightStatistic = newEstimateColumnStatistics
								.setAverageRowSize(rightColumnStatistic.getAverageRowSize()).build();
		} else {
				newLeftStatistic = newEstimateColumnStatistics
								.setAverageRowSize(leftColumnStatistic.getAverageRowSize())
								.setNullsFraction(leftColumnStatistic.getNullsFraction())
								.build();
				newRightStatistic = newEstimateColumnStatistics
								.setAverageRowSize(rightColumnStatistic.getAverageRowSize())
								.setNullsFraction(rightColumnStatistic.getNullsFraction())
								.build();
		}
    // 更新 Statistics row count 以及column Statistics(步骤2生成的Statistics)
		Statistics.Builder builder = Statistics.buildFrom(statistics);
		if (leftColumn instanceof ColumnRefOperator) {
				builder.addColumnStatistic((ColumnRefOperator) leftColumn, newLeftStatistic);
		}
		if (rightColumn instanceof ColumnRefOperator) {
				builder.addColumnStatistic((ColumnRefOperator) rightColumn, newRightStatistic);
		}
		builder.setOutputRowCount(rowCount);
		return builder.build();
}
```

- 根据`joinType` 决定是选择**crossJoinStats** 还是**innerJoinStats** 作为后续计算的 Statistics(除了CROSS_JOIN 和 INNER_JOIN，其他jointype 都是选择innerJoinStats)
- join on 中**非等值条件** 继续裁剪

```java
// com.starrocks.sql.optimizer.statistics.BinaryPredicateStatisticCalculator#estimateColumnToColumnComparison
public static Statistics estimateColumnToColumnComparison(ScalarOperator leftColumn,
																													ColumnStatistic leftColumnStatistic,
																													ScalarOperator rightColumn,
																													ColumnStatistic rightColumnStatistic,
																													BinaryPredicateOperator predicate,
																													Statistics statistics) {
		switch (predicate.getBinaryType()) {
				case EQ:
						// 等值判断
						return estimateColumnEqualToColumn(leftColumn, leftColumnStatistic,
										rightColumn, rightColumnStatistic, statistics, false);
				case EQ_FOR_NULL:
						return estimateColumnEqualToColumn(leftColumn, leftColumnStatistic,
										rightColumn, rightColumnStatistic, statistics, true);
				case NE:
						// 不等值判断 != 
						return estimateColumnNotEqualToColumn(leftColumnStatistic, rightColumnStatistic, statistics);
				case LE:
				case GE:
				case LT:
				case GT:
						// 0.5 is unknown filter coefficient
						// 不等值判断 <=、>=、<、>
						double rowCount = statistics.getOutputRowCount() * 0.5;
						return Statistics.buildFrom(statistics).setOutputRowCount(rowCount).build();
				default:
						throw new IllegalArgumentException("unknown binary type: " + predicate.getBinaryType());
		}
}
 
// 不等于条件裁剪 - row count
//   有unknown 列 直接乘系数 0.8
//   否则：row count * (1 - 1/Math.max(leftDistinctValuesCount, rightDistinctValuesCount)) => 类似等值裁剪
public static Statistics estimateColumnNotEqualToColumn(
				ColumnStatistic leftColumn,
				ColumnStatistic rightColumn,
				Statistics statistics) {
		double leftDistinctValuesCount = leftColumn.getDistinctValuesCount();
		double rightDistinctValuesCount = rightColumn.getDistinctValuesCount();
		double selectivity = 1.0 / Math.max(1, Math.max(leftDistinctValuesCount, rightDistinctValuesCount));

		double rowCount = statistics.getOutputRowCount();
		// If any ColumnStatistic is default, give a default selectivity
		if (leftColumn.isUnknown() || rightColumn.isUnknown()) {
				rowCount = rowCount * 0.8;
		} else {
				rowCount = rowCount * (1.0 - selectivity)
								* (1 - leftColumn.getNullsFraction()) * (1 - rightColumn.getNullsFraction());
		}
		return Statistics.buildFrom(statistics).setOutputRowCount(rowCount).build();
}
```

#### CostEstimator

Join 有三种策略：`HashJoin`、`MergeJoin`、`NestLoopJoin`

- `HashJoin`：一侧构建hash map，另一侧遍历从hash map 判断是否有相等值

```java
// com.starrocks.sql.optimizer.cost.HashJoinCostModel#getCpuCost
// 取决于左右侧的 row count 和当前 node 的 row count
// 复杂在于，hash join 不同的执行方式，cost 略有差异
public double getCpuCost() {
		JoinExecMode execMode = deriveJoinExecMode();
		double buildCost;
		double probeCost;
		// leftOutput/rightOutput = outputRowCount * sum(AverageRowSize)
		double leftOutput = leftStatistics.getOutputSize(context.getChildOutputColumns(0));
		double rightOutput = rightStatistics.getOutputSize(context.getChildOutputColumns(1));
		int parallelFactor = Math.max(ConnectContext.get().getAliveBackendNumber(),
						ConnectContext.get().getSessionVariable().getDegreeOfParallelism());
		switch (execMode) {
				case BROADCAST:
						buildCost = rightOutput;
						probeCost = leftOutput * getAvgProbeCost();
						break;
				case SHUFFLE:
						// 一个hash map 被分解乘多个小hash map
						buildCost = rightOutput / parallelFactor;
						probeCost = leftOutput * getAvgProbeCost();
						break;
				default:
						buildCost = rightOutput;
						probeCost = leftOutput;
		}
		double joinCost = buildCost + probeCost;
		// should add output cost
		joinCost += joinStatistics.getComputeSize();
		return joinCost;
}

// mem cost 取决于 右表的大小(build hash map)
public double getMemCost() {
		JoinExecMode execMode = deriveJoinExecMode();
		// rightOutput = outputRowCount * sum(AverageRowSize)
		double rightOutput = rightStatistics.getOutputSize(context.getChildOutputColumns(1));
		double memCost;
		int beNum = Math.max(1, ConnectContext.get().getAliveBackendNumber());

		if (JoinExecMode.BROADCAST == execMode) {
				memCost = rightOutput * beNum;
		} else {
				memCost = rightOutput;
		}
		return memCost;
}
```

- `MergeJoin`：两侧先排序，然后在比较(有序后，不用循环比较，而是移动两侧的指针进行比较)
  - 与当前 row count 无关，只与input row count 相关

```java
// com.starrocks.sql.optimizer.cost.CostModel.CostEstimator#visitPhysicalMergeJoin
public CostEstimate visitPhysicalMergeJoin(PhysicalMergeJoinOperator join, ExpressionContext context) {
    ......
		Statistics leftStatistics = context.getChildStatistics(0);
		Statistics rightStatistics = context.getChildStatistics(1);

		List<BinaryPredicateOperator> eqOnPredicates =
						JoinHelper.getEqualsPredicate(leftStatistics.getUsedColumns(), rightStatistics.getUsedColumns(),
										Utils.extractConjuncts(join.getOnPredicate()));
		if (join.getJoinType().isCrossJoin() || eqOnPredicates.isEmpty()) {
				return CostEstimate.of(leftStatistics.getOutputSize(context.getChildOutputColumns(0))
												+ rightStatistics.getOutputSize(context.getChildOutputColumns(1)),
								rightStatistics.getOutputSize(context.getChildOutputColumns(1))
												* StatisticsEstimateCoefficient.CROSS_JOIN_COST_PENALTY * 100D, 0);
		} else {
				return CostEstimate.of((leftStatistics.getOutputSize(context.getChildOutputColumns(0))
												+ rightStatistics.getOutputSize(context.getChildOutputColumns(1)) / 2),
								0, 0);

		}
}
```

- `NestLoopJoin`：多层for 循环匹配两侧数据是否相等；显而易见，这种方式的 `Cost` 最大

```java
// com.starrocks.sql.optimizer.cost.CostModel.CostEstimator#visitPhysicalNestLoopJoin
public CostEstimate visitPhysicalNestLoopJoin(PhysicalNestLoopJoinOperator join, ExpressionContext context) {
		Statistics leftStatistics = context.getChildStatistics(0);
		Statistics rightStatistics = context.getChildStatistics(1);

		double leftSize = leftStatistics.getOutputSize(context.getChildOutputColumns(0));
		double rightSize = rightStatistics.getOutputSize(context.getChildOutputColumns(1));
		// cpuCost = leftSize * rightSize
		double cpuCost = StatisticUtils.multiplyOutputSize(StatisticUtils.multiplyOutputSize(leftSize, rightSize),
						StatisticsEstimateCoefficient.CROSS_JOIN_COST_PENALTY);
		// memCost = rightSize * 200
		double memCost = StatisticUtils.multiplyOutputSize(rightSize,
						StatisticsEstimateCoefficient.CROSS_JOIN_COST_PENALTY * 100D);

		// Right cross join could not be parallelized, so apply more punishment
		if (join.getJoinType().isRightJoin()) {
				// Add more punishment when right size is 10x greater than left size.
				if (rightSize > 10 * leftSize) {
						cpuCost *= StatisticsEstimateCoefficient.CROSS_JOIN_RIGHT_COST_PENALTY;
				} else {
						cpuCost += StatisticsEstimateCoefficient.CROSS_JOIN_RIGHT_COST_PENALTY;
				}
				memCost += rightSize;
		}
		if (join.getJoinType().isOuterJoin() || join.getJoinType().isSemiJoin() ||
						join.getJoinType().isAntiJoin()) {
				cpuCost += leftSize;
		}

		return CostEstimate.of(cpuCost, memCost, 0);
}
```

根据 `CostModel` 分析

- 一般不选`NestLoopJoin`，因为**Cost** 是乘积，会很大
- 两表数据量都很大时，选中`MergeJoin`，因为大表情况下，`HashJoin`  构建的 hash map 很占内存
- 一般情况下，`HashJoin` 时最优解

