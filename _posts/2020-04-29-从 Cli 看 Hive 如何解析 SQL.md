---
layout:     post
title:      从 Cli 看 Hive 如何解析 SQL
subtitle:   
date:       2020-04-29
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 大数据
    - SQL
    - Hive
---

Cli 接收用户输入 **SQL**，交给 **Driver** 执行，具体执行流程如下：

``` java
// CliDriver

// 启动
public static void main(String[] args) throws Exception {
  int ret = new CliDriver().run(args);
  System.exit(ret);
}
 public  int run(String[] args) throws Exception {
   ...
   // 创建 SessionState，这一步很重要。接收 System.in
   CliSessionState ss = new CliSessionState(new HiveConf(SessionState.class));
    ss.in = System.in;
   ...
    // execute cli driver work
   return executeDriver(ss, conf, oproc);
 }
private int executeDriver(CliSessionState ss, HiveConf conf, OptionsProcessor oproc)
  throws Exception {
		...
    // 处理 sql
    ret = cli.processLine(line, true);
 		 ...
}
public int processLine(String line, boolean allowInterrupting) {
	...
  ret = processCmd(command);
  ...
}
public int processCmd(String cmd) {
	...
  CommandProcessor proc = CommandProcessorFactory.get(tokens, (HiveConf) conf);
  ret = processLocalCmd(cmd, proc, ss);
  ...
}
int processLocalCmd(String cmd, CommandProcessor proc, CliSessionState ss) {
	...
  if (proc instanceof Driver) {
    Driver qp = (Driver) proc;
    ret = qp.run(cmd).getResponseCode();
}
```

```java
// Driver

public CommandProcessorResponse run(String command, boolean alreadyCompiled)
  throws CommandNeedRetryException {
  CommandProcessorResponse cpr = runInternal(command, alreadyCompiled);
}
private CommandProcessorResponse runInternal(String command, boolean alreadyCompiled)
  throws CommandNeedRetryException {
  // compile internal will automatically reset the perf logger
  ret = compileInternal(command, true);
  ...
  // 执行 sql 
  ret = execute(true);
}
private int compileInternal(String command, boolean deferClose) {
	  ...
	 ret = compile(command, true, deferClose);
  	...
}
// 重点
public int compile(String command, boolean resetTaskIds, boolean deferClose) {
  ...
  ASTNode tree = ParseUtils.parse(command, ctx);
	BaseSemanticAnalyzer sem = SemanticAnalyzerFactory.get(queryState, tree);
  sem.analyze(tree, ctx);
  // sql 语义分析
  // validate the plan
  sem.validate();
  ...
  // get the output schema
  schema = getSchema(sem, conf);
  // 得到查询计划，很重要
  plan = new QueryPlan(queryStr, sem, perfLogger.getStartTime(PerfLogger.DRIVER_RUN), queryId,
                       queryState.getHiveOperation(), schema);
}
// sql tasks
public int execute(boolean deferClose) throws CommandNeedRetryException {
  ...
  // 从查询计划获取 task
  // Add root Tasks to runnable
  for (Task<? extends Serializable> tsk : plan.getRootTasks()) {
    // This should never happen, if it does, it's a bug with the potential to produce
    // incorrect results.
    assert tsk.getParentTasks() == null || tsk.getParentTasks().isEmpty();
    driverCxt.addToRunnable(tsk);

    if (metrics != null) {
      tsk.updateTaskMetrics(metrics);
    }
  }

   // Loop while you either have tasks running, or tasks queued up
  while (driverCxt.isRunning()) {
    // Launch upto maxthreads tasks
    Task<? extends Serializable> task;
    while ((task = driverCxt.getRunnable(maxthreads)) != null) {
      TaskRunner runner = launchTask(task, queryId, noName, jobname, jobs, driverCxt);
      if (!runner.isRunning()) {
        break;
      }
    }
  ...
}
```

总结下：

- 接收用户输入 **SQL**，生成 `CommandProcessor` (Driver)
- Driver  compile 成 QueryPlan
  - ParseUtils 解析 **SQL**  生成**语法树**
  - BaseSemanticAnalyzer 进行**语义分析**
  - 生成 QueryPlan
- 从 QueryPlan 中获取 task 并执行

了解执行步骤，我们就可以做个 `HSQL` 的**语法检查工具**。

``` java
CliSessionState ss = new CliSessionState(hiveConf);
SessionState.start(ss);
Context context = new Context(hiveConf);
ASTNode astNode = ParseUtils.parse(sql, context);
BaseSemanticAnalyzer sem = SemanticAnalyzerFactory.get(queryState, astNode);

sem.analyze(astNode,context);
sem.validate();
```

