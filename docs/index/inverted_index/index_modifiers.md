
# Indexlib 倒排索引修改器 (Modifier) 解析

**涉及文件:**
* `index/inverted_index/InvertedIndexModifier.h`
* `index/inverted_index/InplaceInvertedIndexModifier.h`
* `index/inverted_index/InplaceInvertedIndexModifier.cpp`
* `index/inverted_index/PatchInvertedIndexModifier.h`
* `index/inverted_index/PatchInvertedIndexModifier.cpp`

## 摘要

本文深入探讨了 Indexlib 中负责实现倒排索引实时更新的核心组件——索引修改器（Inverted Index Modifier）。我们首先分析了 `InvertedIndexModifier` 这一抽象基类，它定义了更新倒排索引的基本接口。接着，我们详细研究了它的两个关键实现：`InplaceInvertedIndexModifier` 和 `PatchInvertedIndexModifier`。`InplaceInvertedIndexModifier` 主要用于处理内存中索引的“原地”更新，而 `PatchInvertedIndexModifier` 则通过生成补丁文件（patch file）的方式来支持对磁盘上存量索引的修改。通过对这些修改器的分析，我们可以更好地理解 Indexlib 是如何实现其强大的实时索引能力的。

## 1. 引言

在实时搜索场景中，对索引的增、删、改操作要求极高的时效性。传统的批量建库方式无法满足这种需求。因此，现代搜索引擎普遍采用增量索引和实时更新技术。Indexlib 作为一款高性能的索引库，其核心能力之一就是支持对倒排索引的实时修改。

实现这一功能的核心组件就是索引修改器（Inverted Index Modifier）。它封装了对不同状态（内存中、磁盘上）的索引进行更新的复杂逻辑，为上层应用提供了简洁、统一的接口。本文将深入剖析 Indexlib 中索引修改器的设计与实现，帮助读者理解其工作原理。

## 2. `InvertedIndexModifier`：修改器基类

`InvertedIndexModifier.h` 中定义的 `InvertedIndexModifier` 是一个抽象基类，它为所有倒排索引修改器定义了统一的接口。

### 2.1. 核心方法

*   `UpdateOneFieldTokens(docid_t docId, const document::ModifiedTokens& modifiedTokens, bool isForReplay)`: 这是修改器的核心纯虚方法。它负责处理对单个文档、单个字段的更新操作。
    *   `docId`: 要更新的文档 ID。
    *   `modifiedTokens`: 一个 `ModifiedTokens` 对象，封装了该字段下需要被添加或删除的词元（token）。
    *   `isForReplay`: 一个布尔标志，用于区分是正常的构建流程还是 OpLog 回放流程。

### 2.2. 设计理念

`InvertedIndexModifier` 的设计体现了**策略模式**。它定义了一个统一的更新接口，而将具体的更新策略（是原地修改还是生成补丁）延迟到子类中去实现。这种设计使得 Indexlib 可以灵活地支持不同的更新方式，以适应不同的应用场景和性能要求。

## 3. `InplaceInvertedIndexModifier`：原地更新

`InplaceInvertedIndexModifier.h` 和 `InplaceInvertedIndexModifier.cpp` 中定义的 `InplaceInvertedIndexModifier` 主要负责对**内存中**的索引进行“原地”修改。这通常发生在实时构建（real-time build）的过程中。

### 3.1. 设计动机

对于正在构建中的、位于内存里的索引段（building segment），最高效的更新方式就是直接在内存中修改其数据结构（例如，更新 B-树或跳表）。`InplaceInvertedIndexModifier` 正是为了实现这种高效的“原地”更新而设计的。

### 3.2. 核心机制

`InplaceInvertedIndexModifier` 在 `Init` 阶段会遍历 `TabletData` 中的所有 segment，并根据 segment 的状态（`ST_BUILT`, `ST_DUMPING`, `ST_BUILDING`）获取相应的 `IInvertedDiskIndexer` 或 `IInvertedMemIndexer`，并将它们存储在 `_buildInfoHolders` 中。`_buildInfoHolders` 是一个 `std::map`，其 key 是 `indexid_t`，value 是一个 `SingleInvertedIndexBuildInfoHolder` 结构体，该结构体中分别保存了不同状态下的 indexer 指针。

当 `Update` 方法被调用时，它会根据 `docId` 找到对应的 indexer，并调用 `InvertedIndexerOrganizerUtil` 中的静态方法来执行实际的更新操作。`InvertedIndexerOrganizerUtil` 会根据 indexer 的类型（disk indexer 或 mem indexer）来调用其相应的更新接口。

### 3.3. 关键实现细节

*   **`_buildInfoHolders`**: 缓存了所有可修改的 indexer 的指针，按 `indexid_t` 进行组织。
*   **`Init`**: 核心初始化逻辑，负责从 `TabletData` 中收集和组织 indexer。
*   **`Update`**: 遍历 `IDocumentBatch`，对每个文档调用 `UpdateOneFieldTokens`。
*   **`UpdateOneFieldTokens`**: 根据 `fieldId` 找到对应的 `indexId`，然后从 `_buildInfoHolders` 中获取相应的 `buildInfoHolder`，并调用 `InvertedIndexerOrganizerUtil` 来执行更新。

### 3.4. 代码示例

```cpp
// indexlib/index/inverted_index/InplaceInvertedIndexModifier.h

class InplaceInvertedIndexModifier final : public InvertedIndexModifier
{
    // ...
public:
    Status Init(const indexlibv2::framework::TabletData& tabletData);
    Status Update(indexlibv2::document::IDocumentBatch* docBatch);
    Status UpdateOneFieldTokens(docid_t docId, const document::ModifiedTokens& modifiedTokens,
                                bool isForReplay) override;
    // ...
private:
    IndexerOrganizerMeta _indexerOrganizerMeta;
    std::map<segmentid_t, std::pair<docid_t, uint64_t>> _segmentId2BaseDocIdAndDocCountPair;
    std::map<indexid_t, SingleInvertedIndexBuildInfoHolder> _buildInfoHolders;
    // ...
};
```

## 4. `PatchInvertedIndexModifier`：补丁更新

`PatchInvertedIndexModifier.h` 和 `PatchInvertedIndexModifier.cpp` 中定义的 `PatchInvertedIndexModifier` 用于对**磁盘上**的存量索引进行修改。它通过生成补丁文件（patch file）的方式来实现这一功能。

### 4.1. 设计动机

直接修改磁盘上的倒排索引文件是一个非常复杂且低效的操作。为了避免这种情况，Indexlib 采用了生成补丁文件的方式。当需要更新一个已落盘的 segment 时，`PatchInvertedIndexModifier` 会将更新操作（哪个文档的哪个词项被添加或删除）记录到一个独立的补丁文件中。在查询时，Indexlib 会加载这些补丁文件，并在内存中将它们与原始索引进行合并，从而提供一个统一的、最新的索引视图。

### 4.2. 核心机制

`PatchInvertedIndexModifier` 内部维护了一个 `std::vector<std::unique_ptr<InvertedIndexPatchWriter>>`。每个 `InvertedIndexPatchWriter` 都对应一个可更新的索引。

在 `Init` 阶段，它会为每个可更新的索引创建一个 `InvertedIndexPatchWriter`。当 `UpdateOneFieldTokens` 被调用时，它会根据 `docId` 找到其所属的 segment，然后将更新操作（`docId`、`termKey`、操作类型）写入到相应的 `InvertedIndexPatchWriter` 中。

### 4.3. 关键实现细节

*   **`_patchWriters`**: 一个 `std::vector`，存储了所有 `InvertedIndexPatchWriter` 的唯一指针。
*   **`_fieldId2PatchWriters`**: 一个 `std::map`，用于快速地根据 `fieldId` 找到需要写入的 `InvertedIndexPatchWriter`。
*   **`Init`**: 创建 `InvertedIndexPatchWriter` 实例。
*   **`UpdateOneFieldTokens`**: 将更新操作写入补丁文件。
*   **`Close`**: 关闭所有的 `InvertedIndexPatchWriter`，确保补丁数据被完整地写入磁盘。

### 4.4. 代码示例

```cpp
// indexlib/index/inverted_index/PatchInvertedIndexModifier.h

class PatchInvertedIndexModifier final : public InvertedIndexModifier
{
public:
    PatchInvertedIndexModifier(const std::shared_ptr<indexlibv2::config::ITabletSchema>& schema,
                               const std::shared_ptr<indexlib::file_system::IDirectory>& workDir);
    // ...
public:
    Status Init(const indexlibv2::framework::TabletData& tabletData);
    Status UpdateOneFieldTokens(docid_t docId, const document::ModifiedTokens& modifiedTokens, bool) override;
    Status Close();

private:
    std::shared_ptr<indexlib::file_system::IDirectory> _workDir;
    std::map<fieldid_t, std::vector<InvertedIndexPatchWriter*>> _fieldId2PatchWriters;
    std::vector<std::unique_ptr<InvertedIndexPatchWriter>> _patchWriters;
    // ...
};
```

## 5. 技术挑战与权衡

*   **更新性能**: `InplaceInvertedIndexModifier` 的性能通常优于 `PatchInvertedIndexModifier`，因为它直接在内存中操作。但是，`InplaceInvertedIndexModifier` 只适用于内存中的索引。
*   **补丁文件管理**: `PatchInvertedIndexModifier` 会产生大量的补丁文件。如何高效地管理和合并这些补丁文件，是 Indexlib 需要解决的一个重要问题。
*   **一致性与可见性**: 在多线程环境下，需要确保更新操作的原子性和一致性，并控制更新结果何时对查询可见。这涉及到复杂的并发控制和内存屏障机制。

## 6. 结论

`InvertedIndexModifier` 及其子类是 Indexlib 实现实时索引功能的核心。通过 `InplaceInvertedIndexModifier` 和 `PatchInvertedIndexModifier` 的协同工作，Indexlib 能够高效地处理对不同状态下索引的更新请求。深入理解这些修改器的设计与实现，对于我们掌握 Indexlib 的实时索引原理，以及进行相关的二次开发和性能优化，都具有至关重要的意义。
