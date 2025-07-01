
# Indexlib 分区数据回收 (Partition Data Reclamation) 代码分析

**涉及文件:**
*   `indexlib/partition/realtime_partition_data_reclaimer_base.cpp`
*   `indexlib/partition/realtime_partition_data_reclaimer_base.h`
*   `indexlib/partition/realtime_partition_data_reclaimer.cpp`
*   `indexlib/partition/realtime_partition_data_reclaimer.h`
*   `indexlib/partition/branch_partition_data_reclaimer.cpp`
*   `indexlib/partition/branch_partition_data_reclaimer.h`

---

## 1. 系统概述

在持续提供服务的在线（Realtime）索引系统中，数据会不断地实时写入。随着时间的推移，系统中会积累大量过期或无效的数据，例如被删除的文档、因更新而废弃的旧版本文档、以及根据业务规则（如保留最近N天的数据）已过期的文档。这些冗余数据不仅会占用宝贵的内存和磁盘资源，还可能因为增加了需要扫描的索引条目而降低查询性能。

为了解决这个问题，Indexlib 设计了**分区数据回收（Partition Data Reclaimer）机制**。该机制的核心目标是定期、自动地清理和回收实时分区中的无效数据，从而保证系统的长期稳定、高效运行。它就像是索引的“垃圾回收器”。

本报告聚焦于实时数据回收的核心实现，主要涉及以下几个类：

*   **`RealtimePartitionDataReclaimerBase`**: 这是一个抽象基类，它定义了数据回收的**核心算法和流程框架**。它采用“模板方法”设计模式，将回收过程分解为几个关键步骤（如“清理过期的段”、“删除过期的文档”），并定义了这些步骤的执行顺序，而将具体的实现细节延迟到子类中。这种设计使得回收算法的核心逻辑得以复用。

*   **`RealtimePartitionDataReclaimer`**: 这是针对**标准实时分区**的默认和主要实现。它继承自 `RealtimePartitionDataReclaimerBase`，并提供了完整的、功能强大的回收策略。当它识别出过期的文档时，它会通过创建一个新的实时段（RT Segment）来“压实”数据，从而物理地移除被删除的文档，并释放其占用的空间。

*   **`BranchPartitionDataReclaimer`**: 这是一个针对**分支分区（Branch Partition）**的特化、轻量级实现。分支分区通常用于索引实验或A/B测试，其生命周期较短，对资源回收的要求也不同。因此，这个实现简化了回收逻辑，例如，它在删除文档时仅在内存中标记，而不会像标准实现那样触发数据的物理“压实”，从而以更低的资源开销完成回收操作。

通过这套体系，Indexlib 为不同应用场景提供了不同粒度和强度的回收策略，确保了系统在各种负载下的健康运行。

---

## 2. 核心设计与实现分析

### 2.1 `RealtimePartitionDataReclaimerBase`：回收算法的模板

`RealtimePartitionDataReclaimerBase` 定义了整个回收流程的骨架，是理解回收机制的入口。

#### 功能目标

*   定义一个统一的 `Reclaim` 接口作为回收任务的入口。
*   实现一个与具体执行细节无关的、通用的回收算法框架。
*   利用特殊的“时间戳索引” (`VIRTUAL_TIMESTAMP_INDEX_NAME`) 来高效地查找需要回收的文档。
*   将具体的执行步骤（如如何应用删除、如何移除段）抽象为纯虚函数，交由子类实现。

#### 核心逻辑与数据结构

`Reclaim` 方法是其核心，完整地展示了回收的算法流程。

```cpp
// indexlib/partition/realtime_partition_data_reclaimer_base.cpp

void RealtimePartitionDataReclaimerBase::Reclaim(int64_t timestamp, const PartitionDataPtr& buildingPartData)
{
    IE_LOG(INFO, "reclaim begin");
    // 步骤 1: 清理过期的和空的实时段
    TrimObsoleteAndEmptyRtSegments(timestamp, buildingPartData);
    // 步骤 2: 删除时间戳早于指定值的实时文档
    RemoveObsoleteRtDocs(timestamp, buildingPartData);
    // 步骤 3: 再次清理，因为步骤2可能导致某些段变空
    TrimObsoleteAndEmptyRtSegments(timestamp, buildingPartData);
    IE_LOG(INFO, "reclaim end");
}
```

这个流程非常清晰：先尝试清理一批明显无用的段，然后执行最核心的文档删除操作，最后再做一次清理。这种设计兼顾了效率和彻底性。

`RemoveObsoleteRtDocs` 的实现是该基类的另一个亮点。它并没有遍历所有文档来检查其时间戳，而是巧妙地利用了一个内置的、名为 `VIRTUAL_TIMESTAMP_INDEX_NAME` 的倒排索引。这个索引的 `key` 就是文档的时间戳，`value` 就是对应的 `docid` 列表。

```cpp
// indexlib/partition/realtime_partition_data_reclaimer_base.cpp

void RealtimePartitionDataReclaimerBase::RemoveObsoleteRtDocs(int64_t reclaimTimestamp,
                                                              const PartitionDataPtr& partitionData)
{
    // ... 获取 VIRTUAL_TIMESTAMP_INDEX_NAME 索引的 reader ...

    std::shared_ptr<KeyIterator> keyIter = indexReader->CreateKeyIterator("");
    PartitionModifierPtr modifier = CreateModifier(partitionData);

    while (keyIter->HasNext()) {
        string strTs;
        std::shared_ptr<PostingIterator> postIter(keyIter->NextPosting(strTs));
        // ... 将 strTs 转换为 int64_t ts ...
        
        if (ts >= reclaimTimestamp) { // 时间戳索引的 key 是有序的，可以提前终止
            break;
        }
        docid_t docId = INVALID_DOCID;
        while ((docId = postIter->SeekDoc(docId)) != INVALID_DOCID) {
            modifier->RemoveDocument(docId); // 将要删除的 docid 记录在 modifier 中
        }
    }
    if (!modifier->IsDirty()) {
        return;
    }
    // 调用子类的具体实现来完成删除操作
    DoFinishRemoveObsoleteRtDocs(modifier, partitionData);
}
```
通过遍历这个特殊索引，可以高效地找到所有时间戳小于 `reclaimTimestamp` 的文档，并调用 `modifier->RemoveDocument()` 将它们标记为删除。真正的删除执行逻辑被封装在纯虚方法 `DoFinishRemoveObsoleteRtDocs` 中。

#### 抽象方法

基类定义了四个纯虚函数，构成了“模板方法”模式的核心：
*   `CreateModifier(...)`: 创建一个用于执行修改操作的 `PartitionModifier`。
*   `DoFinishTrimBuildingSegment(...)`: 如何处理正在构建中的内存段。
*   `DoFinishRemoveObsoleteRtDocs(...)`: 如何应用 `modifier` 中记录的文档删除操作。
*   `DoFinishTrimObsoleteAndEmptyRtSegments(...)`: 如何物理地移除那些被标记为待回收的段。

### 2.2 `RealtimePartitionDataReclaimer`：标准的回收策略

这是为普通在线分区设计的标准实现，它的策略是“空间优先”，即通过数据“压实”来彻底回收空间。

#### 功能目标

*   实现基类定义的所有抽象方法。
*   在删除过期文档后，通过生成一个新的实时段来物理地移除这些文档，从而回收内存。
*   在清理段后，更新版本文件 (`version`)，确保回收结果被持久化。

#### 关键实现：`DoFinishRemoveObsoleteRtDocs`

这个方法的实现是标准回收策略的核心。它没有简单地只在删除图（DeletionMap）中标记文档，而是采取了更积极的策略：

```cpp
// indexlib/partition/realtime_partition_data_reclaimer.cpp

void RealtimePartitionDataReclaimer::DoFinishRemoveObsoleteRtDocs(PartitionModifierPtr modifier,
                                                                  const PartitionDataPtr& partitionData)
{
    // 1. 创建一个新的空内存段，用于存放“压实”后的数据
    InMemorySegmentPtr inMemSegment = partitionData->CreateNewSegment();
    
    // 2. 创建一个 NormalSegmentDumpItem，它封装了将 modifier 中的修改（主要是删除）
    //    应用并转储到一个新段的逻辑。
    NormalSegmentDumpItemPtr segmentDumpItem(new NormalSegmentDumpItem(mOptions, mSchema, ""));
    segmentDumpItem->Init(NULL, partitionData, modifier->GetDumpTaskItem(modifier), FlushedLocatorContainerPtr(), true);
    
    // 3. 执行转储，这会将旧段中未被删除的文档连同新的删除信息一起写入新的 inMemSegment
    segmentDumpItem->Dump();
    
    // 4. 将新生成的段加入到分区数据中，并提交版本
    partitionData->AddBuiltSegment(inMemSegment->GetSegmentId(), inMemSegment->GetSegmentInfo());
    partitionData->CommitVersion();
    inMemSegment->SetStatus(InMemorySegment::DUMPED);
    
    // 5. 再创建一个新的空段，为后续的实时写入做准备
    partitionData->CreateNewSegment();
}
```
这个过程本质上是一次**小型的、在内存中进行的段合并（compaction）**。它将所有旧的实时段中的有效数据（即未被删除的文档）和新的删除信息，合并到了一个全新的实时段中。这样做的好处是，旧段所占用的内存可以被完全释放，实现了彻底的垃圾回收。代价是这个过程会消耗一定的计算和内存资源。

### 2.3 `BranchPartitionDataReclaimer`：轻量级的回收策略

分支分区是用于隔离实验流量的轻量级索引副本。因此，它的回收策略也以“轻量”和“低开销”为目标。

#### 功能目标

*   提供一个资源消耗极低的回收实现。
*   避免标准实现中昂贵的“压实”操作。

#### 关键实现：空实现

`BranchPartitionDataReclaimer` 的实现非常有特点，它的大部分核心方法都是空的。

```cpp
// indexlib/partition/branch_partition_data_reclaimer.cpp

void BranchPartitionDataReclaimer::DoFinishTrimBuildingSegment(int64_t reclaimTimestamp, int64_t buildingTs,
                                                               const index_base::PartitionDataPtr& partData)
{
    // 空实现
}

void BranchPartitionDataReclaimer::DoFinishRemoveObsoleteRtDocs(PartitionModifierPtr modifier,
                                                                const index_base::PartitionDataPtr& partData)
{
    // 空实现
}

void BranchPartitionDataReclaimer::DoFinishTrimObsoleteAndEmptyRtSegments(
    const index_base::PartitionDataPtr& partData, const std::vector<segmentid_t>& segIdsToRemove)
{
    if (segIdsToRemove.empty()) {
        return;
    }
    // 只从内存视图中移除段，不提交版本
    partData->RemoveSegments(segIdsToRemove);
}
```

*   **`DoFinishRemoveObsoleteRtDocs` 是空实现**：这意味着，当基类通过 `modifier` 标记了要删除的文档后，分支回收器**不执行任何后续操作**。文档的删除仅仅停留在 `modifier` 的内存状态中（通常是更新了 `DeletionMap`）。它不会创建新段，也不会进行数据压实。这是一种典型的“逻辑删除”，开销极小。
*   **`DoFinishTrimObsoleteAndEmptyRtSegments` 的简化实现**：它只调用了 `partData->RemoveSegments`，这会从 `PartitionData` 的内存视图中移除段信息，但它**没有调用 `partData->CommitVersion()`**。这意味着这个变更不会被持久化到新的 `version` 文件中。这符合分支“临时”和“易变”的特性。

这种设计选择清晰地反映了分支分区的定位：作为一个临时的、用于实验的副本，它不需要像主分区那样严格地管理磁盘空间和内存。它的回收机制以尽可能低的资源开销为代价，牺牲了空间回收的彻底性。

---

## 3. 总结与展望

Indexlib 的实时数据回收机制是一个设计精良的系统，它成功地解决了在线索引系统长期运行下的核心痛点——垃圾数据膨胀的问题。

*   **模板方法模式的成功应用**：`RealtimePartitionDataReclaimerBase` 通过定义算法骨架和抽象执行步骤，将“做什么”（回收算法）和“怎么做”（具体实现）清晰地分离，为不同场景下的策略扩展提供了极佳的灵活性。
*   **高效的过期数据定位**：利用 `VIRTUAL_TIMESTAMP_INDEX_NAME` 索引来定位过期文档，避免了全量数据扫描，是保证回收效率的关键设计。
*   **策略的场景化**：为标准分区和分支分区提供了两种截然不同的回收策略（“压实” vs “逻辑删除”），体现了系统设计的精细化和对不同业务场景的深刻理解。

**未来展望**：
1.  **更精细的回收策略**：当前的回收是基于时间戳的。未来可以考虑引入更复杂的回收策略，例如基于文档热度（冷数据优先回收）、基于文档质量（低质量文档优先回收）等，以满足更精细化的运营需求。
2.  **回收任务的调度与资源控制**：数据回收，特别是标准策略下的“压实”操作，会消耗较多资源。未来可以设计一个更智能的调度系统，能够根据系统当前的负载（CPU、内存、IO）动态地调整回收任务的触发时机和执行强度，使其对在线服务的影响降到最低。
3.  **与合并策略的联动**：数据回收和段合并（Merge）在功能上有一定的重叠（都能清理已删除的文档）。未来可以考虑将两者进行更深度的融合，例如，在执行合并时，可以利用回收机制提供的信息，优先合并那些包含大量过期数据的段，从而提升整体的资源利用效率。
