# Indexlib 索引任务框架：监控与历史追溯机制剖析

## 涉及文件

- `framework/index_task/IndexTaskHistory.h`
- `framework/index_task/IndexTaskHistory.cpp`
- `framework/index_task/IndexTaskMetrics.h`
- `framework/index_task/IndexTaskMetrics.cpp`
- `framework/index_task/IndexOperationMetrics.h`
- `framework/index_task/IndexOperationMetrics.cpp`

## 1. 引言

一个生产级的系统，除了强大的功能外，还必须具备完善的可观测性（Observability）。对于 Indexlib 的后台任务框架而言，这意味着需要能够实时监控任务状态、度量关键性能指标，并能追溯历史任务的执行情况。这不仅对于问题排查至关重要，也是性能优化和容量规划的基础。Indexlib 通过 `IndexTaskMetrics` 和 `IndexTaskHistory` 两大组件，构建了一套全面的监控与历史追溯体系。本文将深入剖析其设计理念、实现细节以及它们如何为 Indexlib 的稳定运行提供保障。

## 2. 架构设计：分离的度量与历史

该模块的设计清晰地将两个核心概念分离开来：

1.  **度量 (Metrics)**: 关注的是**瞬时**或**聚合**的数值型指标。例如，当前正在运行的任务是什么？合并操作耗时多少？这些数据通常用于实时监控、告警和性能趋势分析。`IndexTaskMetrics` 和 `IndexOperationMetrics` 负责此部分。

2.  **历史 (History)**: 关注的是**事件**的记录和追溯。例如，历史上执行过哪些合并任务？每次任务的触发时间、输入版本和输出版本是什么？这些信息是结构化的日志，用于事后分析、审计和调试。`IndexTaskHistory` 负责此部分。

这种分离的设计使得两部分可以独立演进，并使用最适合各自场景的技术栈。度量系统对接了 `kmonitor`，这是一个专业的时序数据库监控系统；而历史记录则使用了简单、可移植的 JSON 格式进行序列化，便于存储和解析。

## 3. `IndexTaskMetrics` & `IndexOperationMetrics`：实时性能的脉搏

这组类负责收集和上报任务执行过程中的性能指标。

### 3.1. `IndexOperationMetrics`：操作级的精细度量

-   **职责**: 专门用于度量单个 `IndexOperation` 的执行性能。
-   **核心指标**: 目前最核心的指标是 `operationExecuteTime`，通过一个 `autil::ScopedTime2` 对象来精确测量 `Execute` 方法的耗时（微秒级）。
-   **实现方式**: 在 `IndexOperation::ExecuteWithLog` 方法中，会为每个操作创建一个 `IndexOperationMetrics` 实例。当操作执行完毕，`ScopedTime2` 析构时会自动记录时间。这些指标通过 `MetricsManager` 定期上报。
-   **Tags**: 为了能够在监控系统中进行多维度分析，上报的指标会附带丰富的 `kmonitor::MetricsTags`，如 `opId` 和 `opName`。这使得用户可以轻松地聚合查询特定类型操作的平均耗时、P99 耗时等。

关键代码 (`IndexOperation.cpp`):

```cpp
Status IndexOperation::ExecuteWithLog(const IndexTaskContext& context)
{
    // ...
    auto manager = context.GetMetricsManager();
    if (manager) {
        kmonitor::MetricsTags tags;
        tags.AddTag("opId", std::to_string(_opId));
        tags.AddTag("opName", ...);
        // ...
        _operationMetrics =
            std::dynamic_pointer_cast<IndexOperationMetrics>(manager->CreateMetrics(identifier, creatorFunc));
    }
    // ...
    status = Execute(context);
    // ...
    if (_operationMetrics) {
        doneTimeUs = _operationMetrics->GetTimerDoneUs();
        _operationMetrics.reset(); // 触发上报
    }
    // ...
    return status;
}
```

### 3.2. `IndexTaskMetrics`：任务级的宏观监控

-   **职责**: 从更宏观的视角监控整个任务框架的状态。
-   **核心指标**:
    -   **合并进度**: `mergeBaseVersionId`, `mergeTargetVersionId`, `mergeCommittedVersionId` 等，直观反映了合并任务的进展。
    -   **任务新鲜度**: `finishedTaskTriggerTimeFreshness`, `runningTaskTriggerTimeFreshness` 等，通过计算当前时间与任务触发时间的差值，可以监控任务是否延迟或卡住。
    -   **运行状态**: `runningMergeLeftOps`, `runningMergeTotalOps`，提供了任务内部操作的完成情况。
-   **线程安全**: `IndexTaskMetrics` 内部使用 `std::mutex` 来保护其成员变量，因为它的状态可能由任务主线程和度量上报线程并发访问。
-   **信息填充**: 提供了 `FillMetricsInfo` 方法，可以将核心指标填充到一个 `map` 中，便于在其他系统（如运维命令行工具）中展示。

## 4. `IndexTaskHistory`：任务执行的备忘录

`IndexTaskHistory` 负责记录和管理已完成任务的日志信息。它是一个可序列化的对象，通常会持久化到磁盘上，作为 Tablet 状态的一部分。

### 4.1. 核心数据结构

`IndexTaskHistory` 的核心是一个 `std::map<std::string, std::vector<IndexTaskLogPtr>>`，其中：

-   `key (string)`: 任务类型 (`taskType`)，例如 `merge` 或 `alter_table`。
-   `value (vector<IndexTaskLogPtr>)`: 一个 `IndexTaskLog` 的智能指针列表。每个 `IndexTaskLog` 代表一次具体的任务执行。

`IndexTaskLog` 本身也是一个可序列化的对象，包含了任务的关键信息：

```cpp
class IndexTaskLog : public autil::legacy::Jsonizable
{
public:
    // ...
    void Jsonize(JsonWrapper& json) override
    {
        json.Jsonize("task_id", _taskId);
        json.Jsonize("base_version", _baseVersion);
        json.Jsonize("target_version", _targetVersion);
        json.Jsonize("trigger_timestamp", _triggerTimestampInSec);
        json.Jsonize("task_description", _taskDescription);
    }
private:
    std::string _taskId;
    versionid_t _baseVersion = -1;
    versionid_t _targetVersion = -1;
    int64_t _triggerTimestampInSec = -1;
    std::map<std::string, std::string> _taskDescription;
};
```

### 4.2. 功能与实现

-   **日志添加与排序 (`AddLog`)**: 当一个任务完成时，一个新的 `IndexTaskLog` 会被添加到 `IndexTaskHistory` 中。`AddLog` 方法会确保每个任务类型下的日志列表按触发时间戳（`_triggerTimestampInSec`）降序排列，即最新的任务日志在最前面。
-   **历史记录裁剪**: 为了防止历史记录无限增长，`AddLog` 方法在添加新日志后，会检查列表大小是否超过 `_maxReserveLogSize`（默认为 10）。如果超过，则会裁剪掉最旧的日志。这是一种简单有效的滚动日志策略。
-   **配置变更追溯**: `IndexTaskHistory` 还包含一个 `_taskConfigMetas` 列表，用于记录任务配置的变更历史。`DeclareTaskConfig` 方法会在任务配置发生变化时，记录下任务名称、类型和变更生效的时间戳。这对于排查因配置变更导致的行为异常问题非常有帮助。
-   **序列化**: `IndexTaskHistory` 实现了 `Jsonize` 方法，可以轻松地将其整体序列化为 JSON 字符串或从字符串中反序列化。这使得历史记录的持久化和传输变得非常简单。

## 5. 技术风险与考量

-   **监控数据粒度**: `IndexOperationMetrics` 提供了操作级别的耗时，但在复杂的 `IndexOperation` 内部，可能还有更细粒度的步骤（如读文件、写文件、计算等）。如果需要更深入的性能分析，目前的度量粒度可能不够，需要在具体 `Operation` 实现中添加更多的自定义指标。
-   **历史记录大小**: `_maxReserveLogSize` 的大小是一个权衡。设置得太小，可能丢失有用的历史信息；设置得太大，则会增加 `IndexTaskHistory` 对象的体积，影响其序列化和存储开销。对于需要长期追溯的场景，可能需要将历史记录导出到外部的日志分析系统。
-   **`task_description` 的滥用**: `IndexTaskLog` 中的 `_taskDescription` 是一个灵活的 `map`，可以存储任意的键值对。虽然灵活，但也可能被滥用，存入过多或过大的数据，导致历史记录膨胀。需要有规范来指导哪些信息适合放入 `description`。

## 6. 总结

Indexlib 的监控与历史追溯机制，通过 `Metrics` 和 `History` 两条线，为任务框架提供了出色的可观测性。`Metrics` 体系与 `kmonitor` 紧密集成，提供了强大、多维度的实时性能监控能力。`History` 体系则通过结构化的日志，为事后问题排查和行为审计提供了坚实的数据基础。这套机制的设计体现了生产级系统对稳定性、可靠性和可维护性的高度重视，是 Indexlib 能够胜任大规模、长时间稳定运行的重要基石。
