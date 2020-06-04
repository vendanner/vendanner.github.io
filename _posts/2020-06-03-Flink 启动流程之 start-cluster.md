---
layout:     post
title:      Flink 启动流程之 start-cluster
subtitle:   
date:       2020-06-03
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - bigdata
---

### start-cluster.sh

``` shell
bin=`dirname "$0"`
bin=`cd "$bin"; pwd`
# 执行同目录下的 config.sh：读取 conf 目录下配置文件
# 并定义一些函数：TMSlaves，readMasters，readSlaves 供调用
. "$bin"/config.sh

# 启动 JobManager，分是否为 HA
shopt -s nocasematch
if [[ $HIGH_AVAILABILITY == "zookeeper" ]]; then
    # HA Mode
    # 执行 readMasters 是读取 conf/masters 文件
    readMasters
    echo "Starting HA cluster with ${#MASTERS[@]} masters."
    for ((i=0;i<${#MASTERS[@]};++i)); do
        master=${MASTERS[i]}
        webuiport=${WEBUIPORTS[i]}

        if [ ${MASTERS_ALL_LOCALHOST} = true ] ; then
            "${FLINK_BIN_DIR}"/jobmanager.sh start "${master}" "${webuiport}"
        else
            ssh -n $FLINK_SSH_OPTS $master -- "nohup /bin/bash -l \"${FLINK_BIN_DIR}/jobmanager.sh\" start ${master} ${webuiport} &"
        fi
    done
else
    echo "Starting cluster."
    # Start single JobManager on this machine
    "$FLINK_BIN_DIR"/jobmanager.sh start
fi
shopt -u nocasematch

# 启动 TaskManager instance(s)
TMSlaves start
```

### jobmanager.sh

```shell
USAGE="Usage: jobmanager.sh ((start|start-foreground) [host] [webui-port])|stop|stop-all"

STARTSTOP=$1
HOST=$2 # optional when starting multiple instances
WEBUIPORT=$3 # optional when starting multiple instances
...
bin=`dirname "$0"`
bin=`cd "$bin"; pwd`
. "$bin"/config.sh
# jobManager 标识
ENTRYPOINT=standalonesession

if [[ $STARTSTOP == "start" ]] || [[ $STARTSTOP == "start-foreground" ]]; then
    if [ ! -z "${FLINK_JM_HEAP_MB}" ] && [ "${FLINK_JM_HEAP}" == 0 ]; then
	    echo "used deprecated key \`${KEY_JOBM_MEM_MB}\`, please replace with key \`${KEY_JOBM_MEM_SIZE}\`"
    else
	    flink_jm_heap_bytes=$(parseBytes ${FLINK_JM_HEAP})
	    FLINK_JM_HEAP_MB=$(getMebiBytes ${flink_jm_heap_bytes})
    fi
    if [[ ! ${FLINK_JM_HEAP_MB} =~ $IS_NUMBER ]] || [[ "${FLINK_JM_HEAP_MB}" -lt "0" ]]; then
        echo "[ERROR] Configured JobManager memory size is not a valid value. Please set '${KEY_JOBM_MEM_SIZE}' in ${FLINK_CONF_FILE}."
        exit 1
    fi
    # 设置 JVM 参数
    if [ "${FLINK_JM_HEAP_MB}" -gt "0" ]; then
        export JVM_ARGS="$JVM_ARGS -Xms"$FLINK_JM_HEAP_MB"m -Xmx"$FLINK_JM_HEAP_MB"m"
    fi
    # Add JobManager-specific JVM options
    export FLINK_ENV_JAVA_OPTS="${FLINK_ENV_JAVA_OPTS} ${FLINK_ENV_JAVA_OPTS_JM}"
    # Startup parameters
    args=("--configDir" "${FLINK_CONF_DIR}" "--executionMode" "cluster")
    if [ ! -z $HOST ]; then
        args+=("--host")
        args+=("${HOST}")
    fi

    if [ ! -z $WEBUIPORT ]; then
        args+=("--webui-port")
        args+=("${WEBUIPORT}")
    fi
fi
# 组装好参数 args 执行另一个 sh 来启动
# flink-console 在控制台执行，只执行 start 命令
# flink-daemon nohup 后台执行，可以执行 start/stop/stop-all 命令
if [[ $STARTSTOP == "start-foreground" ]]; then
    exec "${FLINK_BIN_DIR}"/flink-console.sh $ENTRYPOINT "${args[@]}"
else
    "${FLINK_BIN_DIR}"/flink-daemon.sh $STARTSTOP $ENTRYPOINT "${args[@]}"
fi

```

### taskmanager.sh

``` shell
# config.sh
# 读取 conf/slaves 文件中并通过 extractHostName 提取的 hostname
# 判断所有 slaver 是否都为本地，SLAVES_ALL_LOCALHOST 标识
readSlaves() {
    SLAVES_FILE="${FLINK_CONF_DIR}/slaves"
    if [[ ! -f "$SLAVES_FILE" ]]; then
        echo "No slaves file. Please specify slaves in 'conf/slaves'."
        exit 1
    fi
    SLAVES=()
    SLAVES_ALL_LOCALHOST=true
    GOON=true
    while $GOON; do
        read line || GOON=false
        HOST=$( extractHostName $line)
        if [ -n "$HOST" ] ; then
            SLAVES+=(${HOST})
            if [ "${HOST}" != "localhost" ] && [ "${HOST}" != "127.0.0.1" ] ; then
                SLAVES_ALL_LOCALHOST=false
            fi
        fi
    done < "$SLAVES_FILE"
}
# readSlaves 获取到所有 slave 节点
# 若包含非本节点，则ssh taskmanager.sh 来启动
TMSlaves() {
    CMD=$1
    readSlaves
    if [ ${SLAVES_ALL_LOCALHOST} = true ] ; then
        # all-local setup
        for slave in ${SLAVES[@]}; do
            "${FLINK_BIN_DIR}"/taskmanager.sh "${CMD}"
        done
    else
        # non-local setup
        # Stop TaskManager instance(s) using pdsh (Parallel Distributed Shell) when available
        command -v pdsh >/dev/null 2>&1
        if [[ $? -ne 0 ]]; then
            for slave in ${SLAVES[@]}; do
                ssh -n $FLINK_SSH_OPTS $slave -- "nohup /bin/bash -l \"${FLINK_BIN_DIR}/taskmanager.sh\" \"${CMD}\" &"
            done
        else
            PDSH_SSH_ARGS="" PDSH_SSH_ARGS_APPEND=$FLINK_SSH_OPTS pdsh -w $(IFS=, ; echo "${SLAVES[*]}") \
                "nohup /bin/bash -l \"${FLINK_BIN_DIR}/taskmanager.sh\" \"${CMD}\""
        fi
    fi
}

# taskmanager.sh
# 在 TMSlaves 中会被多次调用来启动多个 TM
# 内容与 jobManager 大同小异：设置 JVM 参数，再另外调 sh 执行：console/daemon
USAGE="Usage: taskmanager.sh (start|start-foreground|stop|stop-all)"
STARTSTOP=$1
ARGS=("${@:2}")
if [[ $STARTSTOP != "start" ]] && [[ $STARTSTOP != "start-foreground" ]] && [[ $STARTSTOP != "stop" ]] && [[ $STARTSTOP != "stop-all" ]]; then
  echo $USAGE
  exit 1
fi
bin=`dirname "$0"`
bin=`cd "$bin"; pwd`

. "$bin"/config.sh
# taskManager 标识
ENTRYPOINT=taskexecutor

if [[ $STARTSTOP == "start" ]] || [[ $STARTSTOP == "start-foreground" ]]; then

    # if memory allocation mode is lazy and no other JVM options are set,
    # set the 'Concurrent Mark Sweep GC'
    if [ -z "${FLINK_ENV_JAVA_OPTS}" ] && [ -z "${FLINK_ENV_JAVA_OPTS_TM}" ]; then
        export JVM_ARGS="$JVM_ARGS -XX:+UseG1GC"
    fi
    # Add TaskManager-specific JVM options
    export FLINK_ENV_JAVA_OPTS="${FLINK_ENV_JAVA_OPTS} ${FLINK_ENV_JAVA_OPTS_TM}"
    # Startup parameters
    jvm_params_output=`runBashJavaUtilsCmd GET_TM_RESOURCE_JVM_PARAMS ${FLINK_CONF_DIR}`
    jvm_params=`extractExecutionParams "$jvm_params_output"`
    if [[ $? -ne 0 ]]; then
        echo "[ERROR] Could not get JVM parameters properly."
        exit 1
    fi
    export JVM_ARGS="${JVM_ARGS} ${jvm_params}"

    IFS=$" "
    dynamic_configs_output=`runBashJavaUtilsCmd GET_TM_RESOURCE_DYNAMIC_CONFIGS ${FLINK_CONF_DIR}`
    dynamic_configs=`extractExecutionParams "$dynamic_configs_output"`
    if [[ $? -ne 0 ]]; then
        echo "[ERROR] Could not get dynamic configurations properly."
        exit 1
    fi
    ARGS+=("--configDir" "${FLINK_CONF_DIR}" ${dynamic_configs[@]})
    export FLINK_INHERITED_LOGS="
$FLINK_INHERITED_LOGS

TM_RESOURCES_JVM_PARAMS extraction logs:
$jvm_params_output

TM_RESOURCES_DYNAMIC_CONFIGS extraction logs:
$dynamic_configs_output
"
fi
if [[ $STARTSTOP == "start-foreground" ]]; then
    exec "${FLINK_BIN_DIR}"/flink-console.sh $ENTRYPOINT "${ARGS[@]}"
else
    if [[ $FLINK_TM_COMPUTE_NUMA == "false" ]]; then
        # Start a single TaskManager
        "${FLINK_BIN_DIR}"/flink-daemon.sh $STARTSTOP $ENTRYPOINT "${ARGS[@]}"
    else
        # Example output from `numactl --show` on an AWS c4.8xlarge:
        # policy: default
        # preferred node: current
        # physcpubind: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35
        # cpubind: 0 1
        # nodebind: 0 1
        # membind: 0 1
        read -ra NODE_LIST <<< $(numactl --show | grep "^nodebind: ")
        for NODE_ID in "${NODE_LIST[@]:1}"; do
            # Start a TaskManager for each NUMA node
            numactl --membind=$NODE_ID --cpunodebind=$NODE_ID -- "${FLINK_BIN_DIR}"/flink-daemon.sh $STARTSTOP $ENTRYPOINT "${ARGS[@]}"
        done
    fi
fi
```

### flink-daemon.sh

分析 JobManager 和 TaskManager 时，最后都是执行  `flink-daemon.sh`

``` shell
STARTSTOP=$1
DAEMON=$2
ARGS=("${@:3}") # get remaining arguments as array
bin=`dirname "$0"`
bin=`cd "$bin"; pwd`
. "$bin"/config.sh
# 根据参数执行对应的 Java Class
case $DAEMON in
    (taskexecutor)
        CLASS_TO_RUN=org.apache.flink.runtime.taskexecutor.TaskManagerRunner
    ;;
    (zookeeper)
        CLASS_TO_RUN=org.apache.flink.runtime.zookeeper.FlinkZooKeeperQuorumPeer
    ;;
    (historyserver)
        CLASS_TO_RUN=org.apache.flink.runtime.webmonitor.history.HistoryServer
    ;;
    (standalonesession)
        CLASS_TO_RUN=org.apache.flink.runtime.entrypoint.StandaloneSessionClusterEntrypoint
    ;;
    (standalonejob)
        CLASS_TO_RUN=org.apache.flink.container.entrypoint.StandaloneJobClusterEntryPoint
    ;;
    (*)
        echo "Unknown daemon '${DAEMON}'. $USAGE."
        exit 1
    ;;
esac
if [ "$FLINK_IDENT_STRING" = "" ]; then
    FLINK_IDENT_STRING="$USER"
fi
FLINK_TM_CLASSPATH=`constructFlinkClassPath`
# pid 路径如果没有在 flink-conf.yaml 设置env.pid.dir，默认 /tmp
# 这个路径很重要，用于检测/杀死进程
# JM 和 TM 进程有各自的 pid 文件
pid=$FLINK_PID_DIR/flink-$FLINK_IDENT_STRING-$DAEMON.pid
mkdir -p "$FLINK_PID_DIR"
# Log files for daemons are indexed from the process ID's position in the PID
# file. The following lock prevents a race condition during daemon startup
# when multiple daemons read, index, and write to the PID file concurrently.
# The lock is created on the PID directory since a lock file cannot be safely
# removed. The daemon is started with the lock closed and the lock remains
# active in this script until the script exits.
command -v flock >/dev/null 2>&1
if [[ $? -eq 0 ]]; then
    exec 200<"$FLINK_PID_DIR"
    flock 200
fi
# Ascending ID depending on number of lines in pid file.
# This allows us to start multiple daemon of each type.
id=$([ -f "$pid" ] && echo $(wc -l < "$pid") || echo "0")
# log path = flink-conf.yaml 中 env.log.dir
# 如果没设置,默认 $FLINK_HOME/log
FLINK_LOG_PREFIX="${FLINK_LOG_DIR}/flink-${FLINK_IDENT_STRING}-${DAEMON}-${id}-${HOSTNAME}"
log="${FLINK_LOG_PREFIX}.log"
out="${FLINK_LOG_PREFIX}.out"
log_setting=("-Dlog.file=${log}" "-Dlog4j.configuration=file:${FLINK_CONF_DIR}/log4j.properties" "-Dlogback.configurationFile=file:${FLINK_CONF_DIR}/logback.xml")
# 获取 java 版本命令 jdk1.8 -> 18
JAVA_VERSION=$(${JAVA_RUN} -version 2>&1 | sed 's/.*version "\(.*\)\.\(.*\)\..*"/\1\2/; 1q')
# Only set JVM 8 arguments if we have correctly extracted the version
if [[ ${JAVA_VERSION} =~ ${IS_NUMBER} ]]; then
    if [ "$JAVA_VERSION" -lt 18 ]; then
        JVM_ARGS="$JVM_ARGS -XX:MaxPermSize=256m"
    fi
fi
# 启动
case $STARTSTOP in
    (start)
        # Rotate log files
        rotateLogFilesWithPrefix "$FLINK_LOG_DIR" "$FLINK_LOG_PREFIX"
        # Print a warning if daemons are already running on host
        # 允许存在多个进程
        if [ -f "$pid" ]; then
          active=()
          while IFS='' read -r p || [[ -n "$p" ]]; do
            kill -0 $p >/dev/null 2>&1
            if [ $? -eq 0 ]; then
              active+=($p)
            fi
          done < "${pid}"
          count="${#active[@]}"
          if [ ${count} -gt 0 ]; then
            echo "[INFO] $count instance(s) of $DAEMON are already running on $HOSTNAME."
          fi
        fi
        # Evaluate user options for local variable expansion
        FLINK_ENV_JAVA_OPTS=$(eval echo ${FLINK_ENV_JAVA_OPTS})
        echo "Starting $DAEMON daemon on host $HOSTNAME."
        # 启动进程
        $JAVA_RUN $JVM_ARGS ${FLINK_ENV_JAVA_OPTS} "${log_setting[@]}" -classpath "`manglePathList "$FLINK_TM_CLASSPATH:$INTERNAL_HADOOP_CLASSPATHS"`" ${CLASS_TO_RUN} "${ARGS[@]}" > "$out" 200<&- 2>&1 < /dev/null &
        mypid=$!
        # 进程号 写进 pid 文件 
        if [[ ${mypid} =~ ${IS_NUMBER} ]] && kill -0 $mypid > /dev/null 2>&1 ; then
            echo $mypid >> "$pid"
        else
            echo "Error starting $DAEMON daemon."
            exit 1
        fi
    ;;
    # 根据 pid 文件 kill 进程
    (stop)
        if [ -f "$pid" ]; then
            # 最后一个进程开始杀
            to_stop=$(tail -n 1 "$pid")
            if [ -z $to_stop ]; then
                rm "$pid" # If all stopped, clean up pid file
                echo "No $DAEMON daemon to stop on host $HOSTNAME."
            else
                sed \$d "$pid" > "$pid.tmp" # all but last line
                # If all stopped, clean up pid file
                [ $(wc -l < "$pid.tmp") -eq 0 ] && rm "$pid" "$pid.tmp" || mv "$pid.tmp" "$pid"
                if kill -0 $to_stop > /dev/null 2>&1; then
                    echo "Stopping $DAEMON daemon (pid: $to_stop) on host $HOSTNAME."
                    kill $to_stop
                else
                    echo "No $DAEMON daemon (pid: $to_stop) is running anymore on $HOSTNAME."
                fi
            fi
        else
            echo "No $DAEMON daemon to stop on host $HOSTNAME."
        fi
    ;;
    (stop-all)
        if [ -f "$pid" ]; then
            mv "$pid" "${pid}.tmp"

            while read to_stop; do
                if kill -0 $to_stop > /dev/null 2>&1; then
                    echo "Stopping $DAEMON daemon (pid: $to_stop) on host $HOSTNAME."
                    kill $to_stop
                else
                    echo "Skipping $DAEMON daemon (pid: $to_stop), because it is not running anymore on $HOSTNAME."
                fi
            done < "${pid}.tmp"
            rm "${pid}.tmp"
        fi
    ;;
    (*)
        echo "Unexpected argument '$STARTSTOP'. $USAGE."
        exit 1
    ;;
esac
```

### stop-cluster.sh

```shell
bin=`dirname "$0"`
bin=`cd "$bin"; pwd`
. "$bin"/config.sh
# Stop TaskManager instance(s)
TMSlaves stop
# Stop JobManager instance(s)
shopt -s nocasematch
if [[ $HIGH_AVAILABILITY == "zookeeper" ]]; then
    # HA Mode
    readMasters
    if [ ${MASTERS_ALL_LOCALHOST} = true ] ; then
        for master in ${MASTERS[@]}; do
            "$FLINK_BIN_DIR"/jobmanager.sh stop
        done
    else
        for master in ${MASTERS[@]}; do
            ssh -n $FLINK_SSH_OPTS $master -- "nohup /bin/bash -l \"${FLINK_BIN_DIR}/jobmanager.sh\" stop &"
        done
    fi
else
    "$FLINK_BIN_DIR"/jobmanager.sh stop
fi
shopt -u nocasematch

```

与 start-cluster 相同的讨论，只不过是 `stop` 操作，最终的执行者还是 `flink-daemon.sh`

### Java Code

#### JobManager

```java
/**
 * org.apache.flink.runtime.entrypoint.StandaloneSessionClusterEntrypoint
 * Entry point for the standalone session cluster.
 */
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
			entrypointClusterConfiguration = commandLineParser.parse(args);
		} catch (FlinkParseException e) {
			LOG.error("Could not parse command line arguments {}.", args, e);
			commandLineParser.printHelp(StandaloneSessionClusterEntrypoint.class.getSimpleName());
			System.exit(1);
		}
		Configuration configuration = loadConfiguration(entrypointClusterConfiguration);
		StandaloneSessionClusterEntrypoint entrypoint = new StandaloneSessionClusterEntrypoint(configuration);
		ClusterEntrypoint.runClusterEntrypoint(entrypoint);
	}
}

```

#### TaskManager

```java
// org.apache.flink.runtime.taskexecutor.TaskManagerRunner
/**
 * This class is the executable entry point for the task manager in yarn or standalone mode.
 * It constructs the related components (network, I/O manager, memory manager, RPC service, HA service)
 * and starts them.
 */
public class TaskManagerRunner implements FatalErrorHandler, AutoCloseableAsync {
	public static void main(String[] args) throws Exception {
		// startup checks and logging
		EnvironmentInformation.logEnvironmentInfo(LOG, "TaskManager", args);
		SignalHandler.register(LOG);
		JvmShutdownSafeguard.installAsShutdownHook(LOG);
		long maxOpenFileHandles = EnvironmentInformation.getOpenFileHandlesLimit();
		if (maxOpenFileHandles != -1L) {
			LOG.info("Maximum number of open file descriptors is {}.", maxOpenFileHandles);
		} else {
			LOG.info("Cannot determine the maximum number of open file descriptors");
		}
    // run
		runTaskManagerSecurely(args, ResourceID.generate());
	}
}
```

### 总结

- config.sh 读取 `conf` 下的文件，以加载环境变量并提供一连串的**功能函数**供调用
- jobmanager.sh 设置一些 **JVM 参数**，并通过 `flink-daemon.sh` 启动 Java 类，**默认**在 `/tmp` 目录有 **pid 文件**
- taskmanager.sh 设置一些 **JVM 参数**，并通过 `flink-daemon.sh` 启动 Java 类，**默认**在 `/tmp` 目录有 **pid 文件**