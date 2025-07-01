
# Indexlib DeletionMap 读取核心：架构设计与实现解析

**涉及文件:**
*   `indexlib/index/normal/deletionmap/deletion_map_reader.cpp`
*   `indexlib/index/normal/deletionmap/deletion_map_reader.h`
*   `indexlib/index/normal/deletionmap/in_mem_deletion_map_reader.cpp`
*   `indexlib/index/normal/deletionmap/in_mem_deletion_map_reader.h`
*   `indexlib/index/normal/deletionmap/deletion_map_reader_adaptor.h`

## 1. 引言：DeletionMap 的“读”操作核心

在复杂的搜索引擎和数据处理系统中，数据的删除是一个普遍且关键的操作。Indexlib 中的 DeletionMap（删除图）模块，正是为了高效管理文档删除状态而设计的。它能够记录哪些文档已被标记为删除，从而在检索、合并等后续环节中可以正确地过滤这些无效数据。

本文将深入剖析 DeletionMap 的**读取核心**，即 `DeletionMapReader` 及其相关组件。这部分是 DeletionMap 功能的“出口”，负责向上层应用（如查询引擎）提供一个统一、高效的接口，用于判断任意一个文档ID（`docid_t`）是否已被删除。它的设计直接影响着整个系统的查询性能和数据一致性。

我们将从以下几个方面展开分析：
*   **统一视图的设计理念**：探讨 `DeletionMapReader` 如何巧妙地将分布在不同物理存储（磁盘上的持久化段和内存中的实时更新）的删除信息整合成一个单一、连续的逻辑视图。
*   **分层与组合的架构**：解析 `DeletionMapReader`、`InMemDeletionMapReader` 和 `DeletionMapReaderAdaptor` 之间如何通过分层与组合，共同构建出一个既能处理历史数据又能响应实时删除的灵活架构。
*   **核心数据结构与算法**：深入 `BitMap` 这一核心数据结构，分析其在空间效率和查询速度上的优势，并揭示其背后的位操作技巧。
*   **性能与线程安全**：分析其在多线程环境下的读写性能考量，以及如何通过无锁化设计来保证高并发场景下的高效查询。

通过本次分析，读者将能清晰地理解 DeletionMap 读取机制的底层工作原理，洞察其在设计上的权衡与考量，并为基于 Indexlib 进行二次开发或性能优化提供坚实的理论基础。

## 2. 整体架构：一个统一的删除视图

DeletionMap 的读取架构核心目标是：**无论删除信息存储在哪里（内存或磁盘），都为上层调用者提供一个统一的、基于全局文档ID的查询视图**。用户只需调用 `DeletionMapReader::IsDeleted(docid_t docId)`，就能知道这个文档是否被删除，而无需关心其背后的复杂性。

为了实现这一目标，系统采用了**分层**和**组合**的设计模式，主要由以下三个关键类协同工作：

*   **`DeletionMapReader` (主读取器)**：这是最顶层的接口类。它本身不直接存储删除数据，而是扮演一个“协调者”的角色。它内部维护了一个 `InMemDeletionMapReader` 的实例（用于处理内存中的增量删除）和一个 `DeletionMapReaderAdaptor` 的实例（用于处理磁盘上已持久化的段数据）。当查询请求到来时，它会根据 `docId` 所属的范围，智能地将请求分发给合适的底层读取器。

*   **`InMemDeletionMapReader` (内存读取器)**：这个类专门负责读取**内存中**的删除信息。在索引构建（Building）过程中，新的删除操作会首先被记录在内存的一个 `BitMap` 中。`InMemDeletionMapReader` 直接访问这个内存 `BitMap`，提供对最新删除状态的实时查询。它的存在是实现“近实时”（Near Real-Time）删除的关键。

*   **`DeletionMapReaderAdaptor` (磁盘适配器)**：这个类是连接 `DeletionMapReader` 和磁盘上各个独立段（Segment）的桥梁。一个索引通常由多个段构成，每个段都可能有自己的删除图文件。`DeletionMapReaderAdaptor` 内部持有一个或多个 `DeletionMapSegmentReader`（虽然代码中未直接出现，但逻辑上存在）的引用，这些 `SegmentReader` 负责加载并查询对应段的删除图文件。`DeletionMapReaderAdaptor` 的核心职责是根据全局 `docId`，计算出它属于哪个段（Segment）以及在段内的局部 `docid_t`，然后调用相应段的读取器来完成查询。

这种架构的精妙之处在于：

1.  **职责分离**：每个类都有明确的职责。`DeletionMapReader` 负责统一调度，`InMemDeletionMapReader` 负责实时性，`DeletionMapReaderAdaptor` 负责历史数据的访问。这使得代码结构清晰，易于维护和扩展。
2.  **透明性**：上层调用者完全无需感知数据的物理分布。无论是刚被删除的文档（信息在内存中），还是很久以前被删除的文档（信息在磁盘上），查询方式完全一样。
3.  **高性能**：通过 `docId` 范围判断，查询可以直接路由到目标位置，避免了不必要的遍历。例如，一个 `docId` 如果小于当前内存中记录的起始 `docId`，那么查询会直接被导向 `DeletionMapReaderAdaptor`，而不会访问内存部分。

### 核心查询流程

当调用 `DeletionMapReader::IsDeleted(docid_t docId)` 时，其内部逻辑如下：

1.  **首先检查内存**：`DeletionMapReader` 会先将查询请求转发给 `InMemDeletionMapReader`。
2.  `InMemDeletionMapReader` 判断 `docId` 是否落在其负责的 `docId` 范围内（即当前正在构建的段）。
    *   如果是，则直接在内存的 `BitMap` 中查找并返回结果。
    *   如果不是，表示该 `docId` 属于已经持久化的历史段。
3.  **然后查询磁盘**：若内存中未命中，`DeletionMapReader` 会将请求继续转发给 `DeletionMapReaderAdaptor`。
4.  `DeletionMapReaderAdaptor` 根据 `docId` 和各段的文档数量信息，定位到该 `docId` 所在的具体段。
5.  调用该段对应的 `DeletionMapSegmentReader`，传入段内 `docId`，从该段加载的 `BitMap` 中查找并返回结果。
6.  如果所有部分都查询不到（例如，`docId` 超出范围或对应段没有删除图），则认为该文档未被删除。

这个流程确保了查询的完整性和正确性，覆盖了所有可能的删除数据来源。

## 3. 关键实现细节

### 3.1 `DeletionMapReader`：顶层协调者

`DeletionMapReader` 是整个读取体系的入口。它的实现虽然不复杂，但却是连接内存与磁盘的关键。

```cpp
// indexlib/index/normal/deletionmap/deletion_map_reader.h

class DeletionMapReader
{
public:
    // ...
    bool IsDeleted(docid_t docId) const;
    // ...
private:
    std::shared_ptr<InMemDeletionMapReader> mInMemDeletionMapReader;
    std::shared_ptr<DeletionMapReaderAdaptor> mReaderAdaptor;
};

// indexlib/index/normal/deletionmap/deletion_map_reader.cpp

bool DeletionMapReader::IsDeleted(docid_t docId) const
{
    if (mInMemDeletionMapReader && mInMemDeletionMapReader->IsDeleted(docId)) {
        return true;
    }
    if (mReaderAdaptor) {
        return mReaderAdaptor->IsDeleted(docId);
    }
    return false;
}
```

**核心代码分析:**

*   `IsDeleted(docid_t docId) const` 方法的实现清晰地体现了其**协调者**的角色。
*   它首先调用 `mInMemDeletionMapReader` 进行查询。`mInMemDeletionMapReader` 内部会判断 `docId` 是否在其管辖范围内，如果不是，会快速返回 `false`，此时控制流才会继续。
*   如果内存读取器没有命中（返回 `false`），查询会继续传递给 `mReaderAdaptor`，由它来处理对磁盘上持久化段的查询。
*   这种**短路求值**的逻辑（`&&` 和 `||` 的特性）确保了查询的效率。一旦在内存中确认文档被删除，就不会再进行后续的磁盘查询。

这种设计模式非常经典，它将两个不同来源的数据源（内存实时数据和磁盘历史数据）通过一个统一的接口暴露出去，实现了逻辑上的聚合。

### 3.2 `InMemDeletionMapReader`：实时删除的哨兵

`InMemDeletionMapReader` 负责处理正在构建中的段（Building Segment）的删除信息。这些信息是“实时”的，尚未被持久化到磁盘。

```cpp
// indexlib/index/normal/deletionmap/in_mem_deletion_map_reader.h

class InMemDeletionMapReader
{
public:
    // ...
    bool IsDeleted(docid_t docId) const;
private:
    const util::BitMap* mInMemDeletionMap;
    docid_t mStartDocId;
    uint32_t mRecordCount;
};

// indexlib/index/normal/deletionmap/in_mem_deletion_map_reader.cpp

bool InMemDeletionMapReader::IsDeleted(docid_t docId) const
{
    if (docId >= mStartDocId && docId < mStartDocId + (docid_t)mRecordCount) {
        return mInMemDeletionMap->Test(docId - mStartDocId);
    }
    return false;
}
```

**核心代码分析:**

*   **`mInMemDeletionMap`**: 这是一个指向 `util::BitMap` 的指针。`BitMap` 是一个位图，用一个 bit 来表示一个文档是否被删除，空间效率极高。`InMemDeletionMapReader` 本身不拥有这个 `BitMap`，而是持有其只读指针，数据由 `DeletionMapWriter` 在写入时更新。
*   **`mStartDocId`**: 这是一个关键字段，表示当前内存删除图所对应的起始全局 `docId`。例如，如果之前的段共有 10000 个文档，那么新段的 `mStartDocId` 就是 10000。
*   **`mRecordCount`**: 记录了当前内存删除图覆盖的文档数量。
*   **`IsDeleted` 的逻辑**:
    1.  **范围检查**: `if (docId >= mStartDocId && docId < mStartDocId + (docid_t)mRecordCount)` 这个判断至关重要。它首先确定查询的 `docId` 是否属于当前内存段。如果不是，直接返回 `false`，避免了无效的位图访问。这使得 `DeletionMapReader` 中的级联查询效率很高。
    2.  **本地ID转换与查询**: 如果 `docId` 在范围内，通过 `docId - mStartDocId` 将全局 `docId` 转换为段内的本地 `docId`。然后调用 `mInMemDeletionMap->Test(...)` 在位图中进行查询。`Test` 操作通常是一个极快的位运算，时间复杂度为 O(1)。

### 3.3 `DeletionMapReaderAdaptor`：磁盘数据的访问入口

`DeletionMapReaderAdaptor` 是一个适配器，它的作用是将对全局 `docId` 的查询，适配到对某个具体磁盘段的查询。它内部管理了所有已提交段（Built Segments）的删除信息。

```cpp
// indexlib/index/normal/deletionmap/deletion_map_reader_adaptor.h

class DeletionMapReaderAdaptor
{
public:
    // ...
    bool IsDeleted(docid_t docId) const;
    void AddSegmentReader(docid_t baseDocId, const std::shared_ptr<DeletionMapSegmentReader>& segReader);
private:
    // a vector of <baseDocId, segmentReader>
    typedef std::pair<docid_t, std::shared_ptr<DeletionMapSegmentReader>> SegmentReaderPair;
    std::vector<SegmentReaderPair> mSegmentReaders;
    // ...
};
```

**核心逻辑推演 (基于设计模式和上下文):**

虽然 `DeletionMapReaderAdaptor::IsDeleted` 的完整实现未在本次提供的代码中完全展示，但我们可以根据其设计目标和成员变量推断出其核心逻辑：

1.  **内部数据结构**: `mSegmentReaders` 是一个 `std::vector`，存储了每个段的起始 `docId` (`baseDocId`) 和对应的段读取器 (`DeletionMapSegmentReader`)。这个 `vector` 很可能是按照 `baseDocId` 有序的。
2.  **查询算法**: 当 `IsDeleted(docid_t docId)` 被调用时：
    *   它会使用 `docId` 在 `mSegmentReaders` 中进行查找。由于 `mSegmentReaders` 是有序的，这里很可能会使用**二分查找**（`std::upper_bound` 或类似算法）来高效地定位 `docId` 属于哪个段。
    *   例如，如果查找到 `mSegmentReaders[i]` 的 `baseDocId` 是第一个大于 `docId` 的，那么 `docId` 就必然属于 `mSegmentReaders[i-1]` 这个段。
    *   一旦定位到段 `i-1`，它就获得了该段的 `baseDocId` 和 `DeletionMapSegmentReader`。
    *   然后，它计算出段内 `docId`：`localDocId = docId - mSegmentReaders[i-1].first`。
    *   最后，调用 `mSegmentReaders[i-1].second->IsDeleted(localDocId)` 来完成最终的查询。

这种基于**分段+二分查找**的策略，使得即使在有大量段的情况下，也能保持极高的查询效率，其时间复杂度约为 O(log N)，其中 N 是段的数量。

## 4. 核心数据结构：`util::BitMap`

DeletionMap 的性能基石是 `util::BitMap`。位图是一种非常紧凑的数据结构，它利用一个比特位（bit）的是 `0` 还是 `1` 来表示某个元素的状态。在 DeletionMap 的场景下，一个比特位对应一个文档，`1` 表示已删除，`0` 表示未删除（或反之，具体取决于实现）。

**优势:**

1.  **极高的空间效率**: 存储 N 个文档的删除状态，理论上只需要 N/8 个字节。相比于使用 `bool` 数组（每个 `bool` 占 1 字节）或 `std::vector<bool>`（有额外开销），`BitMap` 的空间利用率要高得多。这对于管理上亿级别的文档集合至关重要，能显著减少内存占用和磁盘IO。
2.  **O(1) 的查询复杂度**: 判断一个文档是否被删除，只需要通过位运算直接访问对应的比特位。具体来说，`Test(docId)` 操作可以分解为：
    *   计算 `docId` 在哪个 `uint32_t`（或 `uint64_t`）的槽（slot）里：`slot_index = docId / 32`。
    *   计算 `docId` 在该槽内的比特位偏移：`bit_offset = docId % 32`。
    *   执行位与操作：`return (slots[slot_index] & (1 << bit_offset)) != 0;`。
    这些都是CPU指令级别的操作，速度极快。

**技术风险与考量:**

*   **线程安全**: `BitMap` 本身通常不是线程安全的。在 DeletionMap 的应用场景中，读取操作（`Test`）是高频的，而写入操作（`Set`）相对低频。读取核心（`DeletionMapReader`）的设计是**只读**的，它持有的 `BitMap` 指针是 `const` 的。写操作由 `DeletionMapWriter` 负责，并通过某种机制（如 Copy-on-Write 或版本控制）来保证数据一致性，从而实现了高并发下的无锁读取。
*   **动态扩展**: 原始的位图大小是固定的。如果需要动态增加文档，就需要重新分配更大的内存并拷贝旧数据，这可能是一个耗时操作。在 Indexlib 的设计中，通过分段（Segment）的机制巧妙地规避了这个问题。每个段的 `BitMap` 大小是固定的，当需要增加文档时，是创建一个新的段和新的 `BitMap`，而不是去扩展旧的。

## 5. 总结与展望

DeletionMap 的读取核心模块通过**分层、组合和适配器模式**，成功地将物理上分离的删除信息（内存实时数据与磁盘历史数据）抽象成一个统一的逻辑视图。其架构清晰，职责明确，具有良好的可扩展性。

*   **`DeletionMapReader`** 作为顶层门面，简化了上层调用。
*   **`InMemDeletionMapReader`** 保证了删除操作的近实时性。
*   **`DeletionMapReaderAdaptor`** 通过高效的查找算法，提供了对海量历史数据的快速访问。
*   底层依赖的 **`util::BitMap`** 数据结构，则从根本上保证了空间和时间效率。

这种设计体现了大型工业级系统在处理复杂数据状态时的典型思路：**用抽象来屏蔽复杂性，用分层来隔离变化，用高效的数据结构来保证性能**。

未来的潜在优化方向可能包括：
*   **压缩位图**: 对于稀疏的删除（即删除的文档比例很低），可以采用如 Roaring Bitmaps 等压缩位图算法，进一步降低存储开销。
*   **硬件加速**: 利用 SIMD 指令集（如 AVX2, AVX-512）可以并行处理更多的位操作，从而在一次CPU周期内判断多个文档的删除状态，对于需要批量检查的场景（如查询结果过滤）能带来显著的性能提升。
