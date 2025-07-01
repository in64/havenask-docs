
# Indexlib Table 合并策略与执行

**涉及文件:**
* `indexlib/table/merge_policy.h`
* `indexlib/table/merge_policy.cpp`
* `indexlib/table/table_merge_plan.h`
* `indexlib/table/table_merge_plan.cpp`
* `indexlib/table/table_merger.h`
* `indexlib/table/table_merger.cpp`
* `indexlib/table/table_merge_plan_meta.h`
* `indexlib/table/table_merge_plan_meta.cpp`
* `indexlib/table/table_merge_plan_resource.h`
* `indexlib/table/table_merge_plan_resource.cpp`

## 1. 系统概述

在 Indexlib 中，为了控制索引的 Segment 数量、优化查询性能并回收已删除或更新的文档所占用的空间，需要定期对 Segment 进行**合并（Merge）**。`table` 模块的**合并策略与执行**机制，正是为了实现这一核心功能而设计的。

本篇文档将深入剖析 `table` 模块中与合并策略及执行相关的代码，阐述其如何决策、规划和执行合并任务，揭示其背后的设计思想和技术实现。

## 2. 合并流程概览

`table` 模块的合并流程可以分为三个主要阶段：

1.  **创建合并计划（Merge Plan）**: `MergePolicy` 根据预设的策略（例如 Segment 的大小、文档数量、删除比例等），从当前的 Segment 列表中筛选出需要被合并的 Segment，并生成一个或多个 `TableMergePlan`。
2.  **创建合并任务（Merge Task）**: `MergePolicy` 进一步将 `TableMergePlan` 分解为一系列具体的 `MergeTaskDescription`。每个 `MergeTaskDescription` 都描述了一个独立的、可执行的合并单元。
3.  **执行合并**: `TableMerger` 负责执行具体的合并任务。它会读取输入 Segment 的数据，进行合并处理，并生成新的 Segment。

下面，我们将详细分析每个阶段涉及的核心组件和实现细节。

## 3. `MergePolicy`：合并决策的核心

`MergePolicy` 是合并策略的制定者，它决定了**何时**、**如何**以及**哪些** Segment 需要被合并。`MergePolicy` 是一个基类，不同的 Table 类型可以根据自身的数据特点和业务需求，实现自定义的 `MergePolicy`。

### 关键设计与实现

*   **初始化 (`Init`)**: `MergePolicy` 的 `Init` 方法接收 `Schema` 和 `Options` 作为参数，以便在制定合并策略时，能够获取到索引的结构信息和配置参数。

*   **创建合并计划 (`CreateMergePlansFor...`)**: `MergePolicy` 提供了两个核心的接口用于创建合并计划：
    *   `CreateMergePlansForFullMerge`: 用于**全量合并**。它会将所有现存的 Segment 合并成一个或少数几个新的 Segment。全量合并通常在离线构建或系统负载较低时进行。
    *   `CreateMergePlansForIncrementalMerge`: 用于**增量合并**。它会根据一定的策略（例如，选择一些较小的、删除比例较高的 Segment）进行合并。增量合并是持续运行的，用于控制 Segment 数量的增长。

*   **创建合并任务描述 (`CreateMergeTaskDescriptions`)**: 在生成 `TableMergePlan` 之后，`MergePolicy` 会调用 `CreateMergeTaskDescriptions` 方法，将合并计划分解为更小粒度的 `MergeTaskDescription`。这种分解使得合并任务可以被**并行化**执行，从而提高合并效率。

*   **合并任务归并 (`ReduceMergeTasks`)**: 当并行的合并任务完成后，`MergePolicy` 的 `ReduceMergeTasks` 方法会被调用。它负责将多个任务的输出结果（例如，多个临时的 Segment 文件）进行整合，最终形成一个完整的、新的 Segment。

#### 核心代码示例 (`merge_policy.h`)

```cpp
class MergePolicy
{
public:
    MergePolicy();
    virtual ~MergePolicy();

public:
    bool Init(const config::IndexPartitionSchemaPtr& schema, const config::IndexPartitionOptions& options);

    virtual std::vector<TableMergePlanPtr> CreateMergePlansForFullMerge(
        const std::string& mergeStrategyStr, const config::MergeStrategyParameter& mergeStrategyParameter,
        const std::vector<SegmentMetaPtr>& allSegmentMetas, const PartitionRange& targetRange) const;

    virtual std::vector<TableMergePlanPtr> CreateMergePlansForIncrementalMerge(
        const std::string& mergeStrategyStr, const config::MergeStrategyParameter& mergeStrategyParameter,
        const std::vector<SegmentMetaPtr>& allSegmentMetas, const PartitionRange& targetRange) const = 0;

    virtual std::vector<MergeTaskDescription>
    CreateMergeTaskDescriptions(const index_base::Version& baseVersion, const TableMergePlanPtr& mergePlan,
                                const TableMergePlanResourcePtr& planResource,
                                const std::vector<SegmentMetaPtr>& inPlanSegmentMetas,
                                MergeSegmentDescription& segmentDescription) const = 0;

    virtual bool ReduceMergeTasks(const TableMergePlanPtr& mergePlan,
                                  const std::vector<MergeTaskDescription>& taskDescriptions,
                                  const std::vector<file_system::DirectoryPtr>& inputDirectorys,
                                  const file_system::DirectoryPtr& outputDirectory, bool isFailOver) const = 0;

    // ... 其他接口 ...
protected:
    config::IndexPartitionSchemaPtr mSchema;
    config::IndexPartitionOptions mOptions;
};
```

## 4. `TableMergePlan` 与相关元数据

`TableMergePlan` 是 `MergePolicy` 决策的结果，它详细描述了一个合并计划。

*   **`TableMergePlan`**: 主要包含了一个 `InPlanSegmentAttributes` 的 `map`，用于记录哪些 Segment 需要被合并，以及这些 Segment 在合并后是否需要被保留（`reservedInNewVersion`）。
*   **`TableMergePlanMeta`**: 存储了合并任务的元数据，例如目标 Segment 的 `locator`、时间戳、`maxTTL` 等。这些信息对于增量构建和数据恢复至关重要。
*   **`TableMergePlanResource`**: 定义了在合并过程中可能需要的一些共享资源。例如，某些 Table 类型可能需要在合并前预加载一些全局词典或数据结构，以提高合并效率。`TableMergePlanResource` 提供了一个统一的接口来管理这些资源的初始化、存储和加载。

## 5. `TableMerger`：合并操作的执行者

`TableMerger` 是合并流程的最终执行者。它负责根据 `MergeTaskDescription`，将输入的 Segment 数据进行合并，并生成新的 Segment。

### 关键设计与实现

*   **初始化 (`Init`)**: `TableMerger` 的 `Init` 方法接收 `Schema`、`Options`、`TableMergePlanResource` 和 `TableMergePlanMeta` 作为参数。它会根据这些信息，为接下来的合并操作做好准备。

*   **内存评估 (`EstimateMemoryUse`)**: 在执行合并之前，系统会调用 `EstimateMemoryUse` 方法来评估该合并任务所需的内存。这有助于进行资源调度和内存控制，防止因内存不足而导致合并失败。

*   **合并 (`Merge`)**: `Merge` 方法是 `TableMerger` 的核心。它接收 `inPlanSegMetas`（待合并的 Segment）和 `taskDescription`（当前任务的描述），并在 `outputDirectory` 中生成合并后的数据。

`TableMerger` 的具体实现与 Table 类型密切相关。例如，KV Table 的 `TableMerger` 需要处理 Key 的冲突和覆盖，而 Normal Table 的 `TableMerger` 则需要对倒排索引进行归并。

#### 核心代码示例 (`table_merger.h`)

```cpp
class TableMerger
{
public:
    TableMerger();
    virtual ~TableMerger();

public:
    virtual bool Init(const config::IndexPartitionSchemaPtr& schema, const config::IndexPartitionOptions& options,
                      const TableMergePlanResourcePtr& mergePlanResources,
                      const TableMergePlanMetaPtr& mergePlanMeta) = 0;

    virtual size_t EstimateMemoryUse(const std::vector<table::SegmentMetaPtr>& inPlanSegMetas,
                                     const table::MergeTaskDescription& taskDescription) const = 0;

    virtual bool Merge(const std::vector<table::SegmentMetaPtr>& inPlanSegMetas,
                       const table::MergeTaskDescription& taskDescription,
                       const file_system::DirectoryPtr& outputDirectory) = 0;
};
```

## 6. 技术考量与设计权衡

*   **策略的灵活性与通用性**: `MergePolicy` 的设计需要在灵活性和通用性之间做出权衡。过于通用的策略可能无法满足特定业务场景的需求，而过于灵活的策略则可能增加用户的理解和使用成本。Indexlib 通过提供一个通用的基类和允许用户自定义子类的方式，较好地解决了这个问题。

*   **合并过程的 I/O 与 CPU**: 合并是一个 I/O 密集型和 CPU 密集型的操作。在设计 `TableMerger` 时，需要充分考虑如何优化数据的读写、如何减少不必要的计算，以提高合并效率。例如，可以使用 `mmap` 来减少文件拷贝，或者使用更高效的算法来归并索引。

*   **容错与恢复**: 合并过程可能会因为各种原因（如磁盘空间不足、机器宕机）而失败。因此，需要有一套完整的容错和恢复机制。Indexlib 通过将合并计划和任务状态持久化，以及支持任务的重试，来保证合并过程的可靠性。

## 7. 总结

Indexlib 的 `table` 模块通过 `MergePolicy`、`TableMergePlan` 和 `TableMerger` 这三个核心组件，构建了一套功能强大、可扩展的合并框架。`MergePolicy` 负责制定合并策略，`TableMergePlan` 描述合并计划，而 `TableMerger` 则负责执行具体的合并操作。这套框架不仅能够有效地控制 Segment 的数量、优化查询性能，还通过插件化的设计，为用户提供了高度的定制能力，使其能够适应各种复杂的业务场景。
