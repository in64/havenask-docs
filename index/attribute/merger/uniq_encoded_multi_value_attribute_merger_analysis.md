
# Indexlib 唯一编码多值属性合并 (`UniqEncodedMultiValueAttributeMerger`) 代码分析报告

## 1. 概述

`UniqEncodedMultiValueAttributeMerger<T>` 是对 `MultiValueAttributeMerger<T>` 的一个重要优化。在许多实际场景中，多值属性的“值集合”本身存在大量重复。例如，两个不同的用户可能拥有完全相同的标签集合 `{"tagA", "tagB"}`。常规的多值存储会为这两个用户分别存储一份 `{"tagA", "tagB"}` 的数据，造成了存储空间的浪费。

唯一编码（Uniq-Encoded）压缩正是为了解决这个问题。其核心思想是：**为每个唯一的“值集合”只存储一份数据，并让所有拥有该集合的文档共享这个数据。** 这是通过在数据和偏移量文件之间增加一个“间接层”来实现的。

本报告将深入分析 `UniqEncodedMultiValueAttributeMerger.h` 的实现，揭示其如何在一个已经很复杂的变长数据合并流程中，进一步实现去重和共享，重点包括：

*   **去重合并的核心逻辑 (`MergeData`)**: 如何在合并过程中识别出重复的数据块，并只写入一次。
*   **数据结构**: `EncodeMap`, `SegmentOffsetMap` 等为实现去重而引入的关键数据结构。
*   **两阶段合并**: 如何通过先合并数据内容，再回填文档偏移量的方式，实现数据共享。
*   **Patch 处理的特殊性**: 在唯一编码的场景下，Patch 合并如何与主数据合并进行交互。

理解此合并器是掌握 Indexlib 高级存储优化的关键，它展示了如何在性能和空间效率之间做出精妙的权衡。

## 2. 系统架构与设计动机

### 2.1. 总体架构

`UniqEncodedMultiValueAttributeMerger<T>` 继承自 `MultiValueAttributeMerger<T>`，并重写了其核心的 `MergeData` 方法。其架构引入了更复杂的逻辑来实现数据去重。

在常规的多值属性中，`offset` 文件直接指向 `data` 文件。而在唯一编码的多值属性中，`offset` 文件中的值不再是 `data` 文件中的物理偏移，而是一个“逻辑偏移”或者说是一个“值 ID”。真正的数据和其物理偏移被存储在一个独立的、去重后的数据区。`UniqEncodedMultiValueAttributeMerger` 的任务就是在合并过程中构建这个新的、去重后的数据区，并为每个文档记录正确的“逻辑偏移”。

其合并流程可以概括为以下几个核心步骤：

1.  **合并补丁 (`MergePatchesInMergingSegments`)**: 首先，它会优先处理所有源 Segment 中的 Patch 数据。对于那些被 Patch 更新过的文档，其新的、唯一编码后的值会直接被计算并写入目标数据区。这些文档会被标记，在后续步骤中不再处理。
2.  **构建偏移映射 (`ConstructSegmentOffsetMap`)**: 遍历所有源 Segment 中未被 Patch 处理的文档，读取它们旧的（去重前的）数据偏移量，并构建一个 `SegmentOffsetMap`。这个 map 存储了 `(oldOffset, oldDocId, targetSegmentId)` 的元组，并按 `oldOffset` 排序。这一步的目的是将拥有相同数据内容（即相同 `oldOffset`）的文档聚合在一起。
3.  **合并去重数据 (`MergeOneSegmentData`)**: 遍历排序后的 `SegmentOffsetMap`。对于每一个唯一的 `oldOffset`，从源数据文件中读取其对应的数据块，然后调用 `FindOrGenerateNewOffset`。这个函数会：
    a.  计算数据块的哈希值。
    b.  在目标 `VarLenDataWriter` 的内部哈希表（`EncodeMap`）中查找该哈希值。如果找到，说明这个数据块已经被写入过了，直接返回其已有的新偏移量 `newOffset`。
    c.  如果没找到，就将这个新的数据块写入目标数据文件，并将 `(hash, newOffset)` 存入 `EncodeMap`，然后返回 `newOffset`。
    d.  将得到的 `newOffset` 更新回 `SegmentOffsetMap` 中。
4.  **回填文档偏移 (`MergeNoPatchedDocOffsets`)**: 经过上一步，`SegmentOffsetMap` 中已经记录了每个 `oldOffset` 到 `newOffset` 的映射。现在，再次遍历所有未被 Patch 的文档，根据其 `oldOffset` 从 `SegmentOffsetMap` 中查找到对应的 `newOffset`，并调用 `dataWriter->SetOffset()` 将这个 `newOffset` 写入该文档在目标偏移量文件中的位置。

### 2.2. 设计动机

*   **极致的空间压缩**: 这是最核心的动机。对于多值数据重复率高的场景，唯一编码能带来数倍甚至数十倍的存储空间节省，从而降低成本，并可能因为数据量减少而提升缓存效率。
*   **分而治之**: 合并过程被巧妙地分解为“处理有补丁的文档”、“处理无补丁文档的数据内容”、“回填无补丁文档的偏移”三个阶段。这种分解使得极其复杂的逻辑变得可以管理。特别是将数据内容的合并和文档偏移的写入分开，是实现去重和共享的关键所在。
*   **性能与空间的权衡**: 去重过程引入了额外的计算（哈希）、查找（`EncodeMap`）和数据结构（`SegmentOffsetMap`），这会比常规的多值合并消耗更多的 CPU 和内存。这是一个典型的用计算换空间的例子。设计者认为，对于许多应用场景，节省的大量存储空间是值得的。
*   **利用现有组件**: 该合并器依然建立在 `VarLenDataWriter` 之上，但巧妙地利用了其 `AppendValueWithoutOffset` 和 `SetOffset` 等高级接口，而不是简单的 `AppendValue`。这体现了 Indexlib 组件化设计的良好复用性。

## 3. 关键实现细节

### 3.1. 重写的 `MergeData` 与两阶段流程

`UniqEncodedMultiValueAttributeMerger` 的 `MergeData` 方法是其与父类的最大区别，它完整地展现了上述的复杂流程。

```cpp
// indexlib/index/attribute/merger/UniqEncodedMultiValueAttributeMerger.h
template <typename T>
inline Status UniqEncodedMultiValueAttributeMerger<T>::MergeData(const std::shared_ptr<DocMapper>& docMapper,
                                                                 const IIndexMerger::SegmentMergeInfos& segMergeInfos)
{
    DocIdSet patchedGlobalDocIdSet;
    // 阶段一：处理所有带 patch 的文档
    auto st = MergePatchesInMergingSegments(segMergeInfos, docMapper, patchedGlobalDocIdSet);
    RETURN_IF_STATUS_ERROR(st, "...");

    OffsetMapVec offsetMapVec(segMergeInfos.srcSegments.size());
    // 阶段二：处理无 patch 的文档
    st = MergeData(docMapper, segMergeInfos, patchedGlobalDocIdSet, offsetMapVec);
    RETURN_IF_STATUS_ERROR(st, "...");
    st = MergeNoPatchedDocOffsets(segMergeInfos, docMapper, offsetMapVec, patchedGlobalDocIdSet);
    RETURN_IF_STATUS_ERROR(st, "...");
    return Status::OK();
}
```

这个 `MergeData` 实际上是一个调度器，它调用的 `MergeData` (重载版本) 和 `MergeNoPatchedDocOffsets` 才是真正干活的。

### 3.2. `ConstructSegmentOffsetMap` 与 `MergeOneSegmentData`：数据内容的去重合并

这是实现去重的核心。首先，`ConstructSegmentOffsetMap` 聚合了所有具有相同 `oldOffset` 的文档信息。

```cpp
// indexlib/index/attribute/merger/UniqEncodedMultiValueAttributeMerger.h
template <typename T>
Status UniqEncodedMultiValueAttributeMerger<T>::ConstructSegmentOffsetMap(...)
{
    // ...
    for (docid_t oldDocId = 0; oldDocId < (int64_t)docCount; ++oldDocId) {
        // ... 跳过被删除或已打 patch 的文档 ...
        auto [st, oldOffset] = typedCtx->offsetReader.GetOffset(oldDocId);
        // 记录 (旧偏移, 旧文档ID, 目标SegmentID)
        segmentOffsetMap.emplace_back(OffsetPair(oldOffset, uint64_t(-1), oldDocId, targetSegmentId));
    }
    // ...
    std::sort(segmentOffsetMap.begin(), segmentOffsetMap.end());
    // 通过 unique 去除 (oldOffset, targetSegmentId) 的重复项
    segmentOffsetMap.assign(segmentOffsetMap.begin(), std::unique(segmentOffsetMap.begin(), segmentOffsetMap.end()));
    return Status::OK();
}
```
排序和 `unique` 操作是关键，它确保了对于每一个源 Segment 中的唯一数据块，我们只处理一次。

然后，`MergeOneSegmentData` 遍历这个 `map`，为每个唯一的数据块生成一个新的、去重后的偏移量。

```cpp
// indexlib/index/attribute/merger/UniqEncodedMultiValueAttributeMerger.h
template <typename T>
inline Status UniqEncodedMultiValueAttributeMerger<T>::MergeOneSegmentData(...)
{
    // ...
    for (; it != segmentOffsetMap.end(); ++it) {
        // ... 从文件流中根据 oldOffset 读取数据到 _dataBuf ...
        diskIndexer->FetchValueFromStreamNoCopy(it->oldOffset, ...);
        
        // 查找或生成新的偏移量
        auto [st, offset] = FindOrGenerateNewOffset((uint8_t*)_dataBuf.GetBuffer(), dataLen, *output);
        RETURN_IF_STATUS_ERROR(st, "...");
        it->newOffset = offset; // 将新的偏移量记录回 map 中
    }
    return Status::OK();
}

template <typename T>
inline std::pair<Status, uint64_t>
UniqEncodedMultiValueAttributeMerger<T>::FindOrGenerateNewOffset(uint8_t* dataBuf, uint32_t dataLen, OutputData& output)
{
    autil::StringView value((const char*)dataBuf, dataLen);
    uint64_t hashValue = output.dataWriter->GetHashValue(value);
    // 核心：AppendValueWithoutOffset 会在内部使用 hash 表去重
    auto [st, offset] = output.dataWriter->AppendValueWithoutOffset(value, hashValue);
    RETURN2_IF_STATUS_ERROR(st, 0, "...");
    return std::make_pair(Status::OK(), offset);
}
```

### 3.3. `MergeNoPatchedDocOffsets`：回填偏移量

在数据内容处理完毕，并且 `offsetMapVec` 中已经填充了所有 `oldOffset -> newOffset` 的映射之后，这个函数负责最后的回填工作。

```cpp
// indexlib/index/attribute/merger/UniqEncodedMultiValueAttributeMerger.h
template <typename T>
inline Status UniqEncodedMultiValueAttributeMerger<T>::MergeNoPatchedDocOffsets(...)
{
    // ...
    DocumentMergeInfoHeap heap;
    heap.Init(segMergeInfos, docMapper);
    DocumentMergeInfo info;
    while (heap.GetNext(info)) { // 再次遍历所有文档
        // ... 跳过已打 patch 的文档 ...
        
        // 获取该文档的 oldOffset
        auto [st, oldOffset] = typedCtx->offsetReader.GetOffset(oldDocId - ...);
        
        const SegmentOffsetMap& offsetMap = offsetMapVec[segIdx];
        // 在 map 中查找 oldOffset 对应的 newOffset
        typename SegmentOffsetMap::const_iterator it = std::lower_bound(...);
        assert(it != offsetMap.end() && ...);
        
        // 将查找到的 newOffset 写入该文档在目标偏移文件中的位置
        output->dataWriter->SetOffset(targetDocId, it->newOffset);
    }
    return Status::OK();
}
```
这个函数通过 `lower_bound` 在已排序的 `SegmentOffsetMap` 中进行二分查找，这比使用哈希表查找更节省内存，尽管时间复杂度稍高（`O(logN)` vs `O(1)`）。查找到 `newOffset` 后，调用 `SetOffset` 将其写入 `targetDocId` 对应的位置。至此，整个合并过程完成。

## 4. 技术风险与挑战

1.  **逻辑极其复杂**: 这是该合并器最大的挑战。两阶段的合并、多种数据结构的协同工作、对 Patch 的特殊处理，使得代码的理解和维护成本非常高。任何微小的逻辑错误都可能导致数据不一致或损坏。
2.  **内存消耗**: `SegmentOffsetMap` 和 `EncodeMap`（在 `VarLenDataWriter` 内部）都需要消耗大量内存。特别是 `SegmentOffsetMap`，其大小与源 Segment 中（未被 Patch 的）文档总数成正比，在非常大的 Segment 合并时可能成为内存瓶颈。
3.  **性能开销**: 相比常规合并，增加了排序、查找、哈希计算等操作，CPU 开销更大。并且，对文档进行了两次遍历（一次构建 map 和合并数据，一次回填偏移），I/O 次数也相应增加。只有在空间节省带来的收益足够大时，这些开销才是合理的。

## 5. 总结

`UniqEncodedMultiValueAttributeMerger<T>` 是 Indexlib 中一个高度复杂且精巧的工程实现。它以增加 CPU 和内存开销为代价，实现了对多值属性数据的有效去重，从而在特定场景下极大地节省了存储空间。

其核心设计思想——**“先合并去重内容，再回填文档指针”** 的两阶段策略，是解决“多对一”共享数据合并问题的经典方案。尽管逻辑复杂，但其带来的空间优化效果在很多业务场景下是不可或缺的。

该合并器的实现，充分体现了 Indexlib 工程师在面对复杂业务需求时，如何在性能、空间、可维护性之间进行权衡，并最终设计出满足要求的高级优化方案的能力。
