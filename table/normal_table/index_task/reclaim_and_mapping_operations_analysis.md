
# Indexlib 空间回收与映射操作（Reclaim & Mapping Operations）深度解析

**涉及文件:**
*   `table/normal_table/index_task/NormalTableReclaimOperation.cpp`
*   `table/normal_table/index_task/NormalTableReclaimOperation.h`
*   `table/normal_table/index_task/ReclaimMapOperation.cpp`
*   `table/normal_table/index_task/ReclaimMapOperation.h`
*   `table/normal_table/index_task/BucketMapOperation.cpp`
*   `table/normal_table/index_task/BucketMapOperation.h`
*   `table/normal_table/index_task/PrepareIndexReclaimParamOperation.cpp`
*   `table/normal_table/index_task/PrepareIndexReclaimParamOperation.h`

## 1. 概述：合并的“导航图”生成器

在 Indexlib 的合并（Merge）流程中，如果说索引合并操作（Index Merge Operations）是执行物理数据迁移的“建筑工人”，那么本文档所解析的空间回收与映射操作（Reclaim & Mapping Operations）就是绘制“建筑蓝图”的“设计师”。这些操作的核心使命是生成合并过程中至关重要的“导航图”——`ReclaimMap` 和 `BucketMap`，它们精确地指导了后续所有索引数据如何从旧的 `Segment` 迁移和重组到新的 `Segment` 中。

这一系列操作通常在物理合并之前执行，它们通过分析现有的 `Segment`、处理删除的文档、应用排序规则等，计算出最优的文档重排方案。这个方案不仅要保证数据的完整性，还要通过优化数据布局来提升查询性能和空间利用率。可以说，映射操作的质量直接决定了合并任务的最终成败和收益。

### 1.1. 设计理念：分离决策与执行

该系列操作的设计完美体现了“决策与执行分离”的原则。

*   **决策（Mapping）**: `ReclaimMapOperation` 和 `BucketMapOperation` 等负责制定决策。它们不直接操作或迁移索引数据，而是专注于计算和生成映射关系。`ReclaimMap` 负责定义哪些文档被保留以及它们在新 `Segment` 中的新位置（`new_docid`），而 `BucketMap` 则在需要排序时，进一步定义文档在排序桶（Bucket）内的局部顺序。
*   **执行（Merging）**: 后续的 `InvertedIndexMergeOperation`、`AttributeIndexMergeOperation` 等则完全依赖这些映射关系来执行物理数据的拷贝和重排。

这种分离带来了巨大的好处：
1.  **清晰性**: 每个组件的职责更加单一，逻辑更清晰。
2.  **可复用性**: 生成的 `ReclaimMap` 可以被所有类型的索引合并操作共享，避免了重复计算。
3.  **可测试性**: 可以独立地对映射逻辑进行单元测试，确保其正确性。

### 1.2. 核心产物：`ReclaimMap` 与 `BucketMap`

*   **`ReclaimMap`**: 这是最核心的产物。它是一个数据结构，存储了从 `(source_segment_id, old_docid)` 到 `(target_segment_id, new_docid)` 的映射。任何在 `ReclaimMap` 中没有对应 `new_docid` 的文档，都意味着它在合并过程中被“回收”（即抛弃）。
*   **`BucketMap`**: 当合并需要按照特定字段排序时，`BucketMap` 会被使用。它将所有待合并的文档根据排序键（Sort Key）的值划分到不同的“桶”中，并记录每个文档在桶内的相对顺序。后续的合并操作会先按桶的顺序，再按桶内顺序来写入数据，从而实现全局排序。

## 2. 核心映射操作详解

### 2.1. `ReclaimMap` 生成 (`ReclaimMapOperation`)

`ReclaimMapOperation` 是生成核心 `ReclaimMap` 的执行单元。

**功能目标**: 综合考虑删除的文档、合并计划以及可选的排序需求，生成一个完整的 `ReclaimMap`，指导后续的合并。

**核心逻辑与算法**:
1.  **初始化 `ReclaimMap`**: 创建一个 `ReclaimMap` 对象，其初始大小根据合并计划中涉及的总文档数来设定。
2.  **加载删除信息**: 遍历合并计划（`MergePlan`）中的所有源 `Segment`，加载它们的 `DeletionMap`，从而知道哪些文档是已经被删除的。
3.  **遍历与填充**: 遍历所有源 `Segment` 中的每一个文档（从 `docid = 0` 到 `max_docid`）。
    *   如果一个文档在 `DeletionMap` 中被标记为已删除，则跳过。
    *   如果文档是有效的，则为它分配一个新的 `docid`，并将这个映射关系 `(old_docid -> new_docid)` 添加到 `ReclaimMap` 中。
4.  **处理排序 (`BucketMap`)**: 如果配置了排序合并，`ReclaimMapOperation` 的逻辑会变得更复杂。它会与 `BucketMap` 协同工作。`BucketMap` 首先根据排序键对所有有效文档进行排序，然后 `ReclaimMapOperation` 会按照 `BucketMap` 排序后的顺序来分配 `new_docid`，从而确保最终生成的数据是全局有序的。
5.  **持久化**: 生成的 `ReclaimMap` 对象会被放入 `IndexTaskContext` 中，供后续操作使用。

**关键实现细节**:

`ReclaimMapOperation::Execute` 是其核心入口。

```cpp
Status ReclaimMapOperation::Execute(const IndexTaskContext& context)
{
    // ... 获取 MergePlan, Schema 等 ...

    auto reclaimMap = std::make_shared<index::ReclaimMap>();
    RETURN_IF_STATUS_ERROR(reclaimMap->Init(plan, context.GetTabletSchema(), context.GetResource<BucketMap>(BUCKET_MAP)),
                           "init reclaim map failed");

    // ... 将 reclaimMap 放入 context ...
    context.SetResource(RECLAIM_MAP, reclaimMap);
    return Status::OK();
}
```
真正的核心逻辑被封装在 `ReclaimMap::Init` 方法中。这种设计使得 `Operation` 本身非常轻量，只负责流程的编排。

```cpp
// ReclaimMap::Init 伪代码逻辑
Status ReclaimMap::Init(const MergePlan& plan, const Schema& schema, const BucketMap* bucketMap) {
    // 1. 根据 plan 计算总文档数和新 Segment 数量，分配内部存储
    // 2. 加载所有源 Segment 的 DeletionMap

    if (bucketMap) { // 排序情况
        // 3a. 按照 bucketMap 提供的顺序遍历文档
        // 4a. 为每个文档分配 new_docid 并填充映射
    } else { // 非排序情况
        // 3b. 依次遍历每个源 Segment 的所有文档
        // 4b. 如果文档未被删除，则分配 new_docid 并填充映射
    }
    // 5. 计算并设置每个目标 Segment 的文档数
    return Status::OK();
}
```

**技术风险**:
*   **内存占用**: `ReclaimMap` 需要为所有参与合并的有效文档存储一个映射条目。在涉及海量文档的合并任务中，`ReclaimMap` 本身可能会占用非常大的内存空间。需要精确计算内存需求，并受 `MemoryQuotaController` 的监控。
*   **性能**: `ReclaimMap` 的生成过程需要遍历所有文档并检查其删除状态，这本身就是一个耗时的操作。如果再加上排序，性能开销会更大。

### 2.2. 排序桶映射 (`BucketMapOperation`)

当需要按某个或某些字段对合并后的数据进行排序时，`BucketMapOperation` 就派上了用场。

**功能目标**: 为所有待合并的有效文档，根据其排序键的值，分配到一个“桶”（Bucket）中，并确定其在桶内的相对顺序。

**核心逻辑与算法**:
1.  **获取排序配置**: 从 `MergeConfig` 中读取排序描述（`SortDescription`），了解需要按哪些字段、以何种顺序（升序/降序）进行排序。
2.  **创建 `BucketMap`**: 实例化一个 `BucketMap` 对象。
3.  **收集文档**: `BucketMap` 会遍历所有源 `Segment` 的有效文档（跳过已删除的）。
4.  **读取排序键**: 对于每个有效文档，它会从对应的正排索引（Attribute Index）中读取其排序键的值。
5.  **排序**: 将所有收集到的 `(文档信息, 排序键)` 对进行排序。这是一个典型的多键值排序问题，排序算法的效率至关重要。
6.  **填充 `BucketMap`**: 排序完成后，将有序的文档列表存入 `BucketMap` 内部的数据结构中。
7.  **持久化**: 将生成的 `BucketMap` 放入 `IndexTaskContext`，供 `ReclaimMapOperation` 使用。

**关键实现细节**:

```cpp
Status BucketMapOperation::Execute(const IndexTaskContext& context)
{
    // ... 获取 MergePlan, Schema, SortDescriptions ...
    if (sortDescs.empty()) {
        // 不需要排序，直接返回
        return Status::OK();
    }

    auto bucketMap = std::make_shared<index::BucketMap>();
    RETURN_IF_STATUS_ERROR(bucketMap->Init(plan, sortDescs),
                           "init bucket map failed");

    context.SetResource(BUCKET_MAP, bucketMap);
    return Status::OK();
}
```
`BucketMap::Init` 内部封装了排序的复杂逻辑，包括创建用于读取排序键的 `AttributeReader`、执行排序算法等。

**技术风险**:
*   **性能瓶颈**: 排序是计算密集型和内存密集型操作。当文档数量巨大时，排序过程可能会成为整个合并流程的性能瓶颈。需要高效的排序算法（例如，基于基数排序或快速排序的变种）和足够的内存。
*   **数据类型支持**: 需要能正确处理各种数据类型的排序键，包括数值、字符串等，并能正确处理多字段排序的逻辑。

### 2.3. 表级别空间回收 (`NormalTableReclaimOperation`)

这是一个更高层次的封装操作，它协调了上述多个原子操作，以完成一个完整的表级别空间回收（Reclaim）任务。

**功能目标**: 对一个 `NormalTable` 执行一次完整的空间回收，包括生成映射、合并各类索引、更新元信息等。

**核心逻辑与算法**:
`NormalTableReclaimOperation` 本身不包含复杂的算法逻辑，它更像一个“任务编排器”。它的 `Execute` 方法会按照预定义的顺序，依次创建并执行一系列底层的 `Operation`。

**关键实现细节**:

```cpp
Status NormalTableReclaimOperation::Execute(const IndexTaskContext& context)
{
    // 1. 准备阶段
    auto paramOp = std::make_shared<PrepareIndexReclaimParamOperation>();
    RETURN_IF_STATUS_ERROR(paramOp->Execute(context), "execute prepare reclaim param op failed");

    // 2. 生成 BucketMap (如果需要排序)
    auto bucketMapOp = std::make_shared<BucketMapOperation>();
    RETURN_IF_STATUS_ERROR(bucketMapOp->Execute(context), "execute bucket map op failed");

    // 3. 生成 ReclaimMap
    auto reclaimMapOp = std::make_shared<ReclaimMapOperation>();
    RETURN_IF_STATUS_ERROR(reclaimMapOp->Execute(context), "execute reclaim map op failed");

    // 4. 依次执行各种索引的合并操作
    // InvertedIndex, Attribute, Source, Summary, DeletionMap ...
    auto invertedMergeOp = std::make_shared<InvertedIndexMergeOperation>();
    RETURN_IF_STATUS_ERROR(invertedMergeOp->Execute(context), "execute inverted merge op failed");
    // ... 其他合并操作 ...

    // 5. 结束阶段，可能包括提交元信息等

    return Status::OK();
}
```
这个 `Execute` 方法清晰地展示了合并任务的完整流程：**准备 -> 计算映射 -> 物理合并**。

**技术风险**:
*   **流程健壮性**: 由于编排了多个子操作，任何一个子操作的失败都可能导致整个任务中断。需要完善的错误处理和回滚机制（尽管在这里更多的是通过任务失败来中断）。
*   **配置依赖**: `NormalTableReclaimOperation` 的行为高度依赖于 `TabletOptions` 中的合并配置。配置的正确性直接关系到任务能否按预期执行。

## 3. 架构思考与展望

### 3.1. 设计的精妙之处

将映射计算与物理合并分离，并通过一个高层 `Operation` (`NormalTableReclaimOperation`) 进行编排，是 Indexlib 合并框架设计的点睛之笔。这种分层、解耦的架构带来了极佳的系统设计特性：

*   **关注点分离**: 每个组件都只做一件事，并把它做好。
*   **可组合性**: 可以通过调整 `NormalTableReclaimOperation` 的编排逻辑，或者替换某个子 `Operation` 的实现，来轻松实现不同的合并策略。
*   **可观测性**: 可以对每个子操作的耗时、资源消耗进行独立的监控，从而更容易定位性能瓶颈。

### 3.2. 未来可能的优化方向

*   **自适应映射策略**: 当前的映射生成逻辑主要基于静态配置。未来可以探索更智能的自适应策略。例如，系统可以根据索引的访问模式、数据的稀疏性等特征，动态决定是否需要排序、如何选择排序键，甚至生成更复杂的 `ReclaimMap` 来优化特定查询场景下的缓存命中率。
*   **并行化计算**: `BucketMap` 和 `ReclaimMap` 的生成过程，特别是文档遍历和排序阶段，存在很大的并行化潜力。可以通过分段处理、并行排序等技术，充分利用多核CPU资源，大幅缩短映射计算的时间。
*   **内存优化**: 对于超大规模的合并任务，可以研究内存占用更低的 `ReclaimMap` 实现，例如通过分段加载、压缩存储等方式，在不显著影响性能的前提下，降低内存峰值。

## 4. 结论

空间回收与映射操作是 Indexlib 合并流程中承上启下的关键一环。它们通过精密的计算，为后续的数据重排铺平了道路。`ReclaimMapOperation` 和 `BucketMapOperation` 分别解决了“哪些文档要保留、去哪里”和“如何排序”的核心问题，而 `NormalTableReclaimOperation` 则将这些原子能力有机地组织起来，形成了一个完整、健壮的表级别空间回收解决方案。

理解这一系列操作的设计哲学和实现细节，对于掌握 Indexlib 的合并机制、诊断性能问题以及进行深度定制开发，都具有不可替代的价值。
