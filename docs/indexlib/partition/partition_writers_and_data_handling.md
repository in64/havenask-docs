
# Indexlib 分区写入器与数据处理代码分析

**涉及文件:**
* `indexlib/partition/partition_writer.cpp`
* `indexlib/partition/partition_writer.h`
* `indexlib/partition/offline_partition_writer.cpp`
* `indexlib/partition/offline_partition_writer.h`
* `indexlib/partition/online_partition_writer.cpp`
* `indexlib/partition/online_partition_writer.h`
* `indexlib/partition/join_segment_writer.cpp`
* `indexlib/partition/join_segment_writer.h`

## 1. 系统概述

分区写入器（Partition Writer）是 Indexlib 数据处理流水线的核心引擎，它承接了从 `IndexBuilder` 传入的文档流，并负责将其转化为结构化的、可被查询的索引格式。这个过程涉及到文档的解析、字段的处理、倒排和正排索引的建立、以及操作日志的记录等一系列复杂操作。可以说，写入器的性能和稳定性直接决定了整个索引系统的构建效率和数据质量。

该模块主要由以下几个关键组件构成：

*   **`PartitionWriter` (抽象基类)**: 定义了所有写入器统一的对外接口，如 `BuildDocument`、`DumpSegment` 等，并引入了 `BuildMode` 的概念，为不同的构建场景（流式、批式）提供了灵活的策略支持。它是整个写入器体系的契约。

*   **`OfflinePartitionWriter`**: 这是 `PartitionWriter` 的主力实现，专为离线和在线环境下的核心构建任务而设计。它内部封装了对 `SegmentWriter`、`PartitionModifier` 和 `OperationWriter` 的精细管理，实现了从文档接收到内存段（In-memory Segment）构建的全过程。无论是离线的大规模批处理，还是在线的实时写入，底层的核心逻辑都由它驱动。

*   **`OnlinePartitionWriter`**: 作为在线服务环境的特定写入器，它更像是一个适配器或代理。它内部持有一个 `OfflinePartitionWriter` 实例，将大部分构建工作委托给它。其自身的职责更多地聚焦于在线场景的特殊逻辑，例如根据服务的时间戳（`rtFilterTimestamp`）过滤掉过时的文档，确保进入实时构建系统的数据是最新的。

*   **`JoinSegmentWriter`**: 这是一个功能高度特化的写入器，仅在在线分区 `Reopen`（版本热切换）的特定阶段被激活。它的任务不是构建新文档，而是“修补”新加载的磁盘段（on-disk segments）。通过回放（replay）在版本切换期间积累的操作日志（如增、删、改），它将这些变更应用到新段上，生成一个临时的“Join Segment”，从而保证了数据在新旧版本切换时的连续性和一致性。

这些写入器共同协作，形成了一个功能完备、适应性强的数据处理体系。`OfflinePartitionWriter` 提供了强大的基础构建能力，`OnlinePartitionWriter` 在其上增加了在线服务的适应性，而 `JoinSegmentWriter` 则解决了在线版本无缝升级中的数据一致性难题。

## 2. `PartitionWriter`：写入器接口与构建模式

`PartitionWriter` 作为一个抽象基类，其核心价值在于**定义规范**和**提供策略**。

### 2.1 功能目标

*   **统一接口**: 为所有具体的写入器实现提供了一套标准的接口，包括 `Open`, `ReOpenNewSegment`, `BuildDocument`, `DumpSegment`, `Close` 等。这使得上层模块（如 `IndexBuilder`）可以面向接口编程，无需关心底层的具体实现是 `OfflinePartitionWriter` 还是其他类型。
*   **定义构建模式 (`BuildMode`)**: 这是 `PartitionWriter` 最重要的设计之一。它抽象了不同的文档构建策略，以满足不同场景下的性能和一致性需求。

### 2.2 核心逻辑：构建模式 (`BuildMode`)

`BuildMode` 是一个枚举类型，定义了三种核心的构建策略：

```cpp
// indexlib/partition/partition_writer.h

enum BuildMode {
    // 流式。每调用一次 Build(Doc*) 就构建一次，单线程，返回后即可查询。
    BUILD_MODE_STREAM,

    // 批式：攒批构建，多线程。内部达到攒批阈值或触发Dump条件后再实际构建...

    // 一致性批式：对于每一个Doc，要么处于没有Build状态，要么处于已Build完成状态...
    BUILD_MODE_CONSISTENT_BATCH,

    // 不一致性批式：对于一个Doc，可能部分字段已Build完成，而部分字段尚未Build...
    BUILD_MODE_INCONSISTENT_BATCH
};
```

*   **`BUILD_MODE_STREAM` (流式模式)**:
    *   **逻辑**: 每当 `BuildDocument` 被调用，文档会立即被完整地处理和构建，构建完成后即可被查询。
    *   **优点**: 逻辑简单，数据可见性高，延迟低。
    *   **缺点**: 单线程处理，对于大量小文档的场景，频繁的调用和处理开销较大，吞吐量受限。
    *   **适用场景**: 文档量不大，或对实时可见性要求极高的场景。

*   **`BUILD_MODE_CONSISTENT_BATCH` (一致性批处理模式)**:
    *   **逻辑**: `BuildDocument` 调用时，文档仅被收集到一个批次中，并不会立即构建。当批次大小、内存占用或时间间隔达到预设阈值时，整个批次的文档会被提交到一个**多线程**的构建流水线中。系统会**等待这个批次的所有文档全部构建完成**后，才返回。返回后，该批次的所有文档都变得完全可查。
    *   **优点**: 多线程并行构建，吞吐量高。保证了批次内数据的一致性，即一个批次要么都不可查，要么都完全可查，不会出现中间状态。
    *   **缺点**: 批次触发构建时，`BuildDocument` 的调用会有明显的延迟（因为需要等待整个批次完成）。
    *   **适用场景**: 在线服务等需要高吞吐量同时又要求数据查询状态一致的场景。

*   **`BUILD_MODE_INCONSISTENT_BATCH` (非一致性批处理模式)**:
    *   **逻辑**: 与一致性批处理类似，也是攒批后多线程构建。但核心区别在于，`BuildDocument` 调用后**不会等待批次完成**，而是立即返回。构建任务在后台异步执行，并且不同字段的构建可能在不同的线程中，完成时间点也不同。
    *   **优点**: **吞吐量最高**。因为它通过跨批次的流水线作业，极大地减少了线程等待和同步开销，延迟最低。
    *   **缺点**: 数据可见性是不确定的。在批次构建完成前，一个文档可能只有部分字段（如主键索引）可查，而其他字段的查询结果是未定义的。这对于需要提供准确查询服务的在线系统是不可接受的。
    *   **适用场景**: 离线构建或上层应用不提供实时查询服务的场景，此时可以最大限度地追求构建速度。

### 2.3 设计动机

`PartitionWriter` 的设计体现了典型的**策略模式**。它将构建文档这一行为的具体实现策略（`BuildMode`）从行为本身（`BuildDocument` 接口）中分离出来。这使得 Indexlib 可以根据不同的应用场景（由 `IndexPartitionOptions` 配置决定）灵活地切换底层实现，以达到最优的性能和一致性表现，而上层代码完全不受影响。这种灵活性是 Indexlib 能够同时高效支持离线和在线环境的关键。

## 3. `OfflinePartitionWriter`：核心构建引擎

`OfflinePartitionWriter` 是 `PartitionWriter` 的核心实现，它包含了将一个文档（`Document`）处理并写入内存段（`InMemorySegment`）的完整逻辑。它不仅被用于离线构建，也是在线实时构建的底层执行者。

### 3.1 功能目标

`OfflinePartitionWriter` 的目标是**高效、准确地完成单个内存段的构建**。其主要职责包括：

*   **生命周期管理**: 管理 `SegmentWriter`、`PartitionModifier` 和 `OperationWriter` 这三个核心组件的生命周期。
*   **文档预处理**: 在文档被正式构建前，对其进行一系列的预处理，如时间戳（TTL）处理、文档重写（`DocumentRewriterChain`）等。
*   **文档分发**: 根据文档的操作类型（`DocOperateType`），将其分发给不同的组件处理。`ADD_DOC` 由 `SegmentWriter` 处理，而 `UPDATE_FIELD` 和 `DELETE_DOC` 由 `PartitionModifier` 处理。
*   **操作记录**: 所有操作（增、删、改）都会被 `OperationWriter` 序列化并记录下来，形成操作日志，用于后续的数据恢复和版本切换。
*   **内存与资源控制**: 监控并估算当前构建任务的内存使用，为上层（`IndexBuilder`）的 Dump 决策提供依据。
*   **Dump 与关闭**: 执行 `DumpSegment` 将内存中的数据固化到磁盘，并通过 `Close` 确保所有资源被正确释放和提交。

### 3.2 核心逻辑与算法

#### 3.2.1 组件协同

`OfflinePartitionWriter` 的核心是协调三大组件的工作：

*   **`SegmentWriter`**: 负责处理 `ADD_DOC` 类型的文档。它内部会为每个字段（Field）创建一个对应的 `IndexSegmentWriter`（如倒排、正排、摘要等），并将文档内容写入这些 `IndexSegmentWriter` 的内存缓冲区中。
*   **`PartitionModifier`**: 负责处理 `UPDATE_FIELD` 和 `DELETE_DOC` 类型的文档。它不会直接修改已有的索引，而是将这些变更操作记录在内存中（例如，在一个 `Bitmap` 中标记删除的文档 ID，或在一个 `Patch` 文件中记录更新的属性值）。这些变更将在查询时或合并时应用。
*   **`OperationWriter`**: 负责将所有到来的操作（ADD, DELETE, UPDATE）序列化后写入一个连续的内存块。这个内存块最终会作为段的一部分（`operation_log` 文件）被 Dump 到磁盘，是实现在线 `Reopen` 和数据恢复的基础。

#### 3.2.2 文档构建流程 (`BuildDocument`)

当 `BuildDocument` 被调用时，会依次执行以下步骤：

```cpp
// indexlib/partition/offline_partition_writer.cpp

bool OfflinePartitionWriter::BuildDocument(const DocumentPtr& doc)
{
    // ...
    // 1. 预处理
    if (!PreprocessDocument(doc)) {
        return false;
    }

    bool ret = false;
    DocOperateType opType = doc->GetDocOperateType();
    switch (opType) {
    case ADD_DOC: {
        // 2. 分发给 SegmentWriter 和 OperationWriter
        ret = AddDocument(doc);
        break;
    }
    case DELETE_DOC:
    case DELETE_SUB_DOC: {
        // 3. 分发给 PartitionModifier 和 OperationWriter
        ret = RemoveDocument(doc);
        break;
    }
    case UPDATE_FIELD: {
        ret = UpdateDocument(doc);
        break;
    }
    // ...
    }
    // ...
    return ret;
}

bool OfflinePartitionWriter::AddDocument(const DocumentPtr& doc)
{
    // 调用 SegmentWriter 处理 ADD，并调用 OperationWriter 记录操作
    bool ret = mSegmentWriter->AddDocument(doc) && AddOperation(doc);
    // ...
    return ret;
}

bool OfflinePartitionWriter::AddOperation(const DocumentPtr& doc)
{
    // ...
    if (!mOperationWriter) {
        return true;
    }
    // 记录操作到操作日志
    return mOperationWriter->AddOperation(doc);
}
```

#### 3.2.3 批量构建 (`BatchBuild`)

在批处理模式下，`OfflinePartitionWriter` 的 `BatchBuild` 方法会利用一个线程池（`GroupedThreadPool`）来并行化构建过程。

```cpp
// indexlib/partition/offline_partition_writer.cpp

bool OfflinePartitionWriter::BatchBuild(std::unique_ptr<document::DocumentCollector> docCollector)
{
    // ...
    // 1. 预处理所有文档
    PreprocessDocuments(docCollector.get());
    // ...

    // 2. 启动一个新的批处理任务
    _buildThreadPool->StartNewBatch();
    
    // 3. 生成并提交构建任务到线程池
    //    - ADD 操作提交给 OperationWriter
    //    - 字段构建任务提交给对应的 IndexSegmentWriter
    //    - 删除/更新任务提交给 PartitionModifier
    _buildThreadPool->PushTask("__ADD_OP__", ...);
    DocBuildWorkItemGenerator::Generate(_buildMode, mSchema, mSegmentWriter, mModifier, sharedDocCollector,
                                        _buildThreadPool.get());

    // 4. 根据构建模式决定是否等待
    if (_buildMode == BUILD_MODE_CONSISTENT_BATCH) {
        _buildThreadPool->WaitCurrentBatchWorkItemsFinish();
    }
    // ...
    return true;
}
```
`DocBuildWorkItemGenerator` 是这里的关键，它会解析 `DocumentCollector` 中的所有文档，根据 Schema 定义，为每个需要构建的字段生成一个独立的构建任务（`WorkItem`），并将其提交到线程池中。这种高度并行的设计是批处理模式高性能的核心。

### 3.3 技术栈与设计动机

*   **组件化与委托**: `OfflinePartitionWriter` 自身不执行具体的索引写入，而是将任务委托给 `SegmentWriter` 和 `PartitionModifier`。这种设计遵循了“单一职责原则”，使得代码结构清晰，易于扩展（例如，可以方便地为新的索引类型添加新的 `IndexSegmentWriter`）。
*   **多线程并行**: 在批处理模式下，通过 `GroupedThreadPool` 和 `DocBuildWorkItemGenerator` 将文档构建任务分解到字段级别，实现了最大程度的并行化，极大地提升了构建吞吐量。
*   **内存估算与控制**: `PartitionWriterResourceCalculator` 负责估算构建过程中的内存使用，包括当前内存占用、Dump 过程中的峰值内存等。这些精确的估算为上层的内存控制和 Dump 决策提供了关键依据。

设计 `OfflinePartitionWriter` 的动机是**创建一个可复用的、高性能的、与具体业务场景无关的核心构建单元**。通过将它从 `Online` 和 `Offline` 的具体分区逻辑中抽离出来，Indexlib 实现了代码的最大化复用。`OnlinePartitionWriter` 几乎是 `OfflinePartitionWriter` 的一个简单代理，这充分证明了其设计的通用性和强大功能。

### 3.4 可能的技术风险

*   **内存碎片**: 在长时间的构建过程中，频繁地创建和销毁各种内存对象（如 `Token`、`Field` 等），可能会导致内存碎片问题，影响性能。
*   **线程同步开销**: 在批处理模式下，虽然任务是并行的，但最终仍需要在某些同步点（如 `WaitCurrentBatchWorkItemsFinish`）进行等待。如果任务分配不均，导致某些线程成为“长尾”，会降低并行化的效率。
*   **数据倾斜**: 如果输入文档的某些字段特别长或者取值特别集中，可能会导致处理这些字段的 `IndexSegmentWriter` 成为性能瓶颈，影响整体构建速度。

## 4. `OnlinePartitionWriter`：在线写入适配器

`OnlinePartitionWriter` 是 `PartitionWriter` 在线服务环境下的实现。与 `OfflinePartitionWriter` 的复杂性不同，`OnlinePartitionWriter` 的设计非常轻量，它更像是一个代理或装饰器。

### 4.1 功能目标

`OnlinePartitionWriter` 的核心目标是**将在线环境的特定需求与通用的构建逻辑（由 `OfflinePartitionWriter` 提供）进行桥接**。

其主要职责是：

*   **持有和管理 `OfflinePartitionWriter`**: 它是 `OnlinePartitionWriter` 内部的核心成员，负责执行所有实际的文档构建工作。
*   **实时数据过滤**: 这是 `OnlinePartitionWriter` 最核心的功能。它会根据 `OnlinePartition` 在 `Open` 或 `Reopen` 时确定的一个时间戳 `mRtFilterTimestamp`，过滤掉所有时间戳小于等于该值的新到文档。这确保了在线系统不会重复处理已经包含在最新增量版本中的数据。
*   **状态同步**: 与 `OnlinePartition` 的状态（如 `mLastDumpTs`）保持同步，用于触发定期的 Dump 操作。

### 4.2 核心逻辑

`OnlinePartitionWriter` 的核心逻辑非常直接，主要体现在 `BuildDocument` 和 `Open` 方法中。

```cpp
// indexlib/partition/online_partition_writer.cpp

void OnlinePartitionWriter::Open(const IndexPartitionSchemaPtr& schema, const PartitionDataPtr& partitionData,
                                 const PartitionModifierPtr& modifier)
{
    // ...
    // 1. 创建一个 OfflinePartitionWriter 实例
    mWriter.reset(new OfflinePartitionWriter(mOptions, mDumpSegmentContainer, ...));
    mWriter->Open(mSchema, partitionData, modifier);
    
    // 2. 计算并设置 rtFilterTimestamp
    Version version = partitionData->GetOnDiskVersion();
    OnlineJoinPolicy joinPolicy(version, mSchema->GetTableType(), partitionData->GetSrcSignature());
    mRtFilterTimestamp = joinPolicy.GetRtFilterTimestamp();
    // ...
}

bool OnlinePartitionWriter::BuildDocument(const DocumentPtr& doc)
{
    // ...
    // 1. 检查时间戳
    if (!CheckTimestamp(doc)) {
        return false; // 时间戳过旧，直接丢弃
    }

    // 2. 将构建任务委托给内部的 OfflinePartitionWriter
    if (!mWriter->BuildDocument(doc)) {
        return false;
    }
    // ...
    return true;
}

bool OnlinePartitionWriter::CheckTimestamp(const DocumentPtr& doc) const
{
    // 如果文档时间戳小于等于过滤时间戳，则认为其是过时数据
    bool ret = doc->CheckOrTrim(mRtFilterTimestamp);
    if (!ret) {
        // ... 上报指标
    }
    return ret;
}
```

### 4.3 设计动机

`OnlinePartitionWriter` 的设计采用了**代理模式（Proxy Pattern）** 或**装饰器模式（Decorator Pattern）**。它没有重新实现复杂的构建逻辑，而是复用了 `OfflinePartitionWriter` 的全部功能。它像一个代理，拦截了所有进入的 `BuildDocument` 请求，在将其转发给 `OfflinePartitionWriter` 之前，增加了一个在线环境特有的 `CheckTimestamp` 逻辑。

这种设计的优点是：
*   **高度复用**: 避免了大量重复代码，降低了维护成本。
*   **职责清晰**: `OfflinePartitionWriter` 专注于通用的构建算法，而 `OnlinePartitionWriter` 专注于在线场景的特定适配逻辑。当需要为在线环境增加新的特性时（例如，更复杂的过滤规则），只需修改 `OnlinePartitionWriter` 即可，不会影响到核心的构建引擎。

## 5. `JoinSegmentWriter`：版本切换的数据缝合器

`JoinSegmentWriter` 是一个高度特化的写入器，它的生命周期非常短暂，只在 `OnlinePartition` 执行 `Reopen`（热加载新版本）的过程中被创建和使用。

### 5.1 功能目标

`JoinSegmentWriter` 的目标是**解决 `Reopen` 过程中的数据一致性问题**。在 `Reopen` 开始到结束的这段时间内，实时系统仍然在接收新的写操作（增、删、改）。这些操作是基于旧的版本进行的。当新版本加载完成后，这些在新旧版本切换期间发生的操作必须被正确地应用到新版本上，否则就会造成数据丢失。`JoinSegmentWriter` 就是负责完成这个“数据缝合”任务的组件。

### 5.2 核心逻辑

`JoinSegmentWriter` 的工作流程分为 `PreJoin`、`Join` 和 `Dump` 三个阶段。

```cpp
// indexlib/partition/join_segment_writer.cpp

void JoinSegmentWriter::Init(const PartitionDataPtr& partitionData, const Version& newVersion,
                             const Version& lastVersion)
{
    // ...
    // 初始化用于读取实时数据的 PartitionData
    mReadPartitionData = partitionData;
    // 初始化用于写入变更的、基于新版本的 PartitionData
    mNewPartitionData = OnDiskPartitionData::CreateOnDiskPartitionData(mFileSystem, mSchema, mOnDiskVersion, "", true);
    
    // 初始化重做策略，它知道哪些操作需要被应用到新版本上
    mRedoStrategy.reset(new ReopenOperationRedoStrategy());
    mRedoStrategy->Init(mNewPartitionData, mLastVersion, ...);
    
    // 创建一个 PatchModifier，用于将变更写入 mNewPartitionData
    mModifier = PartitionModifierCreator::CreatePatchModifier(...);
    // ...
}

bool JoinSegmentWriter::PreJoin()
{
    // ...
    // 在 Reopen 的准备阶段，回放一部分操作日志
    OperationReplayer opReplayer(preJoinPartData, mSchema, mMemController);
    if (!opReplayer.RedoOperations(mModifier, mOnDiskVersion, mRedoStrategy)) {
        return false;
    }
    mOpCursor = opReplayer.GetCursor(); // 记录回放到的位置
    return true;
}

bool JoinSegmentWriter::Join()
{
    // ...
    // 在 Reopen 的执行阶段，从上次的位置继续回放剩余的操作日志
    OperationReplayer opReplayer(mReadPartitionData, mSchema, mMemController);
    opReplayer.Seek(mOpCursor);
    if (!opReplayer.RedoOperations(mModifier, mOnDiskVersion, mRedoStrategy)) {
        return false;
    }
    // ...
    // 合并删除图
    return JoinDeletionMap();
}

bool JoinSegmentWriter::Dump(const PartitionDataPtr& writePartitionData)
{
    // ...
    // 将所有变更（由 mModifier 记录）Dump 成一个临时的 Join Segment
    InMemorySegmentPtr inMemSegment = writePartitionData->CreateJoinSegment();
    // ...
    mModifier->Dump(inMemSegment->GetDirectory(), inMemSegment->GetSegmentId());
    // ...
    return true;
}
```

其核心逻辑是：
1.  **初始化**: `Init` 方法会准备好两个 `PartitionData`：一个指向当前的实时分区数据（`mReadPartitionData`），用于读取操作日志；另一个基于即将加载的新版本（`mOnDiskVersion`）创建，用于写入变更（`mNewPartitionData`）。
2.  **回放操作日志 (`RedoOperations`)**: `OperationReplayer` 是执行回放的核心。它会从 `mReadPartitionData` 中读取操作日志，并根据 `ReopenOperationRedoStrategy` 的判断，将需要应用到新版本的操作（通常是那些发生在新版本切片时间点之后的操作）通过 `mModifier` 应用到 `mNewPartitionData` 上。
3.  **分阶段执行**: `Reopen` 是一个多阶段过程，为了不阻塞主流程，`JoinSegmentWriter` 也将回放分为 `PreJoin` 和 `Join` 两个阶段，在不同的锁定粒度下执行。
4.  **Dump Join Segment**: 所有回放完成后，`mModifier` 中就包含了切换期间所有的增量变更。`Dump` 方法会将这些变更写入一个特殊的、临时的内存段，即 “Join Segment”。这个 Join Segment 会和新加载的磁盘段一起，构成最终对外服务的、数据完全一致的新版本。

### 5.3 设计动机

`JoinSegmentWriter` 的设计是**为了在保证服务连续性的前提下，实现数据在版本切换时的最终一致性**。如果没有这个机制，`Reopen` 期间的所有实时写入都将丢失。通过精确地回放操作日志，`JoinSegmentWriter` 像一个数据裁缝，将新旧版本之间的数据断层严丝合缝地连接起来，是 Indexlib 在线服务高可用性的关键保障之一。
