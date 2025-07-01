
# Indexlib DeletionMap 写入与构建：架构设计与实现解析

**涉及文件:**
*   `indexlib/index/normal/deletionmap/deletion_map_writer.cpp`
*   `indexlib/index/normal/deletionmap/deletion_map_writer.h`
*   `indexlib/index/normal/deletionmap/deletion_map_segment_writer.cpp`
*   `indexlib/index/normal/deletionmap/deletion_map_segment_writer.h`

## 1. 引言：DeletionMap 的“写”操作核心

在数据索引系统中，除了高效的读取，可靠、高效的写入同样至关重要。对于 DeletionMap 而言，“写入”操作即响应外部请求，将指定的文档标记为删除状态。这个过程必须是原子的、线程安全的，并且能够与读取、持久化等流程无缝协作。

本文将深入剖析 Indexlib DeletionMap 的**写入与构建**机制，其核心是 `DeletionMapWriter` 和 `DeletionMapSegmentWriter`。这套组合构成了 DeletionMap 数据的“生产者”，负责接收删除信号，并将其准确地反映到内存中的数据结构上，为后续的持久化和实时查询提供数据源。

我们将重点探讨以下几个方面：
*   **写入流程的架构**：分析 `DeletionMapWriter` 和 `DeletionMapSegmentWriter` 如何协同工作，将一个简单的 `Delete(docId)` 调用转化为对内存位图的精确修改。
*   **内存中的数据结构**：聚焦于 `util::BitMap` 在写入场景下的应用，以及如何管理其生命周期和内存分配。
*   **持久化衔接**：揭示写入器如何与 `DeletionMapDumpItem` 交互，将内存中的“脏”数据打包，为最终刷写到磁盘做准备。
*   **线程安全与并发控制**：探讨在多线程环境下，系统如何保证对删除图写入操作的一致性和原子性，避免数据竞争和状态不一致。

通过对写入核心的分析，读者可以理解 DeletionMap 如何在索引构建（Building）阶段处理和暂存删除信息，洞察其在实时性、可靠性和性能之间的设计权衡，为理解 Indexlib 的近实时（NRT）索引机制打下坚实基础。

## 2. 整体架构：一个分层的写入体系

与读取核心类似，DeletionMap 的写入体系也采用了**分层**的设计，以实现职责分离和逻辑解耦。其核心目标是：**为上层应用提供一个简单的删除接口，同时在底层管理好内存分配、数据更新和持久化触发等复杂操作。**

*   **`DeletionMapWriter` (顶层写入器)**：这是暴露给上层模块（如 `IndexWriter`）的统一接口。它屏蔽了底层的段（Segment）细节，让调用者感觉像是在操作一个连续的、完整的删除图。`DeletionMapWriter` 内部持有一个或多个 `DeletionMapSegmentWriter` 的实例，并负责将删除请求路由到当前活跃的（Building）段写入器。

*   **`DeletionMapSegmentWriter` (段写入器)**：这是实际执行写入操作的“工人”。每个 `DeletionMapSegmentWriter` 与索引中的一个特定段（Segment）绑定，并独立管理该段的删除数据。它直接拥有一块内存（`util::BitMap`），所有对该段的删除操作最终都会落在这个 `BitMap` 上。此外，它还负责在适当的时机（如段被提交时）创建 `DeletionMapDumpItem`，从而启动数据的持久化流程。

这种分层架构带来了显著的优势：

1.  **简化上层逻辑**：上层模块无需关心当前有多少个段、文档ID如何映射到段内ID等问题。只需调用 `DeletionMapWriter::Delete(docId)` 即可。
2.  **隔离段级操作**：每个段的删除数据被封装在各自的 `DeletionMapSegmentWriter` 中，互不干扰。这使得段的生命周期管理（创建、提交、合并）变得更加清晰和安全。
3.  **易于扩展**：如果未来需要支持更复杂的写入逻辑，例如分区（Partitioned）删除图，可以在 `DeletionMapWriter` 和 `DeletionMapSegmentWriter` 之间引入新的层次，而无需大规模修改现有代码。

### 核心写入流程

当调用 `DeletionMapWriter::Delete(docid_t docId)` 时，其内部的简化流程如下：

1.  **定位段写入器**：`DeletionMapWriter` 根据 `docId` 判断它属于哪个段。在实时构建场景中，通常只有一个正在写入的活动段，因此这个定位过程非常直接。
2.  **转换文档ID**：`DeletionMapWriter` 将全局 `docId` 转换为该段的本地 `docId`。例如，如果当前活动段的起始 `docId` 是 10000，那么全局 `docId` 10005 对应的本地 `docId` 就是 5。
3.  **调用段写入器**：`DeletionMapWriter` 调用定位到的 `DeletionMapSegmentWriter` 的 `Delete(localDocId)` 方法。
4.  **更新位图**：`DeletionMapSegmentWriter` 在其内部的 `util::BitMap` 上执行 `Set(localDocId)` 操作，将对应文档的比特位置为 `1`，标记为删除。
5.  **原子性保证**：这个 `Set` 操作通常需要加锁（如 `autil::ThreadMutex`）来保证线程安全，因为多个线程可能同时调用 `Delete`。

这个流程确保了每一次删除操作都能准确、安全地记录在内存中，为后续的查询和持久化提供了可靠的数据基础。

## 3. 关键实现细节

### 3.1 `DeletionMapWriter`：统一的删除入口

`DeletionMapWriter` 作为顶层接口，其核心职责是接收全局 `docId` 并将其转发给正确的段写入器。

```cpp
// indexlib/index/normal/deletionmap/deletion_map_writer.h

class DeletionMapWriter
{
public:
    // ...
    void Delete(docid_t docId);
    void Dump(const std::shared_ptr<file_system::Directory>& directory) override;

private:
    // ...
    std::shared_ptr<DeletionMapSegmentWriter> mBuildingSegmentWriter;
    docid_t mBaseDocId;
};

// indexlib/index/normal/deletionmap/deletion_map_writer.cpp

void DeletionMapWriter::Delete(docid_t docId)
{
    if (!mBuildingSegmentWriter) { // 如果没有正在构建的段，直接忽略
        return;
    }
    docid_t baseDocId = mBaseDocId;
    if (docId >= baseDocId) { // 确保 docId 属于当前段
        mBuildingSegmentWriter->Delete(docId - baseDocId);
    }
}
```

**核心代码分析:**

*   `mBuildingSegmentWriter`: 指向当前唯一活跃的段写入器。在 Indexlib 的模型中，通常只有一个段处于“正在构建”（Building）状态，接收新的写入。
*   `mBaseDocId`: 记录了当前活动段的起始全局 `docId`。
*   **`Delete(docid_t docId)`**: 逻辑非常清晰。它首先检查 `mBuildingSegmentWriter` 是否存在。然后，通过 `docId >= baseDocId` 判断该删除请求是否合法（即是否属于当前段）。如果合法，就将全局 `docId` 减去 `baseDocId` 得到段内 `docId`，并调用 `mBuildingSegmentWriter` 的 `Delete` 方法。这种设计将全局 `docId` 的处理逻辑收敛在了 `DeletionMapWriter`，使得 `DeletionMapSegmentWriter` 可以只关心段内逻辑。
*   **`Dump(...)`**: 这个方法体现了写入器与持久化流程的衔接。当上层模块（如 `InMemorySegment`) 决定将内存数据刷写到磁盘时，会调用 `DeletionMapWriter` 的 `Dump` 方法，而该方法会进一步调用 `mBuildingSegmentWriter` 的 `Dump` 方法来完成实际的持久化操作。

### 3.2 `DeletionMapSegmentWriter`：核心写入与持久化触发器

`DeletionMapSegmentWriter` 是真正干活的类。它管理着一个段的删除位图，并负责将其持久化。

```cpp
// indexlib/index/normal/deletionmap/deletion_map_segment_writer.h

class DeletionMapSegmentWriter
{
public:
    // ...
    void Delete(docid_t localDocId);
    std::shared_ptr<DeletionMapDumpItem> CreateDumpItem() const;

private:
    util::BitMap* mDeletionMap;
    // ...
    mutable autil::ThreadMutex mMutex;
};

// indexlib/index/normal/deletionmap/deletion_map_segment_writer.cpp

void DeletionMapSegmentWriter::Delete(docid_t localDocId)
{
    autil::ScopedLock lock(mMutex);
    if (!mDeletionMap->Set(localDocId)) {
        // ... handle error, e.g., localDocId out of bounds
    }
}

std::shared_ptr<DeletionMapDumpItem> DeletionMapSegmentWriter::CreateDumpItem() const
{
    autil::ScopedLock lock(mMutex);
    return std::make_shared<DeletionMapDumpItem>(mDeletionMap->Clone(), GetPatchFileName());
}
```

**核心代码分析:**

*   **`mDeletionMap`**: 一个指向 `util::BitMap` 的指针。与 `InMemDeletionMapReader` 不同，`DeletionMapSegmentWriter` **拥有**这个 `BitMap` 的所有权，负责其创建和销毁。
*   **`mMutex`**: 一个 `autil::ThreadMutex` 互斥锁。这是保证并发写入安全的关键。任何对 `mDeletionMap` 的修改操作都必须先获取这个锁。
*   **`Delete(docid_t localDocId)`**: 实现非常直接。它使用 `autil::ScopedLock` 在进入作用域时自动加锁，在离开时自动解锁。这是一种良好实践，可以有效避免忘记解锁导致的死锁问题。在锁的保护下，它调用 `mDeletionMap->Set(localDocId)` 来将相应的位设置为1。
*   **`CreateDumpItem() const`**: 这是连接写入与持久化的桥梁。当需要将内存数据转储到磁盘时，这个方法被调用。
    *   它同样在锁的保护下进行操作，以确保在创建 `DumpItem` 的过程中，`mDeletionMap` 不会被其他线程修改，保证了数据的一致性。
    *   核心操作是 `mDeletionMap->Clone()`。它创建了当前位图的一个完整拷贝。这意味着持久化操作将在一个数据快照上进行，而不会阻塞后续的 `Delete` 请求。这是一种**写时复制（Copy-on-Write）**思想的体现，提升了系统的并发性能。
    *   最后，它用克隆出的 `BitMap` 和目标文件名创建了一个 `DeletionMapDumpItem` 对象，并将其返回。这个 `DumpItem` 随后会被文件系统异步地写入磁盘。

## 4. 线程安全与并发模型

DeletionMap 写入模块的线程安全主要依赖于 `autil::ThreadMutex` 实现的**互斥锁**机制。

*   **保护目标**：锁的核心保护目标是 `DeletionMapSegmentWriter` 中的 `mDeletionMap` 指针所指向的位图数据。任何可能修改位图内容的操作（如 `Delete`）或需要读取一致性快照的操作（如 `CreateDumpItem`）都必须在锁的临界区内执行。
*   **并发模型**：
    *   **多线程写入**：多个上层应用的线程可以同时调用 `DeletionMapWriter::Delete`。这些调用最终会竞争同一个 `DeletionMapSegmentWriter` 实例的互斥锁。同一时刻，只有一个线程能够成功获取锁并修改位图，其他线程则会阻塞等待。虽然锁会带来一定的性能开销，但由于位操作本身极快，临界区非常小，因此在大多数场景下这种开销是可以接受的。
    *   **读写分离**：写入操作（`Delete`）和读取操作（`IsDeleted`）是天然分离的。读取核心（`DeletionMapReader`）持有的是对 `mDeletionMap` 的 `const` 引用，并且不访问写入器的锁。写入器在更新 `mDeletionMap` 时，CPU的原子写操作（对于对齐的字）保证了读取操作要么读到旧值，要么读到新值，不会读到中间状态。这种**无锁读**的设计，极大地提升了系统的并发查询性能。
    *   **写入与持久化的并发**：`CreateDumpItem` 通过克隆 `BitMap` 的方式，巧妙地将耗时的IO操作（文件写入）与高频的内存操作（`Delete`）解耦。一旦克隆完成，锁就可以被释放，`Delete` 操作可以继续进行，而 `DumpItem` 会在后台线程中安全地将数据快照写入磁盘。这避免了磁盘IO阻塞内存写入，是提升系统吞吐量的关键设计。

## 5. 总结与展望

DeletionMap 的写入与构建模块通过**分层设计**和**精细的并发控制**，实现了一个既简单易用又高效可靠的删除处理系统。

*   **`DeletionMapWriter`** 提供了清晰的顶层API，屏蔽了底层实现复杂性。
*   **`DeletionMapSegmentWriter`** 作为核心执行单元，通过**互斥锁**保证了并发写入的原子性和线程安全。
*   通过**克隆位图（Copy-on-Write）**的方式，实现了写入与持久化操作的解耦，避免了IO阻塞，提升了系统整体的并发能力。

这个模块的设计充分体现了在高性能索引系统中处理实时更新的常见挑战与解决方案：
*   **如何在提供简单接口的同时，管理复杂的底层状态？** -> 分层与抽象。
*   **如何在保证数据一致性的前提下，最大化并发性能？** -> 细粒度锁、读写分离、写时复制。

未来的优化可以考虑在极高写入吞吐量的场景下，探索**无锁数据结构（Lock-Free Data Structures）**来替代当前的互斥锁。例如，可以使用基于 `std::atomic` 的CAS（Compare-and-Swap）操作来更新位图，从而完全消除锁竞争，进一步提升写入性能。但这会显著增加实现的复杂性，需要在性能收益和维护成本之间做出权衡。
