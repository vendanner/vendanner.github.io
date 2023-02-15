---
layout:     post
title:      【译】Query Engines：Push vs. Pull
subtitle:   
date:       2023-02-13
author:     danner
header-img: img/bg.jpg
catalog: true
tags:
    - db
    - query
---

原文地址：https://justinjaffray.com/query-engines-push-vs.-pull/

人们经常谈论基于`“拉”`和`“推”`的查询引擎，虽然从名词看很容易理解各自的含义，但有些细节可能有点难以理解。有人已经对两者的区别深思熟虑，Snowflake’s Sigmod paper :

>  基于推送的执行是指关系运算符将其结果推送给下游运算符，而不是等待下游运算符拉取数据（经典的Volcano模式）。基于推送的执行提高了缓存效率，因为它消除循环中的控制流逻辑。它还使Snowflake 能够高效地处理DAG形状的计划，而不仅仅是树，也为中间结果的共享和管道化创造了额外的机会。

上面就是Snowflake 想表达的，但我有两个问题：

- 为什么基于推送的系统“使Snowflake能够高效地处理DAG形状的计划”，而这种方式不受基于拉的系统支持？（DAG代表有向非循环图。）
- 为什么这会提高缓存效率，“消除循环中的控制流逻辑” 意味着什么？

在这篇文章中，我们将讨论基于拉和推查询引擎之间的一些哲学差异，然后我们试图回答问题：讨论您为什么在实际中可能更喜欢一个而不是另一个。思考下面这个SQL

```sql
SELECT DISTINCT customer_first_name FROM customer WHERE customer_balance > 0
```

查询计划器通常将这样的SQL查询编译成一系列离散运算符

![](https://vendanner.github.io/img/pushpull/distinct.png)

Distinct <br><- Map(customer_first_name)<br><- Select(customer_balance > 0)<br><- customer

在基于拉的系统中，是**消费者驱动**的系统。当被请求时，每个操作符都会生成一行：用户向根节点（Distinct）请求一行，这会使向Map请求一行、继而向Select请求一行等等。

在基于推的系统中，是**生产者驱动**的系统。每一个操作符，当它有一些数据时，都会告诉它的下游操作符。customer 作为这个查询中的基表，会告诉Select 它的所有行，进而告诉Map它的行，等等。

让我们开始构建每种查询引擎的超简单实现。

### A basic pull-based query engine

基于拉的查询引擎通常也被称为使用 `Volcano`/`Iterator` 模型。 这是最古老、最著名的查询执行模型，是以 1994 年对其约定进行标准化的论文命名。首先，我们从关系和将其转换为迭代器的方法开始

```javascript
let customer = [
  { id: 1, firstName: "justin", balance: 10 },
  { id: 2, firstName: "sissel", balance: 0 },
  { id: 3, firstName: "justin", balance: -3 },
  { id: 4, firstName: "smudge", balance: 2 },
  { id: 5, firstName: "smudge", balance: 0 },
];

function* Scan(coll) {
  for (let x of coll) {
    yield x;
  }
}
```

当我们有一个迭代器，我们就可以反复询问它的**下一个**元素。

```javascript
let iterator = Scan(customer);

console.log(iterator.next());
console.log(iterator.next());
console.log(iterator.next());
console.log(iterator.next());
console.log(iterator.next());
console.log(iterator.next());
```

输出

```shell
{ value: { id: 1, firstName: 'justin', balance: 10 }, done: false }
{ value: { id: 2, firstName: 'sissel', balance: 0 }, done: false }
{ value: { id: 3, firstName: 'justin', balance: -3 }, done: false }
{ value: { id: 4, firstName: 'smudge', balance: 2 }, done: false }
{ value: { id: 5, firstName: 'smudge', balance: 0 }, done: false }
{ value: undefined, done: true }
```

然后我们可以创建运算符将迭代器转换为另一种形式

```javascript
function* Select(p, iter) {
  for (let x of iter) {
    if (p(x)) {
      yield x;
    }
  }
}

function* Map(f, iter) {
  for (let x of iter) {
    yield f(x);
  }
}

function* Distinct(iter) {
  let seen = new Set();
  for (let x of iter) {
    if (!seen.has(x)) {
      yield x;
      seen.add(x);
    }
  }
}
```

然后我们可以翻译原始查询

```sql
SELECT DISTINCT customer_first_name FROM customer WHERE customer_balance > 0
```

为

```javascript
console.log([
  ...Distinct(
    Map(
      (c) => c.firstName,
      Select((c) => c.balance > 0, Scan(customer))
    )
  ),
]);
```

正如期望那样输出

```shell
[ 'justin', 'smudge' ]
```

### A basic push-based query engine

基于推的查询引擎，有时被称为 Reactive、Observer、Stream 或callback hell 模型，如您所料，与我们之前的示例类似，但它是**从头开始**的。让我们从定义恰当的 Scan 运算符开始。

```javascript
let customer = [
  { id: 1, firstName: "justin", balance: 10 },
  { id: 2, firstName: "sissel", balance: 0 },
  { id: 3, firstName: "justin", balance: -3 },
  { id: 4, firstName: "smudge", balance: 2 },
  { id: 5, firstName: "smudge", balance: 0 },
];

function Scan(relation, out) {
  for (r of relation) {
    out(r);
  }
}
```

We model “this operator tells a downstream operator” as a closure that it calls.

```javascript
Scan(customer, (r) => console.log("row:", r));
```

输出

```shell
row: { id: 1, firstName: 'justin', balance: 10 }
row: { id: 2, firstName: 'sissel', balance: 0 }
row: { id: 3, firstName: 'justin', balance: -3 }
row: { id: 4, firstName: 'smudge', balance: 2 }
row: { id: 5, firstName: 'smudge', balance: 0 }
```

我们可以类似地定义其余的运算符：

```javascript
function Select(p, out) {
  return (x) => {
    if (p(x)) out(x);
  };
}

function Map(f, out) {
  return (x) => {
    out(f(x));
  };
}

function Distinct(out) {
  let seen = new Set();
  return (x) => {
    if (!seen.has(x)) {
      seen.add(x);
      out(x);
    }
  };
}
```

我们的查询现在写成：

```javascript
let result = [];
Scan(
  customer,
  Select(
    (c) => c.balance > 0,
    Map(
      (c) => c.firstName,
      Distinct((r) => result.push(r))
    )
  )
);

console.log(result);
```

输出

```shell
[ 'justin', 'smudge' ]
```

### Comparison

在基于拉的系统中，operators 空闲，直到有人向他**请求数据**。这意味着如何从系统中获取结果是显而易见的：**你向它请求一行，它就会给你**。然而，这也意味着系统的行为与其消费者紧密**耦合**； 如果被请求，你会工作，否则不会。

在基于推的系统中，系统处于空闲状态，直到有人告诉它**有数据**。 因此，系统正在做的工作和它的消费者是**分离**的。

您可能已经注意到，与我们基于拉的系统相比，基于推的系统中必须通过创建**缓冲区**（结果），并指示查询将其结果推送到其中。这就是推系统的运作方式，它们与指定的消费者无关，它们只是存在，并在发生事件时做出响应。

### DAG, yo

让我们回到第一个主要问题：

> 为什么基于推送的系统“使Snowflake能够高效地处理DAG形状的计划”，而这种方式不受基于拉的系统支持？

“DAG-shaped plans”指的是将数据输出到多个下游操作符。事实证明，即使在SQL的上下文中，这比听起来更有用，我们通常认为SQL具有树形结构。

SQL 有一个名为 WITH 的结构，它允许用户在查询中多次引用一个结果集。 这意味着以下查询是有效的 SQL。

```sql
WITH foo as (<some complex query>)
SELECT * FROM
    (SELECT * FROM foo WHERE c) AS foo1
  JOIN
    foo AS foo2
  ON foo1.a = foo2.b
```

查询计划看起来像这样

![](https://vendanner.github.io/img/pushpull/withqueryplan.png)

除了这个明确的例子之外，聪明的查询规划器通常可以利用DAG-ness 来重复使用结果。例如，Jamie Brandon 在他的[文章](https://www.scattered-thoughts.net/writing/materialize-decorrelation)中描述了一种在SQL中解耦子查询的通用方法，这种方法利用DAG 查询计划来提高效率。因为这些原因，能够处理这些情况而不仅仅是复制计划树的一个分支是很有价值的。

在拉模型中有两个主要因素使这(重用)变得困难：调度和生命周期。

#### Scheduling

每个Operator 只有一个输出时，Operator 何时运行并产生输出是显而易见：当您的消费者需要它时。 但对于多个输出，这变得更加混乱，因为“请求行”和“生成行的计算”不再是一对一的(多对一)。

相比之下，在推的系统中，操作符的调度从一开始就没有与它们的输出相关联，因此失去这些信息并没有什么区别(没理解)。

#### Lifetime

在基于拉的模型中，DAG 的另一个棘手问题是，此类系统中的运算符受其**下游运算符的支配**：将来可能被其任何消费者读取的行必须保留（或可以重新计算）。 一个通用的解决方案是让操作员**缓冲其所有获得输出的行**，以便您可以重新分发它们，但是在每个操作员边界引入潜在的无界缓冲是不可取的（但这是必要的，Postgres 和 CockroachDB 所做的 因为 WITH 有多个消费者）。

在推的系统中，这不是太大的问题，因为Operator 驱动(push)消费者(下游Operator)处理一行时，可以有效地强制他们拥有一行并处理它。在大多数情况下，这要么会导致某种必要的缓冲（即使在没有DAG的情况下也是如此，例如Distinct或散列连接操作 -> 收集全部数据才能处理），要么会**立即处理**并传递。

### Cache Efficiency

现在回答第二个问题

> 为什么这会提高缓存效率，“消除循环中的控制流逻辑” 意味着什么？

首先，Snowflake 论文引用了 Thomas Neumann 的一篇[论文](https://www.vldb.org/pvldb/vol4/p539-neumann.pdf)来支持这一说法。 我真的不认为这篇论文孤立地支持这一主张，如果我必须总结这篇论文，它更像是，“我们希望将**查询编译为机器代码以提高缓存效率**，为此， 基于推送的范式更可取。” 这篇论文非常有趣，我建议你读一读，但在我看来，它的结论并不真正适用，除非你想要使用 LLVM 之类的东西编译你的查询（它来自一些粗略的研究，我不清楚 Snowflake 是否这样做）。

在为本节做研究时，我发现了 Shaikhha、Dashti 和 Koch 的这篇[论文](https://arxiv.org/pdf/1610.09166.pdf)，这篇论文很好地突出了每个模型的一些优点和缺点。 事实上，他们参考了 Neumann 论文

> 近期的研究提出了一种Operator Chain 模型，可以不需要保存中间结果，但它颠倒了控制流; 元组从源关系向前推送到生成最终结果的操作符。最近的论文似乎表明，这种推模型始终比拉模型具有更好的查询处理性能，尽管没有提供直接的公平比较。
>
> 本文的主要贡献之一是揭穿这个神话(Push 很好)。 正如我们所展示的，如果公平比较，基于推和拉的引擎具有非常相似的性能，各有优缺点，而且都不是明显的赢家。 推引擎本质上只在查询编译的上下文中被考虑，将推范例的潜在优势与代码内联的优势混为一谈。 为了公平地比较它们，必须将这些方面分离。

他们得出结论，这里没有明显的赢家，但观察到编译基于推的查询可以简化代码。 主要思想是事实证明，将同步的、基于推送的查询展开为手写的等效代码实际上非常容易。看之前的查询

```javascript
let result = [];
Scan(
  customer,
  Select(
    (c) => c.balance > 0,
    Map(
      (c) => c.firstName,
      Distinct((r) => result.push(r))
    )
  )
);

console.log(result);
```

很容易将其编写为以下代码

```java
let result = [];
let seen = new Set();
for (let c of customer) {
  if (c.balance > 0) {
    if (!seen.has(c.firstName)) {
      seen.add(c.firstName);
      result.push(c.firstName);
    }
  }
}

console.log(result);
```

如果您试图展开等效的拉模型查询，您会发现生成的代码要不太自然。

我认为很难基于此得出关于哪个“更好”的任何真正结论，我认为最明智的做法是根据任何特定查询引擎的需求做出选择。

### Considerations

#### Impedance Mismatch

这些系统可能遇到的一个问题是它们之间的边界不匹配。从拉系统到推系统的边界需要轮询其状态(拉系统由下游触发，推系统由上游触发，拉->推直连导致都不会触发，应该要加个中间层轮询拉系统输出是否有变化)，

而从推系统到拉系统的边界需要物化其状态(推的输出数据要保存，供拉来获取)。这两者都不是致命问题，但都会产生一些成本。

这就是为什么在像 Flink 或 Materialise 这样的**流系统**中，您通常会看到使用基于推的系统：此类系统的输入本质上是基于推的，因为要监听 Kafka 流或类似的东西。

在流式设置中，如果您希望您的最终消费者实际上能够以基于**拉**的方式与系统交互（例如，在需要时通过对其**查询**），您需要引入某种物化层，根据结果建立索引。

相反，在一个不提供某种流式/追踪机制的系统中，如果您想知道某些数据何时发生了更改，唯一的选择就是定期轮询它。

译注：

- Kafka -> Flink => pull -> push，Flink-Kafka-Connector 实现就是定时去拉取Kafka 数据进入Flink 开始 Push模式
- Flink -> database => push -> pull，Flink 处理结果物化(存储)到db，等待拉取(查询)

#### Algorithms

有些算法根本**不适合**在推系统中使用。 正如 Shaikhha 论文中所讨论的：`merge join` 算法的工作原理基本上是基于同步遍历两个迭代器的能力，这在消费者几乎没有控制权的推送系统中是不切实际的(中间状态要多次使用)。

同样，`LIMIT` 运算符在推送模型中**可能会出现问题**。 如果不引入双向通信，或将 `LIMIT` 融合到底层运算符（这并不总是可能的），生产运算符无法知道消费者何时满意，他们何时可以停止工作(上游工作状态依赖于下游)。 在拉式系统中，这不是问题，因为消费者可以在不再需要时停止要求更多结果。

#### Cycles

不详细展开，无论是DAGs 还是循环图，在这两个模型中都是复杂的，但解决这个问题最著名的系统是[Naiad](https://users.soe.ucsc.edu/~abadi/Papers/naiad_final.pdf)，它是一个Timely Dataflow系统，其现代版本是[Timely Dataflow](https://github.com/timelydataflow/timely-dataflow)。这两个系统都是推系统，与DAGs一样，许多东西在这里的推模型中效果更好。

### Conclusion

绝大多数介绍数据库的资料都关注**迭代器模型**，但现代系统，**尤其是分析型系统，开始更多地探索推模型**。 正如 Shaikhha 论文中所指出的，很难找到同类比较，因为很多迁移到推模型的动机是希望将**查询编译为较低级别的代码**并从中获得的好处，这使结果(推拉模型哪个更好)变得模糊不清。

尽管如此，还是存在一些数量上的差异，这使得每个模型都适用于不同的场景，如果您对数据库感兴趣，那么对它们的工作原理有一个大概的了解是值得的。 将来我想更详细地介绍这些系统是如何构建的，并尝试揭示使它们工作的一些魔力。

## 参考资料

[How Materialize and other databases optimize SQL subqueries](https://www.scattered-thoughts.net/writing/materialize-decorrelation)

[Efficiently Compiling Efficient Query Plans For Modern Hardware](https://www.vldb.org/pvldb/vol4/p539-neumann.pdf)

[Push vs. Pull-Based Loop Fusion in Query Engines](https://arxiv.org/pdf/1610.09166.pdf)

[Naiad: A Timely Dataflow System](https://users.soe.ucsc.edu/~abadi/Papers/naiad_final.pdf)