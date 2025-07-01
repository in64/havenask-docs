
# Indexlib 定长哈希表合并时写入模块代码分析

**涉及文件:**
* `indexlib/index/kv/hash_table_fix_merge_writer.h`
* `indexlib/index/kv/hash_table_fix_merge_writer.cpp`

## 1. 模块概述

本模块的核心是 `HashTableFixMergeWriter` 类，它负责在 Indexlib 的索引合并（merge/compaction）过程中，处理定长键值（KV）对的写入。合并是将多个旧的、小的、可能包含冗余数据（如已删除或已更新的键）的索引段（segment）合并成一个新的、更紧凑、查询效率更高的段的过程。

与构建时写入（`HashTableFixWriter`）主要处理增量数据不同，合并时写入的核心任务是：

*   **数据去重与更新**: 从多个源段中读取数据，对于相同的键，只保留时间戳最新且未被删除的版本。
*   **内存高效利用**: 需要预估合并后新段的总大小，并一次性分配足够的内存来构建新的哈希表。
*   **空间优化**: 在持久化到磁盘前，对哈希表进行收缩（shrink），移除所有未使用的空间，以生成最紧凑的索引文件。

## 2. 核心设计与工作流程

`HashTableFixMergeWriter` 的工作流程与 `HashTableFixWriter` 类似，也分为初始化（Init）、数据写入（AddIfNotExist）和优化持久化（OptimizeDump）三个阶段，但其内部逻辑和侧重点有显著不同。

### 2.1. 初始化 (Init)

合并任务开始时，`HashTableFixMergeWriter` 被创建和初始化。`Init` 方法负责为合并过程准备好一个空的、内存中的哈希表。

**核心逻辑:**
1.  **预估最大键数 (EstimateMaxKeyCount)**: 这是合并过程与构建过程最关键的区别之一。合并前，可以通过遍历所有待合并段的元信息（`segmentMetrics`）来精确地预估出新段**最多**可能包含多少个独立的键。这是通过累加所有源段的 `KV_KEY_COUNT` 指标实现的。这个预估值是后续内存分配的依据。
2.  **创建哈希表实例**: 与构建时类似，调用 `HashTableFixCreator::CreateHashTableForWriter` 来创建哈希表。但这里传递的 `isOnline` 参数为 `false`，表示是离线场景。
3.  **内存分配**: `HashTableFixMergeWriter` 直接从内存池（`autil::mem_pool::PoolBase* pool`）中分配一块大内存（`pool->allocate(hashTableSize)`），而不是像 `HashTableFixWriter` 那样使用内存映射文件。这是因为合并过程通常在独立的离线环境中执行，直接使用内存池更直接、可控。
4.  **挂载哈希表**: 将哈希表挂载（`MountForWrite`）到从内存池申请的内存块上。
5.  **准备文件写入器**: 创建一个常规的文件写入器（`mFileWriter`），用于在最后 `OptimizeDump` 阶段将哈希表数据写入磁盘。

**设计动机:**
*   **精确的内存预分配**: 通过 `EstimateMaxKeyCount`，可以非常准确地计算出新哈希表所需的内存，避免了构建时可能发生的内存超用或浪费，也无需动态扩容。
*   **离线环境的直接性**: 在离线合并任务中，生命周期是明确的，任务结束后内存池会被统一释放。直接从池中分配大块内存，比管理内存文件更简单高效。

**核心代码分析：`Init`**
```cpp
// in indexlib/index/kv/hash_table_fix_merge_writer.cpp

void HashTableFixMergeWriter::Init(
    /* ... a lot of parameters ... */,
    const framework::SegmentMetricsVec& segmentMetrics,
    const index_base::SegmentTopologyInfo& targetTopoInfo)
{
    // ...
    // 1. 预估最大键数
    uint64_t maxKeyCount = EstimateMaxKeyCount(segmentMetrics, targetTopoInfo);
    int32_t occupancyPct = mKVConfig->GetIndexPreference().GetHashDictParam().GetOccupancyPct();
    Options options(occupancyPct);
    options.mayStretch = true; // 允许在极端情况下进行扩容

    // 2. 创建哈希表实例
    auto hashTableInfo = HashTableFixCreator::CreateHashTableForWriter(mKVConfig, kvOptions, false);
    mHashTable.reset(static_cast<common::HashTableBase*>(hashTableInfo.hashTable.release()));
    // ...

    // 3. 从内存池分配内存
    size_t hashTableSize = mHashTable->CapacityToTableMemory(maxKeyCount, options);
    void* baseAddr = pool->allocate(hashTableSize);

    // 4. 挂载哈希表
    mHashTable->MountForWrite(baseAddr, hashTableSize, options);
    assert(maxKeyCount <= mHashTable->Capacity());

    // 5. 准备文件写入器
    mKVDir = indexDirectory->MakeDirectory(mKVConfig->GetIndexName());
    mFileWriter = mKVDir->CreateFileWriter(KV_KEY_FILE_NAME);
}
```

### 2.2. 数据写入 (AddIfNotExist)

合并逻辑的核心。外部的合并器（Merger）会遍历所有源段的数据，并调用 `AddIfNotExist` 来尝试将数据写入新的哈希表。

**核心逻辑:**
*   **存在性检查**: 方法名 `AddIfNotExist` 明确了其核心行为。在写入前，首先调用 `mHashTable->Find()` 检查当前键是否已经存在于新的哈希表中。由于合并器通常会按时间戳从新到旧的顺序处理数据，因此第一个遇到的键值对就是最新的，后续遇到的同名键都可以被忽略。
*   **写入新值**: 如果键不存在，则调用 `HashTableFixWriter::Add()` 的静态版本（这是一个代码复用）将键值对写入哈希表。这个静态 `Add` 方法的逻辑与 `HashTableFixWriter` 中的成员函数版本完全相同。
*   **动态扩容 (Stretch)**: 尽管 `Init` 阶段已经预估了内存，但在极少数情况下（例如，哈希函数分布极不均匀导致冲突严重），哈希表仍可能被填满。`AddIfNotExist` 包含了对这种情况的处理：如果 `Add` 失败，它会尝试调用 `mHashTable->Stretch()` 来进行扩容，然后重试写入。这是一个重要的容错机制。

**核心代码分析：`AddIfNotExist`**
```cpp
// in indexlib/index/kv/hash_table_fix_merge_writer.cpp

bool HashTableFixMergeWriter::AddIfNotExist(const keytype_t& key, const autil::StringView& keyRawValue,
                                            const autil::StringView& value, uint32_t timestamp, bool isDeleted,
                                            regionid_t regionId)
{
    autil::StringView uselessStr;
    // 1. 检查键是否存在
    if (mHashTable->Find(key, uselessStr) != util::NOT_FOUND) {
        // 如果已存在，直接返回 true，表示处理成功（忽略了旧数据）
        return true;
    }

    // 2. 如果不存在，尝试添加
    if (likely(HashTableFixWriter::Add(key, value, timestamp, isDeleted, regionId, *mHashTable, *mValueUnpacker))) {
        return true;
    }

    // 3. 添加失败，尝试扩容并重试
    IE_LOG(WARN, "Add key[%lu] failed, trigger stretch and retry", key);
    return mHashTable->Stretch() &&
           HashTableFixWriter::Add(key, value, timestamp, isDeleted, regionId, *mHashTable, *mValueUnpacker);
}
```

### 2.3. 优化持久化 (OptimizeDump)

当所有源段的数据都处理完毕后，`OptimizeDump` 方法被调用，负责将内存中的新哈希表写入磁盘。

**核心逻辑:**
1.  **收缩哈希表 (Shrink)**: 这是合并过程非常重要的一步优化。调用 `mHashTable->Shrink()`，它会移除所有预留的但未使用的桶（bucket）空间，以及因哈希冲突而可能产生的空闲槽位，使得哈希表在内存中的布局达到最紧凑的状态。
2.  **写入磁盘**: 使用在 `Init` 阶段创建的 `mFileWriter`，将收缩后的哈希表内存区域（从 `mHashTable->Address()` 开始，长度为 `mHashTable->MemoryUse()`）一次性写入到 `key` 文件中。
3.  **关闭文件**: 关闭文件写入器，确保数据落盘。
4.  **写入格式化选项**: 与 `HashTableFixWriter` 一样，写入 `kv_format_option` 文件，记录元信息。

**核心代码分析：`OptimizeDump`**
```cpp
// in indexlib/index/kv/hash_table_fix_merge_writer.cpp

void HashTableFixMergeWriter::OptimizeDump()
{
    // 1. 收缩哈希表以优化空间
    bool ret = mHashTable->Shrink();
    assert(ret);
    (void)ret;
    IE_LOG(INFO, "shrink key file[%s], ...", mFileWriter->DebugString().c_str());

    // 2. 将紧凑后的哈希表内存块写入文件
    mFileWriter->Write(mHashTable->Address(), mHashTable->MemoryUse()).GetOrThrow();
    mFileWriter->Close().GetOrThrow();

    // 3. 写入格式化选项文件
    KVFormatOptions kvOptions;
    kvOptions.SetShortOffset(false);
    kvOptions.SetUseCompactBucket(mUseCompactBucket);
    mKVDir->Store(KV_FORMAT_OPTION_FILE_NAME, kvOptions.ToString(), file_system::WriterOption());

    IE_LOG(INFO, "flush key file[%s], shrinkSize[%lub], ...", ...);
}
```

## 3. 关键技术与设计考量

*   **读写分离**: `HashTableFixMergeWriter` 的设计体现了合并过程的“读-处理-写”模式。它自身只负责“写”的阶段，而“读”则由外部的 `KVSegmentIterator` 等组件完成。这种分离使得逻辑清晰。
*   **空间优化优先**: 与追求写入速度的 `HashTableFixWriter` 不同，`HashTableFixMergeWriter` 的一个核心目标是生成空间最优的索引段。`EstimateMaxKeyCount` 的精确预估和 `OptimizeDump` 中的 `Shrink` 操作都是为了这个目标服务。
*   **代码复用**: 通过调用 `HashTableFixWriter::Add` 的静态方法，复用了核心的哈希表插入逻辑，避免了代码冗余。

## 4. 技术风险

*   **内存预估偏差**: `EstimateMaxKeyCount` 依赖于源段 `SegmentMetrics` 的准确性。如果 metrics 信息有误，可能导致预分配的内存过多或不足。虽然有 `Stretch` 机制作为补充，但频繁的 `Stretch` 会严重影响合并性能。
*   **大内存占用**: 合并任务，特别是全量合并（full merge），可能会一次性处理非常多的数据。`Init` 阶段一次性分配的内存可能会非常巨大，对系统的总内存容量提出挑战。

## 5. 总结

`HashTableFixMergeWriter` 是 Indexlib KV 索引合并流程中的关键执行者。它围绕“先预估、再写入、后优化”的核心思想，设计了一套高效、稳健的合并写入机制。通过精确的内存预估、只写最新数据的策略以及最终的收缩优化，它能够将多个零散、冗余的旧索引段，整合成一个查询高效、存储紧凑的新段，是保障 Indexlib KV 索引长期健康运行和高性能查询的重要基石。
