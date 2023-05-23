---
layout:     post
title:      StarRocks SQL Binder
subtitle:
date:       2023-02-01
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - StarRocks
    - OLAP
---

上文 [StarRocks 学习：AST2LogicalPLan](https://vendanner.github.io/2023/01/11/StarRocks-%E5%AD%A6%E4%B9%A0-AST2LogicalPLan/) 将SQL 转化成 logicalPlan，但此 logicalPlan 还没有任何意义 - 没有与table、column、function 关联。binder 之后，logicalPlan 包含 `metadata`。

- 源码路径：com.starrocks.sql.analyzer.AnalyzeSubqueryTest#testSimple
- sql

```sql
select * from (select count(v1) from t0) a
```

代码追踪

```java
com.starrocks.sql.StatementPlanner#plan(com.starrocks.sql.ast.StatementBase, com.starrocks.qe.ConnectContext)
  com.starrocks.sql.analyzer.Analyzer#analyze
  	com.starrocks.sql.analyzer.Analyzer.AnalyzerVisitor#visitQueryStatement
      com.starrocks.sql.analyzer.QueryAnalyzer.Visitor#visitQueryStatement
```

### table

```java
// 一、先解析 Relation
// 二、process(Relation, scope);
// 三、最后解析select 字段
@Override
public Scope visitSelect(SelectRelation selectRelation, Scope scope) {
    AnalyzeState analyzeState = new AnalyzeState();
    //Record aliases at this level to prevent alias conflicts
    Set<TableName> aliasSet = new HashSet<>();
    Relation resolvedRelation = resolveTableRef(selectRelation.getRelation(), scope, aliasSet);
    if (resolvedRelation instanceof TableFunctionRelation) {
        throw unsupportedException("Table function must be used with lateral join");
    }
    selectRelation.setRelation(resolvedRelation);
    Scope sourceScope = process(resolvedRelation, scope);    // 新的 scope
    sourceScope.setParent(scope);

    SelectAnalyzer selectAnalyzer = new SelectAnalyzer(session);
    selectAnalyzer.analyze(
            analyzeState,
            selectRelation.getSelectList(),
            selectRelation.getRelation(),
            sourceScope,
            selectRelation.getGroupByClause(),
            selectRelation.getHavingClause(),
            selectRelation.getWhereClause(),
            selectRelation.getOrderBy(),
            selectRelation.getLimit());

    selectRelation.fillResolvedAST(analyzeState);
    return analyzeState.getOutputScope();
}
```

##### binder table

```java
// com.starrocks.sql.analyzer.QueryAnalyzer.Visitor#resolveTableRef
private Relation resolveTableRef(Relation relation, Scope scope, Set<TableName> aliasSet) {
    ...
    } else if (relation instanceof TableRelation) {
        TableRelation tableRelation = (TableRelation) relation;
        TableName tableName = tableRelation.getName();
        ...
        TableName resolveTableName = relation.getResolveTableName();
        // table 填充 catalog、db
        MetaUtils.normalizationTableName(session, resolveTableName);
        if (aliasSet.contains(resolveTableName)) {
            ErrorReport.reportSemanticException(ErrorCode.ERR_NONUNIQ_TABLE,
                    relation.getResolveTableName().getTbl());
        } else {
            aliasSet.add(new TableName(resolveTableName.getCatalog(),
                    resolveTableName.getDb(),
                    resolveTableName.getTbl()));
        }

        // 获取 table metadata
        Table table = resolveTable(tableRelation.getName());
        if (table instanceof View) {
            View view = (View) table;
            QueryStatement queryStatement = view.getQueryStatement();
            ViewRelation viewRelation = new ViewRelation(tableName, view, queryStatement);
            viewRelation.setAlias(tableRelation.getAlias());
            return viewRelation;
        } else {
            if (tableRelation.getTemporalClause() != null) {
                if (table.getType() != Table.TableType.MYSQL) {
                    throw unsupportedException(
                            "unsupported table type for temporal clauses: " + table.getType() +
                                    "; only external MYSQL tables support temporal clauses");
                }
            }

            if (table.isSupported()) {
                // binder table
                tableRelation.setTable(table);
                return tableRelation;
            } else {
                throw unsupportedException("unsupported scan table type: " + table.getType());
            }
        }
    ...
}

private Table resolveTable(TableName tableName) {
    try {
        MetaUtils.normalizationTableName(session, tableName);
        String catalogName = tableName.getCatalog();
        String dbName = tableName.getDb();
        String tbName = tableName.getTbl();
        if (Strings.isNullOrEmpty(dbName)) {
            ErrorReport.reportAnalysisException(ErrorCode.ERR_NO_DB_ERROR);
        }

        if (!GlobalStateMgr.getCurrentState().getCatalogMgr().catalogExists(catalogName)) {
            ErrorReport.reportAnalysisException(ErrorCode.ERR_BAD_CATALOG_ERROR, catalogName);
        }

        Database database = metadataMgr.getDb(catalogName, dbName);
        MetaUtils.checkDbNullAndReport(database, dbName);

        // 根据 (catalogName, dbName, tbName) 从 metadataMgr 获取 Table
        Table table = metadataMgr.getTable(catalogName, dbName, tbName);
        if (table == null) {
            ErrorReport.reportAnalysisException(ErrorCode.ERR_BAD_TABLE_ERROR, dbName + "." + tbName);
        }

        if (table.isNativeTableOrMaterializedView() &&
                (((OlapTable) table).getState() == OlapTable.OlapTableState.RESTORE
                        || ((OlapTable) table).getState() == OlapTable.OlapTableState.RESTORE_WITH_LOAD)) {
            ErrorReport.reportAnalysisException(ErrorCode.ERR_BAD_TABLE_STATE, "RESTORING");
        }
        return table;
    } catch (AnalysisException e) {
        throw new SemanticException(e.getMessage());
    }
}
```

`tableRelation.setTable(table)` 设置之前 table 为null，set 后 `tableRelation` 才真正意义上有语义。

#### get column

`Scope sourceScope = process(resolvedRelation, scope);`  => 当resolvedRelation  = TableRelation 时，获取table columns ，构建新的`scope` 为后续解析准备

```java
// com.starrocks.sql.analyzer.QueryAnalyzer.Visitor#visitTable
public Scope visitTable(TableRelation node, Scope outerScope) {
    TableName tableName = node.getResolveTableName();
    Table table = node.getTable();

    ImmutableList.Builder<Field> fields = ImmutableList.builder();
    ImmutableMap.Builder<Field, Column> columns = ImmutableMap.builder();

    List<Column> fullSchema = node.isBinlogQuery()
            ? appendBinlogMetaColumns(table.getFullSchema()) : table.getFullSchema();
    List<Column> baseSchema = node.isBinlogQuery()
            ? appendBinlogMetaColumns(table.getBaseSchema()) : table.getBaseSchema();
    for (Column column : fullSchema) {
        Field field;
        if (baseSchema.contains(column)) {
            field = new Field(column.getName(), column.getType(), tableName,
                    new SlotRef(tableName, column.getName(), column.getName()), true, column.isAllowNull());
        } else {
            field = new Field(column.getName(), column.getType(), tableName,
                    new SlotRef(tableName, column.getName(), column.getName()), false, column.isAllowNull());
        }
        columns.put(field, column);
        fields.add(field);
    }
    // set columns
    node.setColumns(columns.build());
    String dbName = node.getName().getDb();

    session.getDumpInfo().addTable(dbName, table);
    ...
    // 构建新的 scope
    Scope scope = new Scope(RelationId.of(node), new RelationFields(fields.build()));
    node.setScope(scope);
    return scope;
}
```

### function

`table` 解析并绑定后，开始解析 `select 片段`

```shell
selectAnalyzer.analyze
sql 片段：select count(v1)
```

代码追踪

```java
com.starrocks.sql.analyzer.SelectAnalyzer#analyze
com.starrocks.sql.analyzer.SelectAnalyzer#analyzeSelect
com.starrocks.sql.analyzer.SelectAnalyzer#analyzeExpression
com.starrocks.sql.analyzer.ExpressionAnalyzer#bottomUpAnalyze
private void bottomUpAnalyze(Visitor visitor, Expr expression, Scope scope) {
    if (expression.hasLambdaFunction(expression)) {
        analyzeHighOrderFunction(visitor, expression, scope);
    } else {
        for (Expr expr : expression.getChildren()) {
            // children 可能就是slotRef => 字段
            bottomUpAnalyze(visitor, expr, scope);
        }
    }
    visitor.visit(expression, scope);
}
```

当前案例中

-  `expression` => `FunctionCallExpr{count(v1)}`
- getChildren => `SlofRef`，解析并绑定列字段v1，下小节分析

继续看 function 时如何绑定的，以内置函数为例

```java
// com.starrocks.sql.analyzer.ExpressionAnalyzer.Visitor#visitFunctionCall
public Void visitFunctionCall(FunctionCallExpr node, Scope scope) {
    Type[] argumentTypes = node.getChildren().stream().map(Expr::getType).toArray(Type[]::new);

    if (node.isNondeterministicBuiltinFnName()) {
        ExprId exprId = analyzeState.getNextNondeterministicId();
        node.setNondeterministicId(exprId);
    }

    Function fn;
    String fnName = node.getFnName().getFunction();
    ...
        // 内置 function 查找
        fn = Expr.getBuiltinFunction(fnName, argumentTypes, Function.CompareMode.IS_NONSTRICT_SUPERTYPE_OF);
    }
    ...
    // 校验函数参数类型
    for (int i = 0; i < fn.getNumArgs(); i++) {
        if (!argumentTypes[i].matchesType(fn.getArgs()[i]) &&
                !Type.canCastToAsFunctionParameter(argumentTypes[i], fn.getArgs()[i])) {
            String msg = String.format("No matching function with signature: %s(%s)", fnName,
                    node.getParams().isStar() ? "*" :
                            Arrays.stream(argumentTypes).map(Type::toSql).collect(Collectors.joining(", ")));
            throw new SemanticException(msg, node.getPos());
        }
    }

    if (fn.hasVarArgs()) {
        Type varType = fn.getArgs()[fn.getNumArgs() - 1];
        for (int i = fn.getNumArgs(); i < argumentTypes.length; i++) {
            if (!argumentTypes[i].matchesType(varType) &&
                    !Type.canCastToAsFunctionParameter(argumentTypes[i], varType)) {
                String msg = String.format("Variadic function %s(%s) can't support type: %s", fnName,
                        Arrays.stream(fn.getArgs()).map(Type::toSql).collect(Collectors.joining(", ")),
                        argumentTypes[i]);
                throw new SemanticException(msg, node.getPos());
            }
        }
    }
    // binder function 及返回值
    node.setFn(fn);
    node.setType(fn.getReturnType());
    FunctionAnalyzer.analyze(node);
    return null;
}
public static Function getBuiltinFunction(String name, Type[] argTypes, Function.CompareMode mode) {
    FunctionName fnName = new FunctionName(name);
    Function searchDesc = new Function(fnName, argTypes, Type.INVALID, false);
    return GlobalStateMgr.getCurrentState().getFunction(searchDesc, mode);
}
public Function getFunction(Function desc, Function.CompareMode mode) {
    return functionSet.getFunction(desc, mode);
}
```

`com.starrocks.catalog.FunctionSet` 中函数是如何生成的，查询[技术内幕｜StarRocks 标量函数与聚合函数](https://mp.weixin.qq.com/s/T8K6ci_sx51oGn8H5fSKTg)

若根据**函数名**和**参数类型**没有找到对应的函数，报异常

```java
// com.starrocks.sql.analyzer.ExpressionAnalyzer.Visitor#visitFunctionCall
if (fn == null) {
    String msg = String.format("No matching function with signature: %s(%s)",
            fnName,
            node.getParams().isStar() ? "*" : Joiner.on(", ")
                    .join(Arrays.stream(argumentTypes).map(Type::toSql).collect(Collectors.toList())));
    throw new SemanticException(msg, node.getPos());
}
```

### column

在binder function 过程中，需要先解析 Children； `count(v1)` 中，`v1` 就是 Children。

 ```java
 // com.starrocks.sql.analyzer.ExpressionAnalyzer.Visitor#visitSlot
 public Void visitSlot(SlotRef node, Scope scope) {
     // 从scope 中获取 field，在解析table 时，已将fileds 设置进 scope => Scope scope = new Scope(RelationId.of(node), new RelationFields(fields.build()));
     ResolvedField resolvedField = scope.resolveField(node);
     // binder column
     node.setType(resolvedField.getField().getType());
     node.setTblName(resolvedField.getField().getRelationAlias());
     // help to get nullable info in Analyzer phase
     // now it is used in creating mv to decide nullable of fields
     node.setNullable(resolvedField.getField().isNullable());
     ...
     handleResolvedField(node, resolvedField);
     return null;
 }
 ```

未绑定之前，`SlofRef` 只有col、label，没有任何含义，binder 赋予 `type`、`tableName`、`Nullable` 属性值，变得有含义。



## 参考资料

[StarRocks Analyzer 源码解析](https://zhuanlan.zhihu.com/p/575973738)