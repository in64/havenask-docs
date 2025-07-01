
# Indexlib KV 存储引擎：变长哈希表实现深度剖析（三）索引合并

**涉及文件:**
*   `indexlib/index/kv/hash_table_var_merge_writer.h`
*   `indexlib/index/kv/hash_table_var_merge_writer.cpp`

## 1. 系统概述与设计哲学

在 Indexlib 中，随着数据的持续写入，会产生大量的索引段 (Segment)。为了控制段的数量、回收已删除文档占用的空间、优化数据布局以提升查询性能，系统会定期执行**合并 (Merging)** 操作。`HashTableVarMergeWriter` 正是 KV 索引在此过程中的核心工作单元，它负责读取多个旧段的数据，处理其中的增、删、改，并最终生成一个全新的、合并后的优化段。

`HashTableVarMergeWriter` 的设计哲学与 `HashTableVarWriter` 既有相似之处，也有本质区别。它同样追求**效率**和**资源控制**，但其工作模式是从“多对一”的数据流中提炼最终结果，这引入了新的挑战。

*   **设计目标**：高效地将多个源 Segment 的 KV 数据合并成一个目标 Segment。这包括：解决来自不同源的同一个 Key 的版本冲突（通常保留最新的）、彻底清除被标记为删除的 Key、重新组织 Value 数据以消除碎片，并构建一个紧凑的、最优化的新哈希表。

*   **设计动机**：
    1.  **数据收敛与去重**：合并的核心是数据收敛。对于同一个主键 (PKey)，在多个 Segment 中可能存在多个版本（例如，先增加，后修改）。`HashTableVarMergeWriter` 必须确保在合并后的新段中，每个 Key 只存在一个最终状态（要么是最新版本的值，要么是被删除）。`AddIfNotExist` 接口的设计正是为此服务。
    2.  **资源预估与一次性分配**：与增量构建不同，合并操作的输入是确定的（待合并的 Segment 列表）。这使得 `HashTableVarMergeWriter` 可以在开始合并前，通过分析源 Segment 的 `SegmentMetrics`（元数据），相对准确地**预估**出合并后总的 Key 数量和大致的内存占用。基于这个预估，它可以一次性分配好所需的全部内存，避免了在合并过程中因内存不足而需要动态扩容（Rehash）的昂贵操作，从而保证了合并过程的稳定性和性能。
    3.  **逻辑复用与特化**：合并写入与普通写入在底层操作（如向哈希表插入、向 Value 文件写入）上有很多相似之处。`HashTableVarMergeWriter` 在实现上巧妙地复用了一些 `HashTableVarWriter` 的静态逻辑和辅助类，但其自身作为一个独立的类，专注于处理合并场景特有的复杂逻辑，如 Key 的存在性判断和 PKey 的存储。

## 2. 核心功能与架构解析

`HashTableVarMergeWriter` 的工作流程同样遵循 **Init -> Add (循环调用) -> OptimizeDump** 的模式。

![HashTableVarMergeWriter Architecture](https://mermaid.ink/svg/eyJjb2RlIjoiZ3JhcGggVERcbiAgICBzdWJncmFwaCBcIkV4dGVybmFsIENhbGxlciAoTWVyZ2VyKVwiXG4gICAgICAgIE0oTWVyZ2VyKSAtLT4gfEluaXQsIEFkZElmTm90RXhpc3QsIE9wdGltaXplRHVtcHwgSE1XKChHYXNoVGFibGVWYXJNZXJnZVdyaXRlcikpXG4gICAgZW5kXG5cbiAgICBzdWJncmFwaCBcIkhhc2hUYWJsZVZhck1lcmdlV3JpdGVyIEludGVybmFsc1wiXG4gICAgICAgIEhJTklUXkJlZ2luIEluaXRcblxuICAgICAgICBISU5JVCAtLT4gfEVzdGltYXRlIFJlc291cmNlc3wgRVNUXG4gICAgICAgIEVTVCAtLT4gfFNlZ21lbnRNZXRyaWNzfCBFU1RcbiAgICAgICAgIEVTVCAtLT4gfCZEb3VibGU6IGF2Z01lbW9yeVJhdGlvLCBVbnQ2NDogbWF4S2V5Q291bnR8IEhJTklUXG5cbiAgICAgICAgIEhJTklUIC0tPiB8Q3JlYXRlIEhhc2hUYWJsZSB2aWEgQ3JlYXRvcnwgQ1JFQVRPUihIYXNoVGFibGVWYXJDcmVhdG9yKVxuICAgICAgICBDUkVBVE9SIC0tPiB8SGFzaFRhYmxlSW5mb3wgSElOSVRcblxuICAgICAgICBISU5JVCAtLT4gfEFsbG9jYXRlIE1lbW9yeSBGb3IgSGFzaFRhYmxlfCBISU5JVFxuICAgICAgICBISU5JVCAtLT4gfENyZWF0ZSBLZXkgJiBWYWx1ZSBGaWxlV3JpdGVyc3wgSElOSVRcblxuICAgICAgICBIQUREXlJlY2VpdmUgS1YgUGFpcl5cbiAgICAgICAgSEFERChgQWRkSWZOb3RFeGlzdGApIC0tPiB8S2V5IEV4aXN0cz98IEZJTkQoZmluZCBpbiBIYXNoVGFibGUpXG4gICAgICAgIEZJTkQgLS0-IHxZZXN8IEhBREQgLS0-IHxEb25lLCByZXR1cm4gdHJ1ZXwgRE9ORVxuICAgICAgICBGSU5ELC0tPiB8Tm98IEFERF9QUk9DRVNTe0FkZCBQcm9jZXNzfVxuICAgICAgICBBRERfUFJPQ0VTUyAtLT4gfEFwcGVuZCBWYWx1ZXwgVkZXXG4gICAgICAgIEFERF9QUk9DRVNTIC0tPiB8UGFjayBPZmZzZXQgJiBUaW1lc3RhbXB8IFZBVFxuICAgICAgICBBRERfUFJPQ0VTUyAtLT4gfEluc2VydC9EZWxldGUgaW50byBIYXNoVGFibGV8IEhUQlxuICAgICAgICBBRERfUFJPQ0VTUyAtLT4gfFN0b3JlIFBLZXkgUmF3IFZhbHVlIChpZiBuZWVkZWQpfCBQS0VZTVBcbblxuICAgICAgICBIREVNUF5PcHRpbWl6ZUR1bXBeXG4gICAgICAgIEhERU1QIC0tPiB8U2hyaW5rIEhhc2hUYWJsZXwgSFRCXG4gICAgICAgIEhERU1QIC0tPiB8Q29tcHJlc3MgJiBEdW1wIEtleSBGaWxlfCBLRkRcbiAgICAgICAgIEhERU1QIC0tPiB8Q2xvc2UgVmFsdWUgRmlsZXwgVkZXXG4gICAgICAgIEhERU1QIC0tPiB8U3RvcmUgTWV0YWRhdGF8IE1EXG4gICAgICAgIEhERU1QIC0tPiB8RHVtcCBQS2V5IEF0dHJpYnV0ZSAoaWYgbmVlZGVkKXwgUEtFWURVTVBcbiAgICBlbmRcblxuICAgIHN0eWxlIE0gZmlsbDojY2ZmYWZmXG4gICAgc3R5bGUgSE1XIGZpbGw6I2ZmZWFjOVxuICAgIHN0eWxlIEhJTklUIGZpbGw6I2QzZmFkMyxzdHJva2U6IzMzMyxzdHJva2Utd2lkdGg6MnB4XG4gICAgc3R5bGUgSEFERCBmaWxsOiNmYWRjZDMsIHN0cm9rZTojMzMzLCBzdHJva2Utd2lkdGg6MnB4XG4gICAgc3R5bGUgSERFTVAgZmlsbDojZjlkOGQzLHN0cm9rZTojMzMzLHN0cm9rZWEtd2lkdGg6MnB4XG4gICAgc3R5bGUgQ1JFQVRPUiBmaWxsOiNmZmNjY2NcbiAgICBzdHlsZSBGRVNUIGZpbGw6I2U2ZTNmY1xuICAgIHN0eWxlIEhUQiwgVkZXLFBBS0VZTVAsIFBLRVlEVU1QLCBFREYsIE1EIGZpbGw6I2VjZjBlN1xuXG4iLCJtZXJtYWlkIjp7InRoZW1lIjoiZGVmYXVsdCJ9LCJ1cGRhdGVFZGl0b3IiOmZhbHNlfQ)

### 2.1. 初始化 (Init)

`Init` 方法的职责是为合并过程准备好所有的基础设施。

1.  **资源预估 (Estimation)**: 这是与 `HashTableVarWriter` 最显著的不同点。
    *   `EstimateMaxKeyCount`: 遍历所有待合并段的 `SegmentMetrics`，将它们的 `KV_KEY_COUNT` 指标进行累加，得到一个合并后**最大可能**的 Key 总数 `maxKeyCount`。这是一个上界，因为不同段中相同的 Key 在合并后只会保留一个。
    *   `EstimateMemoryRatio`: 同样遍历 `SegmentMetrics`，计算所有段的 `KV_KEY_VALUE_MEM_RATIO` 的**平均值**，作为新段的内存分配比例 `mAvgMemoryRatio`。
2.  **创建哈希表**: 与 `Writer` 类似，调用 `HashTableVarCreator::CreateHashTableForWriter` 获取 `HashTableInfo`。
3.  **内存分配**: 基于第一步的预估结果，进行一次性内存分配。
    *   调用 `mHashTable->CapacityToTableMemory(maxKeyCount, options)`，根据预估的 `maxKeyCount` 和配置的 `occupancyPct`，计算出足以容纳所有 Key 的哈希表大小 `hashTableSize`。
    *   直接向内存池 (`autil::mem_pool::PoolBase`) 申请 `hashTableSize` 大小的内存，并调用 `mHashTable->MountForWrite` 将其挂载。
    *   **关键点**：这里不再有复杂的内存比例计算，而是直接为哈希表分配其预估需要的所有内存。Value 的存储则直接写入文件，不占用预分配的内存池。
4.  **创建文件写入器**: 使用 `mIndexDirectory->CreateFileWriter` 创建 `kv_key` 和 `kv_value` 的文件写入器。Value 文件写入器可以配置压缩选项。

### 2.2. 添加数据 (AddIfNotExist)

合并器 (Merger) 在遍历源 Segment 时，会为每个 KV 对调用 `AddIfNotExist` 方法。这个方法是合并逻辑的核心所在。

1.  **存在性检查**: `autil::StringView uselessStr; if (mHashTable->Find(key, uselessStr) != util::NOT_FOUND)`
    *   在尝试添加任何数据之前，首先使用 `mHashTable->Find(key, ...)` 检查当前 Key 是否已经存在于新构建的哈希表中。
    *   如果 `Find` 成功，意味着一个更新版本的同名 Key（来自一个更新的 Segment）已经被处理过了。根据“保留最新”原则，当前这个旧版本的 KV 对将被直接忽略，方法返回 `true`，处理结束。
2.  **执行添加**: 只有当 `Find` 失败时，才会执行真正的添加逻辑，这个逻辑由内部的 `Add` 方法实现，与 `HashTableVarWriter::Add` 非常相似：
    *   将 `value` 写入 `mValueFileWriter`，获取 `offset`。
    *   格式化 `offset` 和 `regionId`。
    *   打包 `offset` 和 `timestamp`。
    *   调用 `mHashTable->Insert(key, ...)` 或 `mHashTable->Delete(key, ...)` 写入哈希表。
3.  **处理 PKey 存储**: 如果配置了 `mNeedStorePKeyValue`（通常用于支持主键索引），会将原始的 PKey 值（`keyRawValue`）与 `key` (PKey 的哈希值) 的映射关系存储在 `mHashKeyValueMap` 中，以便在 `Dump` 阶段构建 PKey 属性索引。

### 2.3. 优化并持久化 (OptimizeDump)

当所有源 Segment 的数据都通过 `AddIfNotExist` 处理完毕后，外部调用 `OptimizeDump` 来完成最终的持久化工作。

1.  **哈希表收缩与压缩 (Shrink & Compress)**: 这是 `OptimizeDump` 的核心优化步骤。
    *   `mHashTable->Shrink()`: 回收因 rehash（尽管在 Merge 模式下不期望发生）或哈希表内部数据结构造成的内存碎片。
    *   `mHashTable->Compress(mBucketCompressor.get())`: **这是一个关键优化**。如果配置了 `EnableShortenOffset` 且总的 Value 文件大小 `mValueFileLength` 小于阈值，系统会尝试对哈希表的桶 (Bucket) 进行压缩。`BucketOffsetCompressor` 会将桶内存储的指向下一个冲突节点的指针（通常是 64 位）压缩成更短的偏移量，从而显著减小最终 Key 文件的大小。如果压缩后的尺寸更小，则将压缩后的数据写入 `mKeyFileWriter`。
2.  **写入与关闭文件**: 将 Key 文件和 Value 文件关闭，数据刷盘。
3.  **存储元数据**: 存储 `KVFormatOptions` 和 `MultiRegionInfo`。
4.  **转储 PKey 属性 (Dump PKey Attribute)**: 如果 `mNeedStorePKeyValue` 为 true，此时会进行一个额外的步骤：
    *   创建一个 `HashTableFileIterator` 来遍历刚刚生成的 Key 文件。
    *   **按 Value (Offset) 排序**: `mHashTableFileIterator->SortByValue()`。这是一个非常重要的操作，它将哈希表中的条目按照它们在 Value 文件中的物理偏移量进行排序。
    *   **顺序添加属性**: 遍历排序后的哈希表，此时可以顺序地从 `mHashKeyValueMap` 中取出 PKey 的原始值，并调用 `mAttributeWriter->AddField()` 将其添加到 PKey 属性索引中。因为是按 Offset 顺序添加，这保证了 PKey 属性的 DocID 与 KV 数据的物理顺序一致，有利于后续的访问局部性。
    *   最后调用 `mAttributeWriter->Dump()` 将 PKey 属性索引持久化。

## 3. 关键实现细节

### 3.1. 预估与一次性分配

预估机制是 `HashTableVarMergeWriter` 性能的基石。通过避免合并过程中的动态内存分配，它将一个不确定性的过程（不知道最终有多少独立 Key）转化为一个确定性的、可预测的过程。

**核心代码片段 (`hash_table_var_merge_writer.cpp`)**
```cpp
uint64_t HashTableVarMergeWriter::EstimateMaxKeyCount(
    const framework::SegmentMetricsVec& segmentMetrics, 
    const index_base::SegmentTopologyInfo& targetTopoInfo)
{
    uint64_t maxKeyCount = 0;
    const std::string& groupName = index_base::SegmentMetricsUtil::GetColumnGroupName(targetTopoInfo.columnIdx);
    for (size_t i = 0; i < segmentMetrics.size(); ++i) {
        maxKeyCount += segmentMetrics[i]->Get<size_t>(groupName, KV_KEY_COUNT);
    }
    return maxKeyCount;
}

void HashTableVarMergeWriter::Init(...)
{
    // ...
    uint64_t maxKeyCount = EstimateMaxKeyCount(segmentMetrics, targetTopoInfo);
    // ...
    auto hashTableInfo = HashTableVarCreator::CreateHashTableForWriter(mKVConfig, kvOptions, false);
    mHashTable.reset(static_cast<common::HashTableBase*>(hashTableInfo.hashTable.release()));
    // ...
    size_t hashTableSize = mHashTable->CapacityToTableMemory(maxKeyCount, options);
    void* baseAddr = pool->allocate(hashTableSize);
    mHashTable->MountForWrite(baseAddr, hashTableSize, options);
    assert(maxKeyCount <= mHashTable->Capacity());
    // ...
}
```

### 3.2. `AddIfNotExist` 的幂等性

`AddIfNotExist` 的核心在于其幂等性。对于同一个 Key，无论调用多少次，只有第一次（通常是来自最新 Segment 的那次）会成功写入，后续的调用都会被 `Find` 操作拦截。这精确地实现了合并的数据收敛逻辑。

### 3.3. 按 Value 排序转储 PKey 属性

这是一个非常精巧的设计。通常情况下，哈希表中的 Key 是无序的。如果直接按哈希表的迭代顺序去写 PKey 属性，会导致属性文件中的 DocID 顺序与 Value 文件中的物理顺序不一致，造成随机 I/O。通过 `SortByValue()`，`HashTableVarMergeWriter` 将 PKey 属性的写入顺序调整为与 Value 数据的物理布局一致，将潜在的随机写操作转换为了高效的顺序写操作，同时优化了未来查询时的缓存局部性。

## 4. 技术风险与考量

1.  **预估不准的风险**: 虽然预估机制很有效，但它依赖于 `SegmentMetrics` 的准确性。如果 `SegmentMetrics` 由于某些原因（如异常关闭、版本兼容问题）不准确，可能导致预估的 `maxKeyCount` 偏小。这会使得 `pool->allocate(hashTableSize)` 分配的内存不足，虽然 `HashTable` 内部有 `Stretch` 机制可以应对，但这违背了 Merge 模式下避免 Rehash 的初衷，可能导致性能下降甚至内存分配失败。
2.  **内存池压力**: `HashTableVarMergeWriter` 直接从一个 `autil::mem_pool::PoolBase` 中分配一大块内存。这意味着在合并期间，这块内存会被持续占用。对于大规模的合并任务，这可能会对系统的总内存造成较大压力。需要有合理的并发合并控制和内存管理策略来避免系统内存耗尽。
3.  **PKey 存储开销**: 当 `mNeedStorePKeyValue` 开启时，`mHashKeyValueMap` 会在内存中缓存所有 Key 的哈希值到其原始值的映射。对于 Key 数量巨大且原始 Key 较长的场景，这个 map 本身会占用不可忽视的内存。虽然这部分内存在 `Dump` 之后会释放，但在合并期间它是一个实实在在的开销。

## 5. 总结

`HashTableVarMergeWriter` 是 Indexlib KV 索引生命周期管理中至关重要的一环。它通过**精确的资源预估**、**一次性内存分配**和**幂等的 `AddIfNotExist` 逻辑**，高效、稳定地完成了多对一的索引合并任务。其在 `OptimizeDump` 阶段引入的**哈希表压缩**和**按 Value 排序转储 PKey 属性**等优化措施，进一步体现了其对性能和资源利用率的极致追求。`HashTableVarMergeWriter` 的设计，是兼顾高性能、资源可控和功能完备性的典范，确保了 Indexlib 中的 KV 索引在长期运行后依然能保持紧凑的结构和高效的查询性能。
