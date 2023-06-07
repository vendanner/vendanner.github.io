---
layout:     post
title:      StarRocks 添加SQL 语法
subtitle:
date:       2023-05-05
author:     danner
header-img: img/bg.jpg
catalog: true
tags:
    - StarRocks
    - Antlr4
---

> 从 SQL 文本到分布式物理执行计划, 在 StarRocks 中，需要经过以下 5 个步骤:
>
> 1、SQL Parse： 将 SQL 文本转换成一个 AST（抽象语法树）
>
> 2、SQL Analyze：基于 AST 进行语法和语义分析  => 本文
>
> 3、SQL Logical Plan： 将 AST 转换成逻辑计划
>
> 4、SQL Optimize：基于关系代数、统计信息、Cost 模型，对逻辑计划进行重写、转换，选择出 Cost “最低” 的物理执行计
>
> 5、生成 Plan Fragment：将 Optimizer 选择的物理执行计划转换为 BE 可以直接执行的 Plan Fragment

之前 [StarRocks 学习：AST2LogicalPLan](https://vendanner.github.io/2023/01/11/StarRocks-%E5%AD%A6%E4%B9%A0-AST2LogicalPLan/)、[StarRocks SQL Binder](https://vendanner.github.io/2023/02/01/StarRocks-SQL-Binder/) 两篇文章大致分析步骤二和三。本文以添加新的窗口聚合函数 `NTH_VALUE` 方式来熟悉 StarRocks Parse，主要逻辑在`StarRocksParser` 、`AstBuilder`。

### Antlr4

本文不重点分析 antlr 语法，只稍微介绍。

语法分析器（parser）是用来识别语言的程序，本身包含两个部分：词法分析器（lexer）和语法分析器（parser）。词法分析阶段主要解决的关键词以及各种标识符，例如 INT、ID 等，语法分析主要是基于词法分析的结果，构造一颗语法分析树。

在 StarRocks 中有两个文件：StarRocks.g4、StarRocksLex.g4

- StarRocksLex.g4：可以简单理解为是**宏定义** - `词法`，在`StarRocks.g4` 会用到
  - INTEGER_VALUE：代表数字
  - SET：代表字符串 'set'
- StarRocks.g4：定义具体的SQL`语法`

```shell
functionCall
    : EXTRACT '(' identifier FROM valueExpression ')'                                     #extract
    ...
    | windowFunction over                                                                 #windowFunctionCall
windowFunction
    : name = ROW_NUMBER '(' ')'
    | name = RANK '(' ')'
    | name = DENSE_RANK '(' ')'
    | name = NTILE  '(' expression? ')'
    | name = LEAD  '(' (expression ignoreNulls? (',' expression)*)? ')' ignoreNulls?
    | name = LAG '(' (expression ignoreNulls? (',' expression)*)? ')' ignoreNulls?
    | name = FIRST_VALUE '(' (expression ignoreNulls? (',' expression)*)? ')' ignoreNulls?
    | name = LAST_VALUE '(' (expression ignoreNulls? (',' expression)*)? ')' ignoreNulls?
    
ignoreNulls
    : IGNORE NULLS
    ;
```

定义完语法，那么窗口函数怎么写呢，以`row_number` 举例

```sql
ROW_NUMBER() over()
```

StarRocks.g4、StarRocksLex.g4 通过`antlr4-maven-plugin` 插件会生成对应的 Listener、Visitor、Parser。其中最重要的是Parser，StarRocks 会生成 `StarRocksParser`。

### StarRocksParser

StarRocksParser 会将输入的 SQL 语句解析成语法分析树，本质是包含各种类型的 `Context`。Astbuilder 将 `Context` 转化成不同的`Node`(这个我们就熟悉了)。

以上面的 `windowFunction` 为例，分析下`StarRocks.g4` 和 `StarRocksParser` 有怎样的关系。

```java
// com.starrocks.sql.parser.StarRocksParser.WindowFunctionContext
public static class WindowFunctionContext extends ParserRuleContext {
	public Token name;
	public TerminalNode ROW_NUMBER() { return getToken(StarRocksParser.ROW_NUMBER, 0); }
	public TerminalNode RANK() { return getToken(StarRocksParser.RANK, 0); }
	public TerminalNode DENSE_RANK() { return getToken(StarRocksParser.DENSE_RANK, 0); }
	public TerminalNode NTILE() { return getToken(StarRocksParser.NTILE, 0); }
	public List<ExpressionContext> expression() {
		return getRuleContexts(ExpressionContext.class);
	}
	public ExpressionContext expression(int i) {
		return getRuleContext(ExpressionContext.class,i);
	}
	public TerminalNode LEAD() { return getToken(StarRocksParser.LEAD, 0); }
	public List<IgnoreNullsContext> ignoreNulls() {
		return getRuleContexts(IgnoreNullsContext.class);
	}
	public IgnoreNullsContext ignoreNulls(int i) {
		return getRuleContext(IgnoreNullsContext.class,i);
	}
	public TerminalNode LAG() { return getToken(StarRocksParser.LAG, 0); }
	public TerminalNode FIRST_VALUE() { return getToken(StarRocksParser.FIRST_VALUE, 0); }
	public TerminalNode LAST_VALUE() { return getToken(StarRocksParser.LAST_VALUE, 0); }
	public TerminalNode INTEGER_VALUE() { return getToken(StarRocksParser.INTEGER_VALUE, 0); }
	public TerminalNode FROM() { return getToken(StarRocksParser.FROM, 0); }
	public TerminalNode FIRST() { return getToken(StarRocksParser.FIRST, 0); }
	public TerminalNode LAST() { return getToken(StarRocksParser.LAST, 0); }
	...
```

- WindowFunctionContext： `StarRocks.g4` 中类似**windowFunction** ，会自动生成 Context；同理可以找到**FunctionCallContext**、**IgnoreNullsContext**
- = : 等号会生成`Token`，`name =` => `public Token name;`
- 词法：StarRocksLex.g4 定义中出现的词法会变成 `TerminalNode`(包含Token)

了解这么多，已足够去添加一个新的语法。

### AstBuilder

同样是 `windowFunction` 为例，如何将Context 转换成Node

```java
// com.starrocks.sql.parser.AstBuilder#visitWindowFunctionCall
public ParseNode visitWindowFunctionCall(StarRocksParser.WindowFunctionCallContext context) {
		FunctionCallExpr functionCallExpr = (FunctionCallExpr) visit(context.windowFunction());
    // 构建 over 语句 node，本文不分析
		return buildOverClause(functionCallExpr, context.over(), createPos(context));
}

@Override
public ParseNode visitWindowFunction(StarRocksParser.WindowFunctionContext context) {
		FunctionCallExpr functionCallExpr = new FunctionCallExpr(context.name.getText().toLowerCase(),
						new FunctionParams(false, visit(context.expression(), Expr.class)), createPos(context));
		boolean ignoreNull = CollectionUtils.isNotEmpty(context.ignoreNulls())
						&& context.ignoreNulls().stream().anyMatch(Objects::nonNull);

		functionCallExpr.setIgnoreNulls(ignoreNull);
		return functionCallExpr;
}
```

重点是`visitWindowFunction` 函数，构造一个 `FunctionCallExpr`，设置相应的属性。

- fnName 属性：Token context.name 获取字符串，得到函数名
- ignoreNulls 属性：TerminalNode context.ignoreNulls() 是否为空
  - 若SQL 语句包含 `IGNORE NULLS` ，那么此TerminalNode 不为空，且会包含两个 Token，分别是 star='IGNORE' 和stop='NULLS'。
  - 如果SQL 语句存在 `IGNORE NIL`，会发生什么？StarRocksParser.parse 中就会报错，因为未定义`IGNORE NIL`语法

再找个例子，判断Token 是否相等，后续添加语法会用到。

#### sortItem

`StarRocks.g4` 

```shell
sortItem
    : expression ordering = (ASC | DESC)? (NULLS nullOrdering=(FIRST | LAST))?
    ;
```

`StarRocksParser`

```java
public static class SortItemContext extends ParserRuleContext {
	public Token ordering;
	public Token nullOrdering;
	public ExpressionContext expression() {
		return getRuleContext(ExpressionContext.class,0);
	}
	public TerminalNode NULLS() { return getToken(StarRocksParser.NULLS, 0); }
	public TerminalNode ASC() { return getToken(StarRocksParser.ASC, 0); }
	public TerminalNode DESC() { return getToken(StarRocksParser.DESC, 0); }
	public TerminalNode FIRST() { return getToken(StarRocksParser.FIRST, 0); }
	public TerminalNode LAST() { return getToken(StarRocksParser.LAST, 0); }
	public SortItemContext(ParserRuleContext parent, int invokingState) {
		super(parent, invokingState);
	}
	...
```

`AstBuilder`

```java
public ParseNode visitSortItem(StarRocksParser.SortItemContext context) {
		return new OrderByElement(
						(Expr) visit(context.expression()),
						getOrderingType(context.ordering),
						getNullOrderingType(getOrderingType(context.ordering), context.nullOrdering),
						createPos(context));
}
private static boolean getOrderingType(Token token) {
		if (token == null) {
				return true;
		}

		return token.getType() == StarRocksLexer.ASC;
}
```

注意`ordering` 在三个文件里的作用

- StarRocks.g4 定义了 ordering 不是ASC ，就是DESC
- StarRocksParser 定义一个Token 名为 ordering
- AstBuilder 展示如何使用 ordering => 如何获取到StarRocks.g4 中定义的一个变量取值

### NTH_VALUE

```sql
NTH_VALUE ( <value expression>, <nth row>)
      [ FROM {FIRST | LAST} ] [{ IGNORE | RESPECT } NULLS ]
      OVER ([PARTITION BY] [ORDER BY])
# 默认是FIRST 和 RESPECT NULLS
```

`OVER` 之后的语法可以不分析，StarRocks.g4 已实现

参考已有的`windowFunction` 语法，不难得到 `NTH_VALUE` 语法

```shell
windowFunction
    : name = ROW_NUMBER '(' ')'
    | name = NTH_VALUE '(' (expression ',' expression) ')' (FROM direction =(FIRST | LAST))? (ignoreNulls | respectNulls)?
    ;

ignoreNulls
    : IGNORE NULLS
    ;
respectNulls
    : RESPECT NULLS
    ;
```

编译后自动 StarRocksParser 文件，我们需要做的是如何在`AstBuilder` 将`NTH_VALUE` 参数信息获取到。

有了上面的知识，依葫芦画瓢。

```java
// com.starrocks.sql.parser.AstBuilder#visitWindowFunction
public ParseNode visitWindowFunction(StarRocksParser.WindowFunctionContext context) {
		FunctionCallExpr functionCallExpr = new FunctionCallExpr(context.name.getText().toLowerCase(),
						new FunctionParams(false, visit(context.expression(), Expr.class)), createPos(context));
		boolean ignoreNull = CollectionUtils.isNotEmpty(context.ignoreNulls())
						&& context.ignoreNulls().stream().anyMatch(Objects::nonNull);

		functionCallExpr.setIgnoreNulls(ignoreNull);
		functionCallExpr.setAsc(getDirectionType(context.direction));
		return functionCallExpr;
}
private static boolean getDirectionType(Token token) {
		if (token == null) {
				return true;
		}

		return token.getType() == StarRocksLexer.FIRST;
}

// com.starrocks.analysis.Expr
private boolean ignoreNulls = false;

// com.starrocks.analysis.FunctionCallExpr
private  boolean isAsc = true;
```

分析下面SQL 如何取参数

```sql
SELECT NTH_VALUE(tc, 1) from last IGNORE NULLS over(partition by ta order by tg) FROM tall
```

- expression：设置到 FunctionParams，本例包含两个 Expr
  - `SlotRef`：tc 列
  - IntLiteral:  int 常量 1
- direction：获取Token context.direction ，比较是否为 First/Last
- ignoreNulls | respectNulls：获取 IgnoreNullsContext context.ignoreNulls() 判断；已有代码，逻辑一致，没有修改





## 参考资料

[StarRocks Parser 源码解析](https://zhuanlan.zhihu.com/p/560685391)

