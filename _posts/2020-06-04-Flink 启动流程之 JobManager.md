---
layout:     post
title:      Flink 启动流程之 JobManager
subtitle:   
date:       2020-06-04
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - bigdata
---

Flink 1.10   

> ```
> StandaloneSessionClusterEntrypoint：Entry point for the standalone session cluster
> ```

```java
// org.apache.flink.runtime.entrypoint.StandaloneSessionClusterEntrypoint
public class StandaloneSessionClusterEntrypoint extends SessionClusterEntrypoint {
	public StandaloneSessionClusterEntrypoint(Configuration configuration) {
		super(configuration);
	}
	@Override
	protected DefaultDispatcherResourceManagerComponentFactory createDispatcherResourceManagerComponentFactory(Configuration configuration) {
		return DefaultDispatcherResourceManagerComponentFactory.createSessionComponentFactory(StandaloneResourceManagerFactory.INSTANCE);
	}
	public static void main(String[] args) {
		// startup checks and logging
		EnvironmentInformation.logEnvironmentInfo(LOG, StandaloneSessionClusterEntrypoint.class.getSimpleName(), args);
		SignalHandler.register(LOG);
		JvmShutdownSafeguard.installAsShutdownHook(LOG);
		EntrypointClusterConfiguration entrypointClusterConfiguration = null;
		final CommandLineParser<EntrypointClusterConfiguration> commandLineParser = new CommandLineParser<>(new EntrypointClusterConfigurationParserFactory());
		try {
      // 解析参数
			entrypointClusterConfiguration = commandLineParser.parse(args);
		} catch (FlinkParseException e) {
			LOG.error("Could not parse command line arguments {}.", args, e);
			commandLineParser.printHelp(StandaloneSessionClusterEntrypoint.class.getSimpleName());
			System.exit(1);
		}
    // 封装成 Flink Config
		Configuration configuration = loadConfiguration(entrypointClusterConfiguration);
		StandaloneSessionClusterEntrypoint entrypoint = new StandaloneSessionClusterEntrypoint(configuration);
    // 启动 StandaloneSession
		ClusterEntrypoint.runClusterEntrypoint(entrypoint);
	}
}
```

- 解析参数

```java
// org.apache.flink.runtime.entrypoint.parser.CommandLineParser
public T parse(@Nonnull String[] args) throws FlinkParseException {
  final DefaultParser parser = new DefaultParser();
  final Options options = parserResultFactory.getOptions();
  final CommandLine commandLine;
  try {
    commandLine = parser.parse(options, args, true);
  } catch (ParseException e) {
    throw new FlinkParseException("Failed to parse the command line arguments.", e);
  }
  return parserResultFactory.createResult(commandLine);
}
// org.apache.flink.runtime.entrypoint.EntrypointClusterConfigurationParserFactory
public EntrypointClusterConfiguration createResult(@Nonnull CommandLine commandLine) {
  // 指定的 flink-conf.yml 路径
  final String configDir = commandLine.getOptionValue(CONFIG_DIR_OPTION.getOpt());
  // -D 设置的 property，例如 -Dlog.file
  final Properties dynamicProperties = commandLine.getOptionProperties(DYNAMIC_PROPERTY_OPTION.getOpt());
  // WEB-UI 端口
  final String restPortStr = commandLine.getOptionValue(REST_PORT_OPTION.getOpt(), "-1");
  final int restPort = Integer.parseInt(restPortStr);
  // WEB-UI host
  final String hostname = commandLine.getOptionValue(HOST_OPTION.getOpt());
  return new EntrypointClusterConfiguration(
    configDir,
    dynamicProperties,
    commandLine.getArgs(),
    hostname,
    restPort);
}
```

- 封装 Flink Config

```java
// org.apache.flink.runtime.entrypoint.ClusterEntrypoint
protected static Configuration loadConfiguration(EntrypointClusterConfiguration entrypointClusterConfiguration) {
  // -D  的参数
  final Configuration dynamicProperties = ConfigurationUtils.createConfiguration(entrypointClusterConfiguration.getDynamicProperties());
  final Configuration configuration = GlobalConfiguration.loadConfiguration(entrypointClusterConfiguration.getConfigDir(), dynamicProperties);
  // 设置 WEB-UI host:port
  ......
  return configuration;
}
// org.apache.flink.configuration.GlobalConfiguration
public static Configuration loadConfiguration(final String configDir, @Nullable final Configuration dynamicProperties) {
  ......
  // get Flink yaml configuration file
  final File yamlConfigFile = new File(confDirFile, FLINK_CONF_FILENAME);
  if (!yamlConfigFile.exists()) {
    throw new IllegalConfigurationException(
      "The Flink config file '" + yamlConfigFile +
      "' (" + confDirFile.getAbsolutePath() + ") does not exist.");
  }
  // 重点：获取 flink-conf.yml 中的参数形成键值对
  Configuration configuration = loadYAMLResource(yamlConfigFile);
  if (dynamicProperties != null) {
    configuration.addAll(dynamicProperties);
  }
  return configuration;
}

```

- Run

```java
// org.apache.flink.runtime.entrypoint.ClusterEntrypoint
/**
 * Base class for the Flink cluster entry points.
 * Specialization of this class can be used for the session mode and the per-job mode
 */
public static void runClusterEntrypoint(ClusterEntrypoint clusterEntrypoint) {
  final String clusterEntrypointName = clusterEntrypoint.getClass().getSimpleName();
  try {
    // start
    clusterEntrypoint.startCluster();
  } catch (ClusterEntrypointException e) {
   ......
}

```

