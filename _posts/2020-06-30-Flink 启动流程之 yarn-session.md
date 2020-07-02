---
layout:     post
title:      Flink 启动流程之 yarn-session
subtitle:   
date:       2020-06-30
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - bigdata
---

之前介绍的 `start-cluster` 启动的是 Flink `Standalone` 模式，接下来看 `Flink On Yarn` 是如何启动。

### yarn-session

`Flink On Yarn`  的 `Session` 模式，运行一个[长服务](https://ci.apache.org/projects/flink/flink-docs-master/ops/deployment/yarn_setup.html#flink-yarn-session)，适用于运行时间短的任务。

```shell
# yarn-session.sh
bin=`dirname "$0"`
bin=`cd "$bin"; pwd`

# get Flink config
. "$bin"/config.sh

if [ "$FLINK_IDENT_STRING" = "" ]; then
        FLINK_IDENT_STRING="$USER"
fi

JVM_ARGS="$JVM_ARGS -Xmx512m"

CC_CLASSPATH=`manglePathList $(constructFlinkClassPath):$INTERNAL_HADOOP_CLASSPATHS`

log=$FLINK_LOG_DIR/flink-$FLINK_IDENT_STRING-yarn-session-$HOSTNAME.log
log_setting="-Dlog.file="$log" -Dlog4j.configuration=file:"$FLINK_CONF_DIR"/log4j-yarn-session.properties -Dlogback.configurationFile=file:"$FLINK_CONF_DIR"/logback-yarn.xml"

$JAVA_RUN $JVM_ARGS -classpath "$CC_CLASSPATH" $log_setting org.apache.flink.yarn.cli.FlinkYarnSessionCli -j "$FLINK_LIB_DIR"/flink-dist*.jar "$@"
```

流程很简单，直接去执行 `org.apache.flink.yarn.cli.FlinkYarnSessionCli`

```java
// org.apache.flink.yarn.cli.FlinkYarnSessionCli
public static void main(final String[] args) {
  // flink conf 目录载入配置:ip,port,jmHeap,tmHeap,solts,parllelism ...
  final String configurationDirectory = CliFrontend.getConfigurationDirectoryFromEnv();
  final Configuration flinkConfiguration = GlobalConfiguration.loadConfiguration();

  int retCode;

  try {
    final FlinkYarnSessionCli cli = new FlinkYarnSessionCli(
      flinkConfiguration,
      configurationDirectory,
      "",
      ""); // no prefix for the YARN session

    SecurityUtils.install(new SecurityConfiguration(flinkConfiguration));
    retCode = SecurityUtils.getInstalledContext().runSecured(() -> cli.run(args));
   ...
  System.exit(retCode);
}
```

看到上面的代码肯定很熟悉：获取 `Hadoop Context`，在其上执行真正操作。

```java
// org.apache.flink.yarn.cli.FlinkYarnSessionCli
public int run(String[] args) throws CliArgsException, FlinkException {
  final CommandLine cmd = parseCommandLineOptions(args, true);
  ...
  // 设置资源参数
  final Configuration configuration = applyCommandLineOptionsToConfiguration(cmd);
  final ClusterClientFactory<ApplicationId> yarnClusterClientFactory = clusterClientServiceLoader.getClusterClientFactory(configuration);
  // YarnClusterDescriptor 很重要，Flink On Yarn 的信息
  final YarnClusterDescriptor yarnClusterDescriptor = (YarnClusterDescriptor) yarnClusterClientFactory.createClusterDescriptor(configuration);

  try {
    // 可以查询集群，输出Memory，vCores，HealthReport
    if (cmd.hasOption(query.getOpt())) {
      final String description = yarnClusterDescriptor.getClusterDescription();
      System.out.println(description);
      return 0;
    } else {
      final ClusterClientProvider<ApplicationId> clusterClientProvider;
      final ApplicationId yarnApplicationId;
      
      if (cmd.hasOption(applicationId.getOpt())) {
        // 设置固定 appid，并获取 clusterClient
        yarnApplicationId = ConverterUtils.toApplicationId(cmd.getOptionValue(applicationId.getOpt()));
        clusterClientProvider = yarnClusterDescriptor.retrieve(yarnApplicationId);
      } else {
        final ClusterSpecification clusterSpecification = yarnClusterClientFactory.getClusterSpecification(configuration);

        clusterClientProvider = yarnClusterDescriptor.deploySessionCluster(clusterSpecification);
        // ClusterClient：封装了将程序提交到远程集群所需的功能，很重要
        ClusterClient<ApplicationId> clusterClient = clusterClientProvider.getClusterClient();
        //------------------ ClusterClient deployed, handle connection details
        yarnApplicationId = clusterClient.getClusterId();
        // web UI
        try {
          System.out.println("JobManager Web Interface: " + clusterClient.getWebInterfaceURL());
         // yarnApplicationId 记录到本地文件，一般为 /tmp/.yarn-properties-user
         // contnt: applicationID=application_1593474472940_0001
          // flink run 启动时会根据此 id 来提交
          writeYarnPropertiesFile(
            yarnApplicationId,
            dynamicPropertiesEncoded);
        } 
      ...
        ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor();

        final YarnApplicationStatusMonitor yarnApplicationStatusMonitor = new YarnApplicationStatusMonitor(
          yarnClusterDescriptor.getYarnClient(),
          yarnApplicationId,
          new ScheduledExecutorServiceAdapter(scheduledExecutorService));
        Thread shutdownHook = ShutdownHookUtil.addShutdownHook(
          () -> shutdownCluster(
            clusterClientProvider.getClusterClient(),
            scheduledExecutorService,
            yarnApplicationStatusMonitor),
          getClass().getSimpleName(),
          LOG);
        try {
          runInteractiveCli(
            yarnApplicationStatusMonitor,
            acceptInteractiveInput);
        } 
      ...
  return 0;
}
```

#### YarnClusterDescriptor

>The descriptor with deployment information for deploying a Flink cluster on Yarn

```java
clusterClientServiceLoader = DefaultClusterClientServiceLoader
  final ClusterClientFactory<ApplicationId> yarnClusterClientFactory = clusterClientServiceLoader.getClusterClientFactory(configuration);

final YarnClusterDescriptor yarnClusterDescriptor = (YarnClusterDescriptor) yarnClusterClientFactory.createClusterDescriptor(configuration);

// org.apache.flink.client.deployment.DefaultClusterClientServiceLoader
// 加载所有 ClusterClientFactory 类
private static final ServiceLoader<ClusterClientFactory> defaultLoader = ServiceLoader.load(ClusterClientFactory.class);
	@Override
	public <ClusterID> ClusterClientFactory<ClusterID> getClusterClientFactory(final Configuration configuration) {
		checkNotNull(configuration);

	final List<ClusterClientFactory> compatibleFactories = new ArrayList<>();
	final Iterator<ClusterClientFactory> factories = defaultLoader.iterator();
		while (factories.hasNext()) {
			try {
				final ClusterClientFactory factory = factories.next();
        // 比对当前的 ClusterClientFactory name 和configuration 中 target 是否相同
				if (factory != null && factory.isCompatibleWith(configuration)) {
					compatibleFactories.add(factory);
				}
			} catch (Throwable e) {
				if (e.getCause() instanceof NoClassDefFoundError) {
					LOG.info("Could not load factory due to missing dependencies.");
				} else {
					throw e;
				}
			}
		}

		if (compatibleFactories.size() > 1) {
			...
			throw new IllegalStateException("Multiple compatible client factories found for:\n" + String.join("\n", configStr) + ".");
		}
		return compatibleFactories.isEmpty() ? null : (ClusterClientFactory<ClusterID>) compatibleFactories.get(0);
	}
// org.apache.flink.yarn.YarnClusterClientFactory
@Override
public YarnClusterDescriptor createClusterDescriptor(Configuration configuration) {
  checkNotNull(configuration);
  return getClusterDescriptor(configuration);
}
private YarnClusterDescriptor getClusterDescriptor(Configuration configuration) {
  final YarnClient yarnClient = YarnClient.createYarnClient();
  final YarnConfiguration yarnConfiguration = new YarnConfiguration();
  yarnClient.init(yarnConfiguration);
  yarnClient.start();
  return new YarnClusterDescriptor(
    configuration,
    yarnConfiguration,
    yarnClient,
    YarnClientYarnClusterInformationRetriever.create(yarnClient),
    false);
}
```

根据 Job target 创建对应集群的 `ClusterClientFactory`，然后通过 `config`、`yarnconfig`、`yarnclient` 构建 `YarnClusterDescriptor 。`

####  ClusterClient

WebUI 服务的本地代理(IP:Port 与JM 相同)，可以通过 REST API 提交 `jar` 运行任务

```java
// 获取 flink config.masterMemoryMB,taskManagerMemoryMB,slotsPerTaskManager
final ClusterSpecification clusterSpecification = yarnClusterClientFactory.getClusterSpecification(configuration);
clusterClientProvider = yarnClusterDescriptor.deploySessionCluster(clusterSpecification);
ClusterClient<ApplicationId> clusterClient = clusterClientProvider.getClusterClient();

// org.apache.flink.yarn.YarnClusterDescriptor
public ClusterClientProvider<ApplicationId> deploySessionCluster(ClusterSpecification clusterSpecification) throws ClusterDeploymentException {
  try {
    return deployInternal(
      clusterSpecification,
      "Flink session cluster",
      getYarnSessionClusterEntrypoint(),
      null,
      false);
  } catch (Exception e) {
    throw new ClusterDeploymentException("Couldn't deploy Yarn session cluster", e);
  }
}
private ClusterClientProvider<ApplicationId> deployInternal(
			ClusterSpecification clusterSpecification,
			String applicationName,
			String yarnClusterEntrypoint,
			@Nullable JobGraph jobGraph,
			boolean detached) throws Exception {

		if (UserGroupInformation.isSecurityEnabled()) {
			// note: UGI::hasKerberosCredentials inaccurately reports false
			// for logins based on a keytab (fixed in Hadoop 2.6.1, see HADOOP-10786),
			// so we check only in ticket cache scenario.
			boolean useTicketCache = flinkConfiguration.getBoolean(SecurityOptions.KERBEROS_LOGIN_USETICKETCACHE);

			UserGroupInformation loginUser = UserGroupInformation.getCurrentUser();
			if (loginUser.getAuthenticationMethod() == UserGroupInformation.AuthenticationMethod.KERBEROS
					&& useTicketCache && !loginUser.hasKerberosCredentials()) {
				LOG.error("Hadoop security with Kerberos is enabled but the login user does not have Kerberos credentials");
				throw new RuntimeException("Hadoop security with Kerberos is enabled but the login user " +
						"does not have Kerberos credentials");
			}
		}
		isReadyForDeployment(clusterSpecification);
		// ------------------ Check if the specified queue exists --------------------
		checkYarnQueues(yarnClient);
		// ------------------ Check if the YARN ClusterClient has the requested resources --------------
		// Create application via yarnClient
		final YarnClientApplication yarnApplication = yarnClient.createApplication();
		final GetNewApplicationResponse appResponse = yarnApplication.getNewApplicationResponse();

		Resource maxRes = appResponse.getMaximumResourceCapability();

		final ClusterResourceDescription freeClusterMem;
		try {
      // 获取集群空闲资源
			freeClusterMem = getCurrentFreeClusterResources(yarnClient);
		} catch (YarnException | IOException e) {
			failSessionDuringDeployment(yarnClient, yarnApplication);
			throw new YarnDeploymentException("Could not retrieve information about free cluster resources.", e);
		}
		final int yarnMinAllocationMB = yarnConfiguration.getInt(YarnConfiguration.RM_SCHEDULER_MINIMUM_ALLOCATION_MB, 0);

		final ClusterSpecification validClusterSpecification;
		try {
      // 检查资源是否满足
			validClusterSpecification = validateClusterResources(
					clusterSpecification,
					yarnMinAllocationMB,
					maxRes,
					freeClusterMem);
		} catch (YarnDeploymentException yde) {
			failSessionDuringDeployment(yarnClient, yarnApplication);
			throw yde;
		}
		LOG.info("Cluster specification: {}", validClusterSpecification);
		final ClusterEntrypoint.ExecutionMode executionMode = detached ?
				ClusterEntrypoint.ExecutionMode.DETACHED
				: ClusterEntrypoint.ExecutionMode.NORMAL;
		flinkConfiguration.setString(ClusterEntrypoint.EXECUTION_MODE, executionMode.toString());
   // 通过 YARN 将封装好的 ApplicationSubmissionContext 提交到 yarn，返回 report
   // yarnClusterEntrypoint 就是yarn 集群入口点，类似之前分析的 StandaloneSessionClusterEntrypoint
		ApplicationReport report = startAppMaster(
				flinkConfiguration,
				applicationName,
				yarnClusterEntrypoint,
				jobGraph,
				yarnClient,
				yarnApplication,
				validClusterSpecification);
		// print the application id for user to cancel themselves.
		if (detached) {
			final ApplicationId yarnApplicationId = report.getApplicationId();
			logDetachedClusterInformation(yarnApplicationId, LOG);
		}
		setClusterEntrypointInfoToConfig(report);
		return () -> {
			try {
				return new RestClusterClient<>(flinkConfiguration, report.getApplicationId());
			} catch (Exception e) {
				throw new RuntimeException("Error while creating RestClusterClient.", e);
			}
		};
	}

/**
 * A {@link ClusterClient} implementation that communicates via HTTP REST requests.
 * org.apache.flink.client.program.rest.RestClusterClient
 */
```

#### runInteractiveCli

命令行的一些交互，主要是在命令行输出任务的一些状态

#### 启动日志

```shell
2020-06-30 07:48:10,167 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.rpc.address, localhost
2020-06-30 07:48:10,169 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.rpc.port, 6123
2020-06-30 07:48:10,169 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.heap.size, 1024m
2020-06-30 07:48:10,169 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: taskmanager.memory.process.size, 1568m
2020-06-30 07:48:10,169 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: taskmanager.numberOfTaskSlots, 1
2020-06-30 07:48:10,169 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: parallelism.default, 1
2020-06-30 07:48:10,170 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.execution.failover-strategy, region
2020-06-30 07:48:10,804 WARN  org.apache.hadoop.util.NativeCodeLoader                       - Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
2020-06-30 07:48:10,933 INFO  org.apache.flink.runtime.security.modules.HadoopModule        - Hadoop user set to danner (auth:SIMPLE)
2020-06-30 07:48:10,975 INFO  org.apache.flink.runtime.security.modules.JaasModule          - Jaas file will be created as /var/folders/rq/kmxm07hx54jbnhfvg93gv62c0000gn/T/jaas-8888642517590934443.conf.
2020-06-30 07:48:10,997 WARN  org.apache.flink.yarn.cli.FlinkYarnSessionCli                 - The configuration directory ('/Users/luyongtao/app/flink-1.10.0/conf') already contains a LOG4J config file.If you want to use logback, then please delete or rename the log configuration file.
2020-06-30 07:48:11,115 INFO  org.apache.hadoop.yarn.client.RMProxy                         - Connecting to ResourceManager at /0.0.0.0:8032
2020-06-30 07:48:11,354 INFO  org.apache.flink.runtime.clusterframework.TaskExecutorProcessUtils  - The derived from fraction jvm overhead memory (156.800mb (164416719 bytes)) is less than its min value 192.000mb (201326592 bytes), min value will be used instead
2020-06-30 07:48:11,610 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Cluster specification: ClusterSpecification{masterMemoryMB=1024, taskManagerMemoryMB=1568, slotsPerTaskManager=1}
2020-06-30 07:48:14,570 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Submitting application master application_1593474472940_0001
2020-06-30 07:48:14,740 INFO  org.apache.hadoop.yarn.client.api.impl.YarnClientImpl         - Submitted application application_1593474472940_0001
2020-06-30 07:48:14,740 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Waiting for the cluster to be allocated
2020-06-30 07:48:14,743 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Deploying cluster, current state ACCEPTED
2020-06-30 07:48:22,574 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - YARN application has been deployed successfully.
2020-06-30 07:48:22,575 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Found Web Interface localhost:54279 of application 'application_1593474472940_0001'.
JobManager Web Interface: http://localhost:54279
```

#### 总结

获取 `yarnClusterDescriptor` 在 `YARN` 集群部署 `JobManager` 。

