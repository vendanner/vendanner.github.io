---
layout:     post
title:      Flink 启动流程之 flink run
subtitle:   
date:       2020-07-02
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - bigdata
---

Flink 1.10

`Flink run` 将我们编译好的代码，提交到集群运行

- Start-cluster 执行是启动 `Standalone` 集群， 任务在 `Standalone` 下运行
- yarn-session 执行是在 `Yarn` 集群启动长服务，任务在 `Flink session cluster` 下运行
- 事先无执行任何脚本直接执行 `flink run` ，任务在 `Flink per-job cluster` 下执行

```shell
# flink 
exec $JAVA_RUN $JVM_ARGS "${log_setting[@]}" -classpath "`manglePathList "$CC_CLASSPATH:$INTERNAL_HADOOP_CLASSPATHS"`" org.apache.flink.client.cli.CliFrontend "$@"
```

```java
// org.apache.flink.client.cli.CliFrontend
// Implementation of a simple command line frontend for executing programs.
/**
* Submits the job based on the arguments.
*/
public static void main(final String[] args) {
  EnvironmentInformation.logEnvironmentInfo(LOG, "Command Line Client", args);

  // 1. find the configuration directory
  final String configurationDirectory = getConfigurationDirectoryFromEnv();

  // 2. load the global configuration
  final Configuration configuration = GlobalConfiguration.loadConfiguration(configurationDirectory);

  // 3. load the custom command lines
  final List<CustomCommandLine> customCommandLines = loadCustomCommandLines(
    configuration,
    configurationDirectory);

  try {
    final CliFrontend cli = new CliFrontend(
      configuration,
      customCommandLines);
    SecurityUtils.install(new SecurityConfiguration(cli.configuration));
    int retCode = SecurityUtils.getInstalledContext()
      .runSecured(() -> cli.parseParameters(args));
    System.exit(retCode);
  }
  ...
}
```

都是老套路：加载配置，获取 `Context` ，我们关注下 ` loadCustomCommandLines`

```java
public static List<CustomCommandLine> loadCustomCommandLines(Configuration configuration, String configurationDirectory) {
  List<CustomCommandLine> customCommandLines = new ArrayList<>();
  // FlinkYarnSessionCli 熟悉嘛？就是 yarn-session 
  final String flinkYarnSessionCLI = "org.apache.flink.yarn.cli.FlinkYarnSessionCli";
  try {
    customCommandLines.add(
      loadCustomCommandLine(flinkYarnSessionCLI,
                            configuration,
                            configurationDirectory,
                            "y",
                            "yarn"));
  } 
  ...
  customCommandLines.add(new ExecutorCLI(configuration));
  //Tips: DefaultCLI must be added at last, because getActiveCustomCommandLine(..) will get the
  // active CustomCommandLine in order and DefaultCLI isActive always return true.
  customCommandLines.add(new DefaultCLI(configuration));

  return customCommandLines;
}
```



```java
public int parseParameters(String[] args) {
  // get action，格式，第一个参数必定是 action
  String action = args[0];

  // remove action from parameters
  final String[] params = Arrays.copyOfRange(args, 1, args.length);
  try {
    // do action
    switch (action) {
      case ACTION_RUN:
        // flink run
        run(params);
        return 0;
      case ACTION_LIST:
        list(params);
        return 0;
      case ACTION_INFO:
        info(params);
        return 0;
      case ACTION_CANCEL:
        cancel(params);
        return 0;
      case ACTION_STOP:
        stop(params);
        return 0;
      case ACTION_SAVEPOINT:
        savepoint(params);
        return 0;
      case "-h":
      case "--help":
        CliFrontendParser.printHelp(customCommandLines);
        return 0;
      case "-v":
      case "--version":
        String version = EnvironmentInformation.getVersion();
        String commitID = EnvironmentInformation.getRevisionInformation().commitId;
        System.out.print("Version: " + version);
        System.out.println(commitID.equals(EnvironmentInformation.UNKNOWN) ? "" : ", Commit ID: " + commitID);
        return 0;
      ...
    
protected void run(String[] args) throws Exception {
    // 解析参数
    final Options commandOptions = CliFrontendParser.getRunCommandOptions();
    final Options commandLineOptions = CliFrontendParser.mergeOptions(commandOptions, customCommandLineOptions);
    final CommandLine commandLine = CliFrontendParser.parse(commandLineOptions, args, true);
    final ProgramOptions programOptions = new ProgramOptions(commandLine);
    ...
    // 封装任务包：jar，mainclass，usejar ...;后面有大用
    final PackagedProgram program;
    try {
        LOG.info("Building program from JAR file");
        program = buildProgram(programOptions);
    }
    ...
    final List<URL> jobJars = program.getJobJarAndDependencies();
    final Configuration effectiveConfiguration =
            getEffectiveConfiguration(commandLine, programOptions, jobJars);

    LOG.debug("Effective executor configuration: {}", effectiveConfiguration);

    try {
        executeProgram(effectiveConfiguration, program);
    } finally {
        program.deleteExtractedLibraries();
    }
 }
// DefaultExecutorServiceLoader
protected void executeProgram(final Configuration configuration, final PackagedProgram program) throws ProgramInvocationException {
    ClientUtils.executeProgram(DefaultExecutorServiceLoader.INSTANCE, configuration, program);
}
```



```java
protected void executeProgram(final Configuration configuration, final PackagedProgram program) throws ProgramInvocationException {
  ClientUtils.executeProgram(DefaultExecutorServiceLoader.INSTANCE, configuration, program);
}

// org.apache.flink.client.ClientUtils
public static void executeProgram(
  PipelineExecutorServiceLoader executorServiceLoader,
  Configuration configuration,
  PackagedProgram program) throws ProgramInvocationException {
  checkNotNull(executorServiceLoader);
  final ClassLoader userCodeClassLoader = program.getUserCodeClassLoader();
  final ClassLoader contextClassLoader = Thread.currentThread().getContextClassLoader();
  try {
    Thread.currentThread().setContextClassLoader(userCodeClassLoader);

    LOG.info("Starting program (detached: {})", !configuration.getBoolean(DeploymentOptions.ATTACHED));

    ContextEnvironmentFactory factory = new ContextEnvironmentFactory(
      executorServiceLoader,
      configuration,
      userCodeClassLoader);
    ContextEnvironment.setAsContext(factory);

    try {
      // 执行用户代码
      program.invokeInteractiveModeForExecution();
    } 
    ...
}
// org.apache.flink.client.program.PackagedProgram
public void invokeInteractiveModeForExecution() throws ProgramInvocationException {
  callMainMethod(mainClass, args);
}
private static void callMainMethod(Class<?> entryClass, String[] args) throws ProgramInvocationException {
    Method mainMethod;
    if (!Modifier.isPublic(entryClass.getModifiers())) {
        throw new ProgramInvocationException("The class " + entryClass.getName() + " must be public.");
    }

    try {
        mainMethod = entryClass.getMethod("main", String[].class);
    } 
    ...
    try {
        mainMethod.invoke(null, (Object) args);
    } 
    ...
}
```

封装 Config，直接跑用户代码，相关介绍看 [深入剖析 Flink Straming WC流程]([https://vendanner.github.io/2020/05/26/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90-Flink-Straming-WC%E6%B5%81%E7%A8%8B/](https://vendanner.github.io/2020/05/26/深入剖析-Flink-Straming-WC流程/))，下节将整个启动流程串起来。



