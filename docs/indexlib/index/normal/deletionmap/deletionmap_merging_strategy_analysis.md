
# Indexlib DeletionMap 合并策略：架构设计与实现解析

**涉及文件:**
*   `indexlib/index/normal/deletionmap/deletion_map_merger.cpp`
*   `indexlib/index/normal/deletionmap/deletion_map_merger.h`

## 1. 引言：合并，索引优化的核心

在像 Indexlib 这样的 LSM-Tree (Log-Structured Merge-Tree) 架构的索引系统中，数据以增量段（Segment）的形式不断写入。随着时间推移，系统会积累大量的小段。为了控制段的数量、回收被删除文档占用的空间、优化查询性能，必须定期执行**合并（Merge）**操作，将多个旧段融合成一个或多个新的、更大的段。

在合并过程中，所有组件的数据都需要被妥善处理，DeletionMap 也不例外。`DeletionMapMerger` 正是为此而生，它负责处理 DeletionMap 在段合并期间的复杂逻辑。它的任务远比听起来要复杂：不仅仅是把几个删除列表拼在一起，更关键的是，它必须处理因文档“搬家”而导致的**文档ID（docid_t）的重新映射（Remapping）**。

本文将深入剖析 `DeletionMapMerger` 的工作机制，重点关注：
*   **合并的必要性与挑战**：为什么 DeletionMap 需要合并？合并过程中最大的挑战是什么（即ID重映射）？
*   **核心合并流程**：`DeletionMapMerger` 如何从多个源段（Source Segments）读取删除信息，并结合ID重映射表，生成一个全新的、针对目标段（Target Segment）的删除图。
*   **与上层框架的交互**：`DeletionMapMerger` 如何与更宏观的合并框架（如 `IndexMerger`）协同工作，获取必要的信息（如 `SegmentMergeInfos` 和 `ReclaimMap`）。
*   **性能与资源考量**：在处理可能涉及上亿文档的合并任务时，`DeletionMapMerger` 在性能和内存使用方面有何设计考量。

理解 `DeletionMapMerger` 的工作原理，对于掌握 Indexlib 的数据整理和空间优化机制至关重要，也是理解其如何维持长期高性能查询能力的关键。

## 2. 整体架构：一个ID重映射驱动的合并器

`DeletionMapMerger` 的核心架构是围绕**ID重映射**来构建的。在段合并时，只有那些未被删除的“存活”文档才会被拷贝到新的目标段中。这意味着，一个文档在旧段中的ID和在新段中的ID是不同的。例如，旧段A中的第100号文档，如果它前面有20个文档被删除了，那么它在新段中可能就变成了第80号文档。

这个新旧ID的对应关系，由一个名为 `ReclaimMap`（回收图）或类似的重映射表来维护。`DeletionMapMerger` 的核心使命就是：**遍历所有参与合并的旧段中的每一个文档，判断其是否存活，如果存活，则在新段的删除图中将其标记为“未删除”**。最终，所有未被标记为“未删除”的文档，在新段中自然就是“已删除”状态（尽管它们并未被拷贝过来）。

这个逻辑听起来可能有点绕，我们换个角度理解：合并后的新段，其删除图的初始状态是“全删除”（所有位都为1）。然后，`DeletionMapMerger` 像一个“赦免官”，根据 `ReclaimMap` 将那些从旧段迁移过来的、存活的文档，在新删除图的对应位置上标记为“未删除”（将位设置为0）。

### 合并流程概览

1.  **初始化 (Initialization)**：`DeletionMapMerger` 由上层合并框架创建，并传入 `SegmentMergeInfos`。这个结构体包含了所有待合并段的信息，如每个段的 `Directory`、起始 `docId`、文档总数等。

2.  **开始合并 (BeginMerge)**：上层框架调用 `BeginMerge`，并传入一个 `ReclaimMap` 的向量。`ReclaimMap` 是关键输入，它告诉 `DeletionMapMerger` 每个旧段中的哪些文档被保留了，以及它们在新段中的新 `docId` 是什么。

3.  **执行合并 (Merge)**：这是核心步骤。`DeletionMapMerger` 会：
    a.  创建一个新的、大小为目标段总文档数的 `BitMap`，并将其所有位初始化为 `1`（代表全部删除）。
    b.  遍历每一个参与合并的旧段。
    c.  对于每个旧段，它会遍历该段的所有文档（从 `docId` 0 到 `N-1`）。
    d.  对于每个旧文档，它会查询 `ReclaimMap`，判断该文档是否被保留。
    e.  如果文档被保留，`ReclaimMap` 会返回其在新段中的新 `docId`。`DeletionMapMerger` 就会在新 `BitMap` 中，将这个新 `docId` 对应的位设置为 `0`（标记为未删除）。
    f.  遍历完所有旧段的所有文档后，新 `BitMap` 的状态就正确反映了合并后新段的删除情况。

4.  **结束合并 (EndMerge)**：合并完成后，`DeletionMapMerger` 会将生成的新 `BitMap` 封装成一个 `DeletionMapDumpItem`，并提交给持久化流程，由后台线程将其写入新段的目录中。

## 3. 关键实现细节

`DeletionMapMerger` 的实现精确地遵循了上述逻辑，并处理了许多边界情况。

```cpp
// indexlib/index/normal/deletionmap/deletion_map_merger.h

class DeletionMapMerger
{
public:
    // ...
    void BeginMerge(const index::SegmentMergeInfos& segMergeInfos);
    void Merge(const index::MergerResource& resource, const index::SegmentMergeInfos& segMergeInfos,
               const index::OutputSegmentMergeInfos& outputSegMergeInfos);
    void EndMerge();

private:
    std::vector<std::shared_ptr<DeletionMapReader>> mDeletionMapReaders;
    std::vector<std::shared_ptr<const ReclaimMap>> mReclaimMaps;
    std::vector<docid_t> mBaseDocIds;
    std::shared_ptr<file_system::Directory> mTargetDirectory;
    // ...
};

// indexlib/index/normal/deletionmap/deletion_map_merger.cpp

void DeletionMapMerger::Merge(const index::MergerResource& resource, const index::SegmentMergeInfos& segMergeInfos,
                              const index::OutputSegmentMergeInfos& outputSegMergeInfos)
{
    // ... (获取 ReclaimMap 和目标段信息)
    const auto& reclaimMap = resource.reclaimMap;
    const auto& targetSegment = outputSegMergeInfos[0];
    uint32_t targetDocCount = targetSegment.docCount;

    // 1. 创建并初始化新的删除图
    util::BitMap* newBitMap = new util::BitMap(targetDocCount);
    newBitMap->Mount(targetDocCount, true); // true 表示全部置为1 (删除)

    // 2. 遍历所有旧段，将被保留的文档在新图中标记为“未删除”
    docid_t newDocId = 0;
    for (size_t i = 0; i < segMergeInfos.size(); ++i) {
        const auto& segInfo = segMergeInfos[i];
        const auto& segReclaimMap = reclaimMap->GetSegmentReclaimMap(segInfo.segmentId);
        for (docid_t oldDocId = 0; oldDocId < (docid_t)segInfo.docCount; ++oldDocId) {
            if (segReclaimMap->IsDocDeleted(oldDocId)) { // 如果在旧段中就被删了，跳过
                continue;
            }
            // 从 ReclaimMap 获取新 docId
            docid_t mappedDocId = segReclaimMap->GetNewId(oldDocId);
            if (mappedDocId != INVALID_DOCID) {
                newBitMap->Reset(mappedDocId); // 在新图中标记为未删除
            }
        }
    }

    // 3. 创建 DumpItem 进行持久化
    auto dumpItem = std::make_shared<DeletionMapDumpItem>(newBitMap, DELETION_MAP_FILE_NAME);
    // ... (将 dumpItem 添加到持久化队列)
}
```

**核心代码分析:**

*   **`Merge` 方法**: 这是合并逻辑的核心所在。
    1.  **获取 `ReclaimMap`**: `resource.reclaimMap` 是从上层合并框架传入的关键资源。它包含了所有旧段到新段的ID映射信息。
    2.  **创建新位图**: `new util::BitMap(targetDocCount)` 创建了一个足够大的位图来容纳新段的所有文档。`newBitMap->Mount(targetDocCount, true)` 是一个关键操作，它将位图的所有位都设置为 `1`，即默认所有文档都是删除状态。这是一个非常聪明的技巧，它简化了后续的逻辑——我们只需要关心哪些文档是“活”的，并将它们的状态反转即可。
    3.  **双重循环遍历**: 代码通过一个外层循环遍历所有参与合并的段（`segMergeInfos`），和一个内层循环遍历段内的每一个 `oldDocId`。
    4.  **查询 `ReclaimMap`**: `segReclaimMap->IsDocDeleted(oldDocId)` 首先检查这个文档在合并前是否已经被标记为删除。如果是，就没必要处理它了。`segReclaimMap->GetNewId(oldDocId)` 则是查询这个旧 `docId` 对应的新 `docId`。如果一个文档被保留下来，这个方法会返回它在新段的 `docId`；如果被丢弃，则返回 `INVALID_DOCID`。
    5.  **重置位（Reset）**: `newBitMap->Reset(mappedDocId)` 是核心的“赦免”操作。它将新 `docId` (`mappedDocId`) 对应的比特位从 `1` 置为 `0`，表示这个文档在新段中是存活的、未被删除的。
    6.  **创建 `DumpItem`**: 遍历完成后，`newBitMap` 就包含了新段最终的、正确的删除状态。随后，它被封装进一个 `DeletionMapDumpItem` 中，准备进行异步的持久化。

*   **与 `DeletionMapReader` 的关系**: 在 `BeginMerge` 阶段（代码未完全展示），`DeletionMapMerger` 会为每个待合并段创建一个 `DeletionMapReader`。这是为了在合并逻辑中，能够查询到文档在合并之前的原始删除状态。例如，`ReclaimMap` 的构建本身可能就需要参考原始的删除图。

## 4. 设计考量与技术挑战

*   **ID重映射的中心地位**: 整个合并逻辑完全由 `ReclaimMap` 驱动。`DeletionMapMerger` 本身不决定哪个文档该保留，它只是一个忠实的执行者，根据 `ReclaimMap` 的“指令”来构建新的删除图。这体现了良好的分层设计，合并策略的决策（在 `ReclaimMap` 中体现）与执行（在 `DeletionMapMerger` 中体现）相分离。

*   **“反向”逻辑的巧妙之处**: 采用“先全置1，再逐个置0”的策略，比“先全置0，再逐个置1”要更简单、更健壮。因为在合并后，大量的文档ID会变得无效（被删除的文档），我们很难去一一标记它们。而存活的文档是少数，我们只需处理这些存活的文档即可。这种“反向”思维大大简化了算法的复杂性。

*   **内存与性能**: 合并是一个非常消耗资源的过程。`DeletionMapMerger` 需要为新段创建一个完整的位图，如果新段非常大（例如上亿文档），这个位图本身就会占用数十MB的内存。同时，对 `ReclaimMap` 的频繁查询和对新位图的随机写入（`Reset` 操作）也对CPU缓存有一定要求。尽管如此，由于所有操作都是在内存中进行的，其速度远快于磁盘IO，因此通常不会成为合并流程的瓶颈。

*   **数据一致性**: `DeletionMapMerger` 的正确性对数据一致性至关重要。如果合并出的删除图是错误的（例如，错误地将一个存活的文档标记为删除），那么这个文档将永久“消失”，无法被检索到。因此，与 `ReclaimMap` 的精确协同是保证数据不出错的生命线。

## 5. 总结与展望

`DeletionMapMerger` 是 Indexlib 索引维护体系中的关键一环。它通过一个由 `ReclaimMap` 驱动的、逻辑清晰的合并算法，解决了在段合并过程中因文档ID变化而带来的删除状态继承难题。

其核心设计思想可以总结为：
*   **职责分离**: 将ID重映射的决策与删除图的构建执行相分离。
*   **反向构建**: 通过“先全删，后赦免”的策略简化合并逻辑。
*   **内存操作优先**: 将复杂的映射和构建过程全部在内存中完成，然后通过 `DumpItem` 异步持久化，保证了合并过程的高效性。

`DeletionMapMerger` 的存在，使得 DeletionMap 能够完美地融入 Indexlib 的 LSM-Tree 架构中，支持索引的持续优化和空间的有效回收。它是保证 Indexlib 在长期运行后依然能保持“健康”和高性能的重要机制之一。
