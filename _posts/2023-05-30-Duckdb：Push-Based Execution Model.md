---
layout:     post
title:      Duckdb：Push-Based Execution Model
subtitle:
date:       2023-05-30
author:     danner
header-img: img/bg.jpg
catalog: true
tags:
    - duckdb
    - db
---

> Push data into operator when data is available

### PhysicalOperator

可以分为三种类型：

- Operator：Transform，数据处理
- Source：数据源
- Sink：数据汇总，当前Pipeline 的Sink 必定是下一个Pipeline 的Source

一般Operator 都是以上某一类别，但有少部分包含以上所有：`PhysicalHashJoin` 包含三种类型。

#### Operator Interface

**处理数据**：`Projection`,` Filter`,` Hash Probe`, …

##### Execute

接收数据，处理后继续往下游发送数据

- NEED_MORE_INPUT：已处理完当前输入，等待下一次输入
- HAVE_MORE_OUTPUT：将使用相同的输入再次调用此 Operator(Operator输入相同 input data 会产不同的输出)
- FINISHED：不再产生输出(提早结束此Pipeline，后续sink 也不会执行)

##### Vector Cach

batch

#### Source Interface

**发出数据**：`Table scan`, `aggregate HT scan`,` ORDER BY scan`

##### GetData

获取数据

- HAVE_MORE_OUTPUT：还可以继续输出数据(当有数据返回时设置，若无数据设置为FINISHED/BLOCKED)
- FINISHED：数据已全部获取
- BLOCKED：在 `async I/O` 情况下

#### Sink Interface

##### sink

- NEED_MORE_INPUT：需要更多输入
- FINISHED：执行已结束(可以提早停止)，继续输入也不会有更多变化
- BLOCKED：在 `async I/O` 情况下

##### Finalize

所有线程都执行完成时才执行 `Finalize`，在pipeline 中只会**执行一次**(上面所有函数都会并行执行)。

- READY：准备就绪
- NO_OUTPUT_POSSIBLE：不会再输出，提早停止pipeline 的后续执行(hash join 时，probe 发现buidl table为空，可以直接结束)
- FINISHED：`sink` 标记完成

### Event

每个pipeline 都包含以下四种 event。

#### PipelineInitializeEvent

当前版本，相当于没有任何操作

#### PipelineEvent

执行`pipeline` ：从Source 获取数据，经过Operator 后推到Sink，如此反复直到Source 无数据

PipelineTask.ExecuteTask

#### PipelineFinishEvent

结束`pipeline` 运行：会执行Sink->Finalize

#### PipelineCompleteEvent

标记`pipeline` 完成，并开始调度依赖此pipeline 的其他pipeline(event 的依赖event 都执行完成时，会执行event.schedule 来对pipeline 生成新的task)

- 描述不是很准确，正确描述看下面
  - Executor 中完成的 pipeline 计数+1 ，表示当前 pipeline 已执行完成
  - 此event 执行结束后，对应pipeline 生成的 task 都已执行结束，调度器会从队列中取出其他pipeline 任务(开始执行其他pipeline)

### PipelineExecutor

```c++
// Executor::SchedulePipeline => 创建 pipeline events
// create events/stack for the base pipeline
auto base_pipeline = meta_pipeline->GetBasePipeline();
auto base_initialize_event = make_shared<PipelineInitializeEvent>(base_pipeline);
auto base_event = make_shared<PipelineEvent>(base_pipeline);
auto base_finish_event = make_shared<PipelineFinishEvent>(base_pipeline);
auto base_complete_event = make_shared<PipelineCompleteEvent>(base_pipeline->executor, event_data.initial_schedule);
PipelineEventStack base_stack(*base_initialize_event, *base_event, *base_finish_event, *base_complete_event);
events.push_back(std::move(base_initialize_event));
events.push_back(std::move(base_event));
events.push_back(std::move(base_finish_event));
events.push_back(std::move(base_complete_event));

// dependencies: initialize -> event -> finish -> complete
base_stack.pipeline_event.AddDependency(base_stack.pipeline_initialize_event);
base_stack.pipeline_finish_event.AddDependency(base_stack.pipeline_event);
base_stack.pipeline_complete_event.AddDependency(base_stack.pipeline_finish_event);

// 设置同个meta_pipeline 内pipeline 间event 依赖
PipelineEventStack pipeline_stack(base_stack.pipeline_initialize_event, *pipeline_event,
                                  *pipeline_finish_event_ptr, base_stack.pipeline_complete_event);
events.push_back(std::move(pipeline_event));

// dependencies: base_initialize -> pipeline_event -> base_finish
pipeline_stack.pipeline_event.AddDependency(base_stack.pipeline_initialize_event);
pipeline_stack.pipeline_finish_event.AddDependency(pipeline_stack.pipeline_event);
...


// Executor::ScheduleEventsInternal
auto &event_map = event_data.event_map;
for (auto &entry : event_map) {
  auto &pipeline = entry.first.get();
  for (auto &dependency : pipeline.dependencies) {
    auto dep = dependency.lock();
    D_ASSERT(dep);
    auto event_map_entry = event_map.find(*dep);
    D_ASSERT(event_map_entry != event_map.end());
    auto &dep_entry = event_map_entry->second;
    // 设置跨 meta_pipeline 之间的event 依赖
    entry.second.pipeline_event.AddDependency(dep_entry.pipeline_complete_event);
  }
}
```

>  initialize -> event -> finish -> complete

每个event 都会触发下游event，若下游event 的**依赖event** 都执行结束则会马上调用自己的 `schedule` 函数。

```c++
// duckdb::Event
void Event::CompleteDependency() {
	idx_t current_finished = ++finished_dependencies;
	D_ASSERT(current_finished <= total_dependencies);
	if (current_finished == total_dependencies) {
		// all dependencies have been completed: schedule the event
		D_ASSERT(total_tasks == 0);
		Schedule();
		if (total_tasks == 0) {		// 若当前 event 不生成 task，直接执行 finish
			Finish();
		}
	}
}

void Event::Finish() {
	D_ASSERT(!finished);
	FinishEvent();
	finished = true;
	// finished processing the pipeline, now we can schedule pipelines that depend on this pipeline
	for (auto &parent_entry : parents) {
		auto parent = parent_entry.lock();
		if (!parent) { // LCOV_EXCL_START
			continue;
		} // LCOV_EXCL_STOP
		// mark a dependency as completed for each of the parents
		parent->CompleteDependency();
	}
	FinalizeFinish();
}
```



#### initialize

没有实质操作

#### event

PipelineExecutor 开始执行，任务会并行(并行数max_threads)运行

```c++
// PipelineEvent::Schedule
// -> Pipeline::Schedule -> ScheduleParallel -> LaunchScanTasks 
bool Pipeline::LaunchScanTasks(shared_ptr<Event> &event, idx_t max_threads) {
	// split the scan up into parts and schedule the parts
	auto &scheduler = TaskScheduler::GetScheduler(executor.context);
	idx_t active_threads = scheduler.NumberOfThreads();
	if (max_threads > active_threads) {
		max_threads = active_threads;
	}
	if (max_threads <= 1) {
		// too small to parallelize
		return false;
	}

	// launch a task for every thread
  // 并行任务 PipelineTask，以pipeline 为单位
	vector<shared_ptr<Task>> tasks;
	for (idx_t i = 0; i < max_threads; i++) {
		tasks.push_back(make_uniq<PipelineTask>(*this, event));
	}
  // 任务写入TaskScheduler 队列，等待调度
	event->SetTasks(std::move(tasks));
	return true;
}  
```

调度后执行: `PipelineTask::ExecuteTask`

主要逻辑在 `PipelineExecutor::Execute`，大致分为以下步骤

- source 还有数据时 (exhausted_source=false)，FetchFromSource 获取数据
  -  -> pipeline.source->GetData

- ExecutePushInternal
  - operators-> execute -> sink
  - PipelineExecutor::ExecutePushInternal() 可以看做是 Pipeline 内的数据消费者。

- PushFinalize： sink->combine
  - source 数据都消费完且数据都 flush，才会执行到这一步
  - 返回 PipelineExecuteResult::FINISHED

`PipelineExecutor::Execute` 执行返回后，判断所有任务是否都已完成(max_threads 任务量)；若都已完成调用 `Finish` 函数

- 执行 `FinishEvent` 函数，大部分情况是空函数，较少情况有重写(PipelineFinishEvent)
- 标记当前Event 已完成
- 依赖此Event 的Event 成员函数 `++finished_dependencies`，若依赖都执行完，开始执行`Event.Schedule`(生成任务压入队列)
- 执行 `FinalizeFinish` 函数，大部分情况是空函数，较少情况有重写(PipelineCompleteEvent)

最后标记当前任务TaskExecutionResult::TASK_FINISHED

> - 先调用 FetchFromSource() 从 Pipeline 的 source PhysicalOperator 中获取计算结果作为 source DataChunk，这里会调用 source 的 GetData() 接口。
> - 再调用 ExecutePushInternal() 依次执行 Pipeline 中 operators 列表中的各个 PhysicalOperator 和最后一个 sink PhysicalOperator 完成这批数据后续的所有计算操作。对于普通 operator 会调用它的 Execute() 接口，对最后的 sink 会调用它的 Sink() 接口。PipelineExecutor::ExecutePushInternal() 可以看做是 Pipeline 内的数据消费者。
> - 最后调用 PushFinalize() 完成当前 ExecutorTask 的执行，这里会调用 sink 的 Combine 接口，用以完成一个 ExecutorTask 结束后的收尾清理工作。

#### finish

什么时候会触发呢？ 当PipelineEvent 执行完后，会触发 => 看上面添加依赖和计算依赖完成数量的代码

```c++
// PipelineFinishEvent::FinishEvent
void PipelineFinishEvent::FinishEvent() {
	pipeline->Finalize(*this);
}
// Pipeline::Finalize
void Pipeline::Finalize(Event &event) {
  ...
  auto sink_state = sink->Finalize(*this, event, executor.context, *sink->sink_state);
	sink->sink_state->state = sink_state;
  ...
}
```

调用 `sink->Finalize`

#### complete

 当PipelineFinishEvent 执行完后，会触发

```c++
// PipelineCompleteEvent::FinalizeFinish
void PipelineCompleteEvent::FinalizeFinish() {
	if (complete_pipeline) {
		executor.CompletePipeline();
	}
}
// Executor::CompletePipeline
void CompletePipeline() {
  completed_pipelines++;
}
```

Executor 中完成的 pipeline 计数+1

### 小结

- Pipeline 内部，将数据推到 Operator 来触发执行
- 同个Meta_Pipeline内不同pipeline 之间，靠 `event` 触发
  - PipelineInitializeEvent 会触发所有 pipeline ：调用各自 PipelineEvent 将生成任务并压入队列
- 不同 Meta_Pipeline，也是靠`event` 触发
  - 当上游Meta_Pipelines 的 `PipelineCompleteEvent` 完成后，会触发下游 `PipelineEvent`
  - 而下游 `PipelineEvent` 会调用schedule 生成任务并压入队列

结合本文，好好理解这句话

> Pipelines are no longer scheduled as-is. Instead, pipelines are split up into "events" and events are scheduled.





## 参考资料

[Switch to Push-Based Execution Model](https://github.com/duckdb/duckdb/pull/2393)

[Push-Based Execution in DuckDB](https://dsdsd.da.cwi.nl/slides/dsdsd-duckdb-push-based-execution.pdf)

[[DuckDB] Push-Based Execution Model](https://zhuanlan.zhihu.com/p/402355976)