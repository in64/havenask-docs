
# Indexlib KV 存储核心：揭秘数据写入与合并

**涉及文件:**

*   `indexlib/index/kv/kv_writer.h`
*   `indexlib/index/kv/kv_merge_writer.h`
*   `indexlib/index/kv/kv_merger.h`
*   `indexlib/index/kv/kv_merger.cpp`
*   `indexlib/index/kv/kv_resource_assigner.h`
*   `indexlib/index/kv/kv_resource_assigner.cpp`

## 1. 引言：KV 数据的“诞生”与“重塑”

在深入了解了 Indexlib KV 模块的底层抽象与配置之后，我们自然会产生一个问题：数据是如何被写入 KV 索引，又是如何在时间的推移中不断演进和优化的？

这个问题的答案，就隐藏在 Indexlib 的数据写入与合并机制之中。这套机制，是 KV 数据从“诞生”到“重塑”的全过程，它直接决定了 KV 索引的构建效率、存储成本和最终的查询性能。

本文将聚焦于 Indexlib KV 模块的数据写入与合并两大核心环节，深入剖析 `KVWriter`、`KVMerger` 以及 `KvResourceAssigner` 等关键组件的设计理念和实现细节。

通过本次探索，您将揭开以下谜题：

*   **实时写入 (`KVWriter`)**: Indexlib 如何在内存中高效地构建哈希表，以支持实时的数据增删改？内存的分配与管理，又蕴含着哪些精巧的设计？
*   **离线合并 (`KVMerger`)**: 为何需要合并？合并过程中，Indexlib 如何处理冗余数据、过期数据，并最终生成一个更加紧凑、高效的新 Segment？
*   **资源分配 (`KvResourceAssigner`)**: 在多路（Column）合并的场景下，Indexlib 如何智能地为每个合并任务分配内存资源，以实现整体构建效率的最大化？

理解数据写入与合并的机制，对于优化 Indexlib 的构建性能、控制存储成本、保障线上服务稳定至关重要。无论您是负责 Indexlib 集群运维的工程师，还是希望提升数据处理效率的算法专家，本文都将为您提供一个深入、全面的视角。让我们一同启程，探索 Indexlib KV 数据“诞生”与“重塑”的奥秘。

## 2. 实时写入的“建筑师”：`KVWriter`

`KVWriter` 是 Indexlib KV 模块中负责实时数据写入的核心组件。它的主要职责，是在内存中构建一个临时的 KV 索引（通常称为内存 Segment 或 In-Memory Segment），以响应实时的增、删、改请求。当内存中的数据积累到一定规模，或者接收到 Dump（转储）指令时，`KVWriter` 会将内存中的索引数据持久化到磁盘，形成一个新的 Segment。

`KVWriter` 的设计，直接关系到实时写入的吞吐量和延迟。一个高效的 `KVWriter`，必须能够在有限的内存资源下，快速地处理大量的写入请求。

### 2.1. `kv_writer.h`：定义写入契约

`kv_writer.h` 文件定义了 `KVWriter` 的纯虚基类，它通过一系列接口，规范了所有具体 `KVWriter` 实现必须遵守的“写入契约”。

```cpp
// indexlib/index/kv/kv_writer.h

class KVWriter
{
public:
    // ...
    virtual void Init(const config::KVIndexConfigPtr& kvIndexConfig, const file_system::DirectoryPtr& segmentDir,
                      bool isOnline, uint64_t maxMemoryUse,
                      const std::shared_ptr<framework::SegmentGroupMetrics>& groupMetrics) = 0;
    virtual bool Add(const keytype_t& key, const autil::StringView& value, uint32_t timestamp, bool isDeleted,
                     regionid_t regionId) = 0;
    virtual void Dump(const file_system::DirectoryPtr& directory, autil::mem_pool::PoolBase* pool) = 0;

    virtual void FillSegmentMetrics(const std::shared_ptr<framework::SegmentMetrics>& metrics,
                                    const std::string& groupName) = 0;

    virtual size_t GetMemoryUse() const = 0;
    virtual bool IsFull() const = 0;
    // ...
};
```

*   **`Init`**: 初始化 `KVWriter`，传入 `KVIndexConfig`、Segment 目录、内存限额等关键参数。
*   **`Add`**: 核心的写入接口。接收一个 Key-Value 对，以及时间戳、删除标记、Region ID 等信息，并将其添加到内存索引中。
*   **`Dump`**: 将内存中的索引数据转储到指定的磁盘目录。
*   **`FillSegmentMetrics`**: 在 Dump 完成后，填充该 Segment 的性能指标（如 Key 数量、内存占用等）。
*   **`GetMemoryUse`**: 获取当前 `KVWriter` 占用的内存大小。
*   **`IsFull`**: 判断当前 `KVWriter` 的内存占用是否已达到上限。

`KVWriter` 的具体实现，分为 `HashTableFixWriter` 和 `HashTableVarWriter` 两种，分别用于处理定长和变长的 KV 数据。这种通过接口进行统一、通过具体类进行实现的模式，是典型的策略模式应用，它使得上层模块可以在不关心具体实现的情况下，与 `KVWriter` 进行交互。

### 2.2. 写入流程的核心：`Add` 方法

`Add` 方法是 `KVWriter` 中最核心、最繁忙的接口。它的内部实现，通常涉及到以下几个关键步骤：

1.  **查找 Key**: 在内存中的哈希表中查找输入的 Key。
2.  **处理冲突**: 
    *   如果 Key 已存在，则根据时间戳判断是否需要更新。通常，只有当新数据的`timestamp` 大于或等于旧数据时，才会进行更新。
    *   如果 Key 不存在，则在哈希表中插入新的 Key-Value 对。
3.  **内存分配**: 为新的 Key 和 Value 分配内存。对于变长数据，还需要处理 Value 的存储和偏移量记录。
4.  **更新状态**: 更新 `KVWriter` 的内部状态，如内存占用、Key 数量等。

### 2.3. 持久化的关键：`Dump` 方法

当 `KVWriter` 的 `IsFull` 方法返回 `true`，或者外部系统触发 Dump 操作时，`Dump` 方法就会被调用。它的核心任务，是将内存中的哈希表和 Value 数据，按照预定义的格式，写入到磁盘文件中。

`Dump` 的过程通常包括：

1.  **创建文件**: 在指定的 Segment 目录下，创建 `kv_key_file` 和 `kv_value_file`。
2.  **写入 Key 文件**: 遍历内存中的哈希表，将其内容（包括 Key、Offset、Timestamp 等）序列化后写入 `kv_key_file`。
3.  **写入 Value 文件**: 将所有 Value 数据写入 `kv_value_file`。
4.  **写入格式化选项**: 将 `KVFormatOptions` 写入 `kv_format_option_file_name` 文件。
5.  **填充 Metrics**: 调用 `FillSegmentMetrics` 方法，将该 Segment 的统计信息记录下来。

`KVWriter` 的设计，通过将实时写入和离线 Dump 分离，实现了高性能的实时数据摄入。内存哈希表的使用，保证了 `Add` 操作的高效性；而批量 Dump 的方式，则避免了频繁的磁盘 I/O，提高了整体的构建吞吐。

## 3. 离线合并的“艺术家”：`KVMerger`

随着时间的推移，系统中会产生大量的 Segment。这些 Segment 中，可能包含大量被删除或被更新的冗余数据，不仅浪费存储空间，还会影响查询性能。为了解决这个问题，Indexlib 引入了合并（Merge）机制。

`KVMerger` 就是负责执行 KV 索引合并的核心组件。它像一位“艺术家”，将多个零散、冗余的旧 Segment，“雕琢”成一个全新的、紧凑、高效的新 Segment。

### 3.1. `kv_merger.h` & `kv_merger.cpp`：定义合并的“艺术手法”

`KVMerger` 的核心逻辑，都封装在 `kv_merger.h` 和 `kv_merger.cpp` 文件中。它的 `Merge` 方法，是整个合并流程的入口。

```cpp
// indexlib/index/kv/kv_merger.h

class KVMerger
{
public:
    // ...
    void Merge(const file_system::DirectoryPtr& segmentDir, const file_system::DirectoryPtr& indexDir,
               const index_base::SegmentDataVector& segmentDataVec,
               const index_base::SegmentTopologyInfo& targetTopoInfo);
    // ...
private:
    void MergeOneSegment(const index_base::SegmentData& segmentData, const KVMergeWriterPtr& writer,
                         bool isBottomLevel);
    void Dump(const KVMergeWriterPtr& writer, const file_system::DirectoryPtr& indexDir, const std::string& groupName);
    // ...
};
```

`Merge` 方法的执行流程，可以概括为以下几个步骤：

1.  **初始化**: 
    *   初始化 `KVTTLDecider`，用于判断数据是否过期。
    *   创建 `KVMergeWriter`，作为新 Segment 的写入器。
    *   加载所有待合并 Segment 的 `SegmentMetrics`。
2.  **迭代合并**: 
    *   按照从新到旧的顺序，依次遍历 `segmentDataVec` 中的每个 Segment。
    *   调用 `MergeOneSegment` 方法，处理单个 Segment 的数据。
3.  **转储**: 
    *   调用 `Dump` 方法，将 `KVMergeWriter` 中合并好的数据持久化到新 Segment 的目录中。

### 3.2. `MergeOneSegment`：合并的核心逻辑

`MergeOneSegment` 是 `KVMerger` 中最核心的方法，它实现了对单个 Segment 数据的读取、过滤和写入。

```cpp
// indexlib/index/kv/kv_merger.cpp

void KVMerger::MergeOneSegment(const SegmentData& segmentData, const KVMergeWriterPtr& writer, bool isBottomLevel)
{
    // ...
    KVSegmentIterator segmentIterator;
    segmentIterator.Open(...);

    while (segmentIterator.IsValid()) {
        // ...
        segmentIterator.Get(pkeyHash, value, timestamp, isDeleted, regionId);

        // 1. TTL 过滤
        if (mTTLDecider->IsExpiredItem(regionId, timestamp, mCurrentTsInSecond)) {
            // ...
            continue;
        }

        // 2. 在更底层的 Level 中进行去重
        if (isBottomLevel) {
            if (mRemovedKeySet.count(pkeyHash) > 0) {
                // ...
                continue;
            }
            if (isDeleted) {
                mRemovedKeySet.insert(pkeyHash);
                // ...
                continue;
            }
        }

        // 3. 写入新 Segment
        if (!writer->AddIfNotExist(pkeyHash, pkeyValueData, value, timestamp, isDeleted, regionId)) {
            // ...
        }
        // ...
    }
}
```

**核心逻辑解读**: 

1.  **创建迭代器**: 首先，创建一个 `KVSegmentIterator`，用于遍历当前 Segment 中的所有 Key-Value 对。
2.  **TTL 过滤**: 对每个 Key-Value 对，使用 `KVTTLDecider` 判断其是否已经过期。如果过期，则直接丢弃。
3.  **去重处理**: 
    *   Indexlib 的合并策略通常是分层的（Level-based）。`isBottomLevel` 参数用于标识当前合并是否发生在最底层。
    *   在最底层的合并中，需要进行严格的去重。`KVMerger` 使用一个 `mRemovedKeySet` 来记录所有已经被删除或被更新的 Key。如果当前 Key 在 `mRemovedKeySet` 中，说明它在更旧的 Segment 中已经被处理过，可以直接丢弃。
    *   如果当前 Key 是一个删除操作（`isDeleted` 为 `true`），则将其加入 `mRemovedKeySet`，以便在后续处理更旧的 Segment 时，能够将对应的旧数据过滤掉。
4.  **写入新 Segment**: 对于通过了所有过滤条件的有效数据，调用 `KVMergeWriter` 的 `AddIfNotExist` 方法，将其写入到新的 Segment 中。`AddIfNotExist` 会确保在新 Segment 中，每个 Key 只存在一份最新的数据。

### 3.3. `KVMergeWriter`：合并过程中的“写入器”

`KVMergeWriter` 是 `KVMerger` 的得力助手。它的接口定义在 `kv_merge_writer.h` 中，与 `KVWriter` 类似，但专为合并场景设计。

`KVMergeWriter` 的核心方法是 `AddIfNotExist`。该方法在内部维护一个哈希表，当添加一个新的 Key-Value 对时，它会先检查该 Key 是否已经存在。只有当 Key 不存在时，才会执行真正的添加操作。这种设计，保证了在从新到旧遍历所有 Segment 的过程中，每个 Key 只有第一次遇到的、最新的版本才会被保留下来。

`KVMerger` 通过这种精巧的迭代和过滤机制，将多个旧 Segment 中的有效数据“萃取”出来，并借助 `KVMergeWriter` 构建出一个全新的、无冗余的 Segment，从而实现了存储空间的优化和查询性能的提升。

## 4. 资源调度的“指挥家”：`KvResourceAssigner`

在大型分布式系统中，资源（尤其是内存）的合理分配，是保证系统稳定和高效运行的关键。在 Indexlib 的离线合并场景中，可能会有多个合并任务（对应不同的 Column 或 Shard）同时进行。如何为每个任务分配合适的内存资源，就成了一个重要的问题。

`KvResourceAssigner` 就是为了解决这个问题而生的。它像一位“指挥家”，根据全局的内存配额和每个合并任务的历史数据量，为 `KVWriter`（在合并场景下由 `KVMergeWriter` 使用）分配合理的内存预算。

### 4.1. `kv_resource_assigner.h` & `kv_resource_assigner.cpp`：定义资源分配策略

`KvResourceAssigner` 的核心逻辑非常清晰，主要体现在其 `Assign` 方法中。

```cpp
// indexlib/index/kv/kv_resource_assigner.cpp

size_t KvResourceAssigner::Assign(const std::shared_ptr<indexlib::framework::SegmentMetrics>& metrics)
{
    // ...
    double curMemRatio = 0;
    GetDataRatio(metrics, curMemRatio);

    double totalBalanceMemory = mTotalMemQuota * DEFAULT_BALANCE_STRATEGY_RATIO;
    double totalDynamicMemory = mTotalMemQuota * (1 - DEFAULT_BALANCE_STRATEGY_RATIO);

    double memQuota = ((double)totalBalanceMemory / mColumnCount) + ((double)totalDynamicMemory * curMemRatio);
    // ...
    return (size_t)memQuota;
}
```

**分配策略解读**: 

`KvResourceAssigner` 采用了一种“静态均衡 + 动态倾斜”的混合策略来分配内存：

1.  **获取总配额**: 首先，从 `QuotaControl` 或 `IndexPartitionOptions` 中获取可用的总内存配额 `mTotalMemQuota`。
2.  **划分内存池**: 将总配额划分为两部分：
    *   **均衡内存池 (`totalBalanceMemory`)**: 占总配额的一部分（由 `DEFAULT_BALANCE_STRATEGY_RATIO` 控制，默认为 30%）。这部分内存将被平均分配给所有的合并任务。
    *   **动态内存池 (`totalDynamicMemory`)**: 剩余的内存。这部分内存将根据每个任务的历史数据量，进行按比例的动态分配。
3.  **计算数据比例**: `GetDataRatio` 方法会读取上一次合并产生的 `SegmentMetrics`，计算出当前任务（Column）的历史内存占用，在所有任务总内存占用中的比例 `curMemRatio`。
4.  **计算最终配额**: 当前任务最终获得的内存配额，等于“均分的均衡内存”加上“按比例分配的动态内存”。

这种混合策略，兼顾了公平性和效率：

*   **均衡内存池**保证了即使是数据量很小的任务，也能获得一个基础的内存配额，避免“饿死”。
*   **动态内存池**则使得数据量大的任务，可以获得更多的内存资源，从而加快其处理速度，最终实现整体合并时间的优化。

`KvResourceAssigner` 的引入，使得 Indexlib 在复杂的多任务合并场景下，能够更加智能、高效地利用宝贵的内存资源，是系统稳定性和性能的重要保障。

## 5. 结论

数据写入与合并，是 Indexlib KV 模块中两个紧密相连、相辅相成的核心环节。它们共同构成了 KV 数据从“诞生”到“重塑”的完整生命周期。

*   **`KVWriter`** 作为实时写入的“建筑师”，通过在内存中高效构建哈希表，为系统提供了低延迟的数据摄入能力。
*   **`KVMerger`** 作为离线合并的“艺术家”，通过精巧的过滤和去重机制，将冗余的旧 Segment “雕琢”成紧凑、高效的新 Segment，优化了存储和查询性能。
*   **`KvResourceAssigner`** 作为资源调度的“指挥家”，通过“静态均衡 + 动态倾斜”的智能分配策略，保证了多任务合并场景下的系统稳定性和整体效率。

深入理解这三个核心组件的设计理念和工作原理，对于我们诊断构建瓶颈、优化资源配置、提升系统性能具有至关重要的意义。它们共同展示了 Indexlib 在追求高性能、高可用道路上的精妙设计与工程智慧。
