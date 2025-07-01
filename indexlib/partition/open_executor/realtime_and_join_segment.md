
# Indexlib Open Executor 实时索引与 Join 段处理机制分析

**涉及文件:**
* `indexlib/partition/open_executor/reclaim_rt_index_executor.h`
* `indexlib/partition/open_executor/reclaim_rt_index_executor.cpp`
* `indexlib/partition/open_executor/reclaim_rt_segments_executor.h`
* `indexlib/partition/open_executor/reclaim_rt_segments_executor.cpp`
* `indexlib/partition/open_executor/generate_join_segment_executor.h`
* `indexlib/partition/open_executor/generate_join_segment_executor.cpp`
* `indexlib/partition/open_executor/prejoin_executor.h`
* `indexlib/partition/open_executor/prejoin_executor.cpp`
* `indexlib/partition/open_executor/realtime_index_recover_executor.h`
* `indexlib/partition/open_executor/realtime_index_recover_executor.cpp`

## 1. 系统概述

在 Indexlib 中，为了实现数据的近实时（Near Real-Time）可见，除了全量索引外，系统还维护了实时（Realtime, RT）索引。`Reopen` 过程的一个核心任务就是将新的全量索引版本与当前的实时索引进行合并与衔接。这一系列复杂的任务被分解到多个专门的 `Executor` 中，主要包括**实时索引回收（Reclaim）**、**Join 段生成**以及**实时索引恢复**。

- **Reclaim (回收)**: `Reopen` 时，新的全量版本（`IncVersion`）已经包含了部分原实时索引中的数据。因此，需要根据一个时间戳（`reclaimTimestamp`）来清理掉实时索引中那些“已过时”的数据和段，避免数据冗余和资源浪费。`ReclaimRtIndexExecutor` 和 `ReclaimRtSegmentsExecutor` 负责此任务。

- **Join (连接)**: 清理后的实时索引需要与新的全量索引进行“连接”，生成一个临时的 `JoinSegment`，这个段在逻辑上将两者合并，使得 `Reader` 可以透明地访问全量和实时数据。`PrejoinExecutor` 和 `GenerateJoinSegmentExecutor` 负责此任务。

- **Recover (恢复)**: 在某些异常场景下，例如服务重启后，需要根据操作日志（Operation Log）来恢复实时索引的状态。`RealtimeIndexRecoverExecutor` 用于处理这种情况。

这些执行器共同构成了 Indexlib 标准 `Reopen` 流程的核心数据处理部分，确保了数据从“实时”到“全量”的平滑过渡。

## 2. 核心组件与工作流程

### 2.1. ReclaimRtIndexExecutor & ReclaimRtSegmentsExecutor: 实时数据回收

这两个执行器目标相似，都是清理过时的实时数据，但侧重点和应用场景略有不同。

#### 功能目标
- **`ReclaimRtIndexExecutor`**: 主要用于标准 `Reopen` 流程。它会创建一个全新的 `BuildingPartitionData`，并根据 `reclaimTimestamp` 从旧的 `PartitionData` 中选择性地加载仍然有效的实时段，抛弃无效段。它处理的是整个实时索引的状态。
- **`ReclaimRtSegmentsExecutor`**: 主要用于 KV/KKV 表的 `Reopen` 或一些轻量级场景。它更侧重于对段列表的直接操作，如裁剪（Trim）掉过时和空的实时段，以及根据需要回收仍在构建中的段（Building Segment）。

#### 核心逻辑: ReclaimRtIndexExecutor

```cpp
// indexlib/partition/open_executor/reclaim_rt_index_executor.cpp
bool ReclaimRtIndexExecutor::Execute(ExecutorResource& resource)
{
    // ...
    OnlineJoinPolicy joinPolicy(resource.mIncVersion, resource.mRtSchema->GetTableType(), resource.mSrcSignature);
    int64_t reclaimTimestamp = joinPolicy.GetReclaimRtTimestamp();

    if (mIsInplaceReclaim) {
        // 优化路径：在分支上直接回收
        reclaimer.reset(new BranchPartitionDataReclaimer(resource.mRtSchema, resource.mOptions));
        reclaimer->Reclaim(reclaimTimestamp, resource.mBranchPartitionDataHolder.Get());
    } else {
        // 标准路径：创建全新的 PartitionData
        resource.mPartitionDataHolder.Reset(PartitionDataCreator::CreateBuildingPartitionData(...));
        reclaimer.reset(new RealtimePartitionDataReclaimer(resource.mRtSchema, resource.mOptions));
        reclaimer->Reclaim(reclaimTimestamp, resource.mPartitionDataHolder.Get());
    }
    // ...
    return true;
}
```
1.  **计算回收时间戳**: 通过 `OnlineJoinPolicy` 计算出 `reclaimTimestamp`。这个时间戳通常与新全量版本的 `timestamp` 相关，标志着一个分界线：早于此时间戳的实时数据理论上已包含在新全量中。
2.  **执行回收**: `RealtimePartitionDataReclaimer::Reclaim` 是核心逻辑。它会遍历旧 `PartitionData` 的实时段目录，只将那些 `timestamp` 大于等于 `reclaimTimestamp` 的段加载到新的 `PartitionData` 中。
3.  **原地回收 (`mIsInplaceReclaim`)**: 这是一个优化选项，用于在“优化 `Reopen`”流程中。它直接在 `BranchPartitionData` 上进行回收，而不是创建一个全新的 `PartitionData`，减少了开销。
4.  **回滚 (`Drop`)**: 如果后续执行器失败，`Drop` 方法会通过 `resource.mPartitionDataHolder.Reset(mOriginalPartitionData)` 将 `PartitionData` 恢复到回收前的状态，保证了操作的原子性。

### 2.2. PrejoinExecutor & GenerateJoinSegmentExecutor: Join 段的生成

`JoinSegment` 是一个逻辑概念，它并不产生一个物理上完整的段，而是生成一些元数据（如 `segment_info`），记录下需要连接的实时段信息。`Reader` 在加载时，会根据这些信息将实时段和全量段统一起来对外服务。

#### 功能目标
- **`PrejoinExecutor`**: 在 `Reopen` 流程的早期、无锁阶段执行 `Join` 的预处理工作。这部分工作通常是 CPU 密集型且不涉及对共享状态的修改，例如读取实时数据并构建用于 `Join` 的内存结构。提前执行可以减少主锁的持有时间。
- **`GenerateJoinSegmentExecutor`**: 在 `Reopen` 流程的后期、持锁阶段完成 `Join` 的剩余工作并最终生成（Dump）`JoinSegment` 的元数据文件。

#### 核心逻辑

`Join` 的核心逻辑被封装在 `JoinSegmentWriter` 中，这两个 `Executor` 只是在 `Reopen` 流程的不同阶段调用了 `JoinSegmentWriter` 的不同方法。

```cpp
// indexlib/partition/open_executor/prejoin_executor.cpp
bool PrejoinExecutor::Execute(ExecutorResource& resource)
{
    // 调用 JoinSegmentWriter 的预处理方法
    bool ret = mJoinSegWriter->PreJoin();
    mJoinSegWriter.reset(); // PreJoin 后 writer 的使命已完成一部分
    return ret;
}
```

```cpp
// indexlib/partition/open_executor/generate_join_segment_executor.cpp
bool GenerateJoinSegmentExecutor::Execute(ExecutorResource& resource)
{
    // ...
    // 创建一个新的 BuildingPartitionData 用于存放 Join 后的结果
    resource.mPartitionDataHolder.Reset(PartitionDataCreator::CreateBuildingPartitionData(...));
    
    // 调用 JoinSegmentWriter 的 Join 和 Dump 方法
    bool ret = mJoinSegWriter->Join() && mJoinSegWriter->Dump(resource.mPartitionDataHolder.Get());
    mJoinSegWriter.reset();
    // ...
    return ret;
}
```

1.  **分离式执行**: `OpenExecutorChainCreator` 创建一个 `JoinSegmentWriter` 实例，并将其同时传递给 `PrejoinExecutor` 和 `GenerateJoinSegmentExecutor`。`PrejoinExecutor` 在前台无锁阶段调用 `PreJoin()`，`GenerateJoinSegmentExecutor` 在后台持锁后调用 `Join()` 和 `Dump()`。
2.  **状态创建**: `GenerateJoinSegmentExecutor` 会创建一个新的 `BuildingPartitionData`，`JoinSegmentWriter::Dump` 会将生成的 `JoinSegment` 信息写入这个新的 `PartitionData` 中。
3.  **回滚 (`Drop`)**: `GenerateJoinSegmentExecutor` 的 `Drop` 方法同样重要，它需要将 `PartitionData` 恢复到 `Join` 之前的状态，并回滚 `JoinSegmentDirectory` 的版本，确保操作的原子性。

### 2.3. RealtimeIndexRecoverExecutor: 实时索引恢复

这个执行器用于系统启动时的 `Open` 流程，特别是当检测到有未处理的实时数据或操作时。

#### 功能目标
- 从上次安全退出的点开始，重放（Redo）操作队列（Operation Queue）中的操作（如 `add`, `delete`, `update`），以恢复内存中实时索引的状态。
- 主要针对普通索引表（`tt_index`），因为 KV/KKV 表有自己的恢复机制。

#### 核心逻辑

```cpp
// indexlib/partition/open_executor/realtime_index_recover_executor.cpp
bool RealtimeIndexRecoverExecutor::Execute(ExecutorResource& resource)
{
    // ...
    // KV/KKV 表直接跳过
    auto tableType = resource.mSchema->GetTableType();
    if (tableType == tt_kkv || tableType == tt_kv) {
        return true;
    }
    // ...
    // 创建一个临时的 Reader 和 Modifier 用于应用操作
    auto realtimeReader = ReleasePartitionReaderExecutor::CreateRealtimeReader(resource);
    PartitionModifierPtr modifier = PartitionModifierCreator::CreateInplaceModifier(resource.mSchema, realtimeReader);
    
    // 创建 OperationReplayer
    OperationReplayer replayer(resource.mPartitionDataHolder.Get(), resource.mSchema, memController, false);
    // 使用特定的恢复策略
    OperationRedoStrategyPtr redoStrategy(new RecoverRtOperationRedoStrategy(realtimeReader->GetVersion()));
    
    // 执行重放
    bool ret = replayer.RedoOperations(modifier, index_base::Version(INVALID_VERSIONID), redoStrategy);
    // ...
    return ret;
}
```
1.  **创建 `Modifier`**: `Redo` 操作需要一个 `PartitionModifier` 来将变更应用到索引上。
2.  **创建 `Replayer` 和 `Strategy`**: `OperationReplayer` 是执行重放的核心类。`RecoverRtOperationRedoStrategy` 提供了恢复场景下的特定逻辑，例如它会根据 `Reader` 的版本来决定从哪里开始重放，避免重复应用操作。
3.  **执行 `Redo`**: `replayer.RedoOperations` 会读取操作队列并应用到 `modifier` 上，直到队列处理完毕。

## 3. 技术风险与考量

1.  **`reclaimTimestamp` 的精确性**: `Reclaim` 逻辑的正确性完全依赖于 `reclaimTimestamp`。如果这个时间戳计算错误（例如，过于保守或过于激进），可能导致数据丢失（清理了不该清理的实时数据）或数据冗余（保留了本应被新全量版本覆盖的数据）。`OnlineJoinPolicy` 的实现是这里的关键和难点。

2.  **Join 的性能**: `JoinSegmentWriter` 的 `PreJoin` 和 `Join` 过程可能涉及大量的数据处理，尤其是在实时数据量很大的情况下。这可能成为 `Reopen` 过程中的一个性能瓶颈。将其分为 `PreJoin` 和 `Join` 两个阶段，并将 `PreJoin` 移到无锁区，是一种有效的优化手段，但设计和实现的复杂度也相应增加。

3.  **回滚的复杂性**: `GenerateJoinSegmentExecutor` 和 `ReclaimRtIndexExecutor` 都创建了全新的 `PartitionData`。它们的 `Drop` 方法必须能够完美地恢复到原始的 `PartitionData` 状态。这不仅是简单的指针切换，还可能涉及文件系统层面的版本回滚（如 `rtSegmentDir->RollbackToCurrentVersion()`），对事务性的要求非常高。

4.  **恢复的幂等性**: `RealtimeIndexRecoverExecutor` 的 `Redo` 过程必须是幂等的。如果因为某些原因（如中断后重试）重复执行，不应导致数据状态的错乱。`RecoverRtOperationRedoStrategy` 通过版本号等信息来保证从正确的断点开始恢复，是保证幂等性的关键。

## 4. 结论

Indexlib 中处理实时索引和 `Join` 段的 `Executor` 集合，是其实现近实时搜索和保证数据平滑过渡的核心。通过将复杂的流程分解为**回收（Reclaim）**、**预连接（Prejoin）**和**生成（Generate）**等多个独立的、职责清晰的步骤，系统实现了高度的模块化和灵活性。`Reclaim` 机制确保了数据不重不漏，`Join` 机制透明地统一了全量和实时视图，而 `Recover` 机制则保证了系统的容错和恢复能力。这些设计共同构成了 Indexlib 在线 `Reopen` 流程中最为关键和复杂的部分，体现了其在高性能、高可用性方面的深入思考。
