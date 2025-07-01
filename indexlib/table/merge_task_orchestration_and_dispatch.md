
# Indexlib Table 合并任务编排与分发

**涉及文件:**
* `indexlib/table/merge_task_dispatcher.h`
* `indexlib/table/merge_task_dispatcher.cpp`
* `indexlib/table/merge_task_description.h`
* `indexlib/table/merge_task_description.cpp`
* `indexlib/table/task_execute_meta.h`
* `indexlib/table/task_execute_meta.cpp`

## 1. 系统概述

在 Indexlib 的合并流程中，当 `MergePolicy` 将一个大的合并计划（`TableMergePlan`）分解为多个独立的合并任务（`MergeTaskDescription`）之后，就需要一个**编排与分发**机制，将这些任务合理地分配给不同的执行单元（例如，不同的机器或进程），以实现并行化，提高整体的合并效率。`table` 模块中的**合并任务编排与分发**组件，正是为了解决这一问题而设计的。

本篇文档将深入分析 `MergeTaskDispatcher` 及其相关的数据结构，阐述其如何对合并任务进行负载均衡的分发，揭示其背后的算法和设计考量。

## 2. 从 `MergeTaskDescription` 到 `TaskExecuteMeta`

在深入 `MergeTaskDispatcher` 之前，我们首先需要理解两个关键的数据结构：`MergeTaskDescription` 和 `TaskExecuteMeta`。

*   **`MergeTaskDescription`**: 这个结构体由 `MergePolicy` 生成，用于描述一个具体的合并任务。它主要包含三个信息：
    *   `name`: 任务的名称，用于标识和调试。
    *   `cost`: 任务的预估开销。这个 `cost` 是一个抽象的、相对的值，它可以代表任务所需的时间、CPU、内存等资源的综合评估。`MergePolicy` 在生成任务时，会根据 Segment 的大小、文档数量等因素，估算出这个 `cost`。
    *   `parameters`: 一个 `KeyValueMap`，用于存储一些任务相关的自定义参数。

*   **`TaskExecuteMeta`**: 这个结构体是对 `MergeTaskDescription` 的一层封装，专门用于任务的分发和执行。它除了包含 `cost` 之外，还增加了两个重要的字段：
    *   `planIdx`: 任务所属的 `TableMergePlan` 的索引。
    *   `taskIdx`: 任务在 `TableMergePlan` 的任务列表中的索引。

通过 `planIdx` 和 `taskIdx`，系统可以唯一地定位到一个 `MergeTaskDescription`。`TaskExecuteMeta` 还提供了一个 `GetIdentifier` 方法，用于生成一个全局唯一的字符串标识符，方便任务的跟踪和管理。

## 3. `MergeTaskDispatcher`：负载均衡的任务分发器

`MergeTaskDispatcher` 是任务分发的核心。它的主要职责是接收一组 `MergeTaskDescription`，并将它们分发给指定数量的执行实例（`instanceCount`），同时尽可能地保证每个实例的**总负载（`workLoad`）**是均衡的。

### 关键设计与算法

`MergeTaskDispatcher` 的 `DispatchMergeTasks` 方法实现了其核心的负载均衡算法。该算法可以概括为以下几个步骤：

1.  **收集所有任务**: 首先，将所有 `TableMergePlan` 中的所有 `MergeTaskDescription`，都转换为 `TaskExecuteMeta`，并放入一个总的任务列表 `allTaskMetas` 中。

2.  **按开销降序排序**: 对 `allTaskMetas` 列表中的所有任务，按照其 `cost` 字段进行**降序**排序。这意味着，开销最大的任务会被排在最前面。

3.  **初始化工作负载计数器**: 创建一个最小堆（`priority_queue`），用于存放 `WorkLoadCounter` 对象。`WorkLoadCounter` 是一个内部类，它记录了每个执行实例当前分配到的任务列表（`taskMetas`）和总的工作负载（`workLoad`）。堆的大小等于 `instanceCount`，并且堆的排序规则是**按 `workLoad` 升序**排列。这意味着，当前总负载最小的执行实例，会位于堆顶。

4.  **贪心分发**: 遍历排好序的 `allTaskMetas` 列表，对于每一个任务（从开销最大的开始），执行以下操作：
    a.  从最小堆的堆顶，取出一个 `WorkLoadCounter`（即当前总负载最小的那个执行实例）。
    b.  将当前任务分配给这个 `WorkLoadCounter`，即将任务的 `TaskExecuteMeta` 添加到其 `taskMetas` 列表中，并将其 `cost`累加到 `workLoad` 上。
    c.  将更新后的 `WorkLoadCounter` 重新插入到最小堆中。

5.  **返回结果**: 当所有任务都分发完毕后，`taskExecMetas` 中就保存了每个执行实例应该执行的任务列表。

这种“**将最重的任务，优先分配给最闲的工人**”的贪心策略，是一种经典的长任务优先（Longest-Processing-Time-First, LPT）调度算法的变种。它虽然不能保证得到绝对最优的解，但在大多数情况下，都能给出一个相当不错的、近似最优的负载均衡结果，并且算法的实现相对简单、高效。

### 核心代码示例 (`merge_task_dispatcher.cpp`)

```cpp
vector<TaskExecuteMetas> MergeTaskDispatcher::DispatchMergeTasks(const vector<MergeTaskDescriptions>& taskDescriptions,
                                                                 uint32_t instanceCount)
{
    std::vector<TaskExecuteMetas> taskExecMetas;
    if (taskDescriptions.size() == 0) {
        return taskExecMetas;
    }
    // ... 初始化 taskExecMetas ...

    TaskExecuteMetas allTaskMetas;
    // ... 将所有 taskDescriptions 转换为 allTaskMetas ...

    // sort allTaskMetas by cost in descending order
    auto CompByCost = [](const TaskExecuteMeta& lhs, const TaskExecuteMeta& rhs) { return lhs.cost > rhs.cost; };
    sort(allTaskMetas.begin(), allTaskMetas.end(), CompByCost);

    // make a min-heap, WorkloadCounter with lowest workLoad on top
    priority_queue<WorkLoadCounterPtr, std::vector<WorkLoadCounterPtr>, WorkLoadComp> totalLoadHeap;
    // ... 初始化 totalLoadHeap ...

    // iterate all taskMetas, and assign task with largest cost
    // to the WorkLoadCounter with lowest total load
    for (const auto& taskMeta : allTaskMetas) {
        WorkLoadCounterPtr counterOnTop = totalLoadHeap.top();
        totalLoadHeap.pop();
        counterOnTop->taskMetas.push_back(taskMeta);
        counterOnTop->workLoad += taskMeta.cost;
        totalLoadHeap.push(counterOnTop);
    }
    return taskExecMetas;
}
```

## 4. 技术考量与设计权衡

*   **`cost` 估算的准确性**: `MergeTaskDispatcher` 的负载均衡效果，在很大程度上依赖于 `MergePolicy` 对 `cost` 估算的准确性。如果 `cost` 估算得不准，可能会导致某些实例的实际负载远高于其他实例，从而影响整体的合并效率。因此，在实现自定义的 `MergePolicy` 时，需要仔细设计 `cost` 的计算逻辑。

*   **算法的普适性**: LPT 调度算法是一种通用的、与具体任务无关的算法。这使得 `MergeTaskDispatcher` 可以被用于各种不同类型的 Table，具有很好的普适性。然而，对于某些特殊的业务场景，可能会有更优的、与业务逻辑相关的调度算法。在这种情况下，可以考虑实现一个自定义的 `MergeTaskDispatcher`。

*   **任务的异构性**: 当前的 `MergeTaskDispatcher` 主要考虑了任务的 `cost`（即负载），但没有考虑任务的其他异构性，例如某些任务可能需要特殊的硬件资源（如 GPU）。如果未来有这样的需求，需要在 `MergeTaskDescription` 和调度算法中引入更多的维度。

## 5. 总结

Indexlib 的**合并任务编排与分发**机制，通过 `MergeTaskDispatcher` 和一系列精心设计的数据结构，成功地将复杂的合并任务，高效、均衡地分配给多个执行实例。其核心的 LPT 调度算法，在保证了较好负载均衡效果的同时，也兼顾了实现的简洁性和通用性。这套机制是 Indexlib 实现高效率、并行化合并的关键所在，为整个系统的可扩展性和性能提供了有力的保障。
