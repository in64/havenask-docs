
# Havenask Normal Tablet 数据读取与查询模块深度解析

**涉及文件:**
*   `table/normal_table/NormalTabletReader.cpp`
*   `table/normal_table/NormalTabletReader.h`
*   `table/normal_table/NormalTabletSessionReader.cpp`
*   `table/normal_table/NormalTabletSessionReader.h`
*   `table/normal_table/NormalTabletSearcher.cpp`
*   `table/normal_table/NormalTabletSearcher.h`
*   `table/normal_table/NormalTabletDocIterator.cpp`
*   `table/normal_table/NormalTabletDocIterator.h`

## 1. 概述

Havenask `Normal Table` 的数据读取与查询模块是整个检索引擎对外提供服务的核心窗口。它负责解析用户的查询请求，高效地从各种索引（倒排、正排、主键、Source、Summary等）中检索、聚合和获取数据，并最终以结构化的形式返回给用户。该模块的设计直接决定了检索引擎的查询性能、功能丰富度和易用性。

本文将深入剖析该模块的四个关键组件：`NormalTabletReader`、`NormalTabletSessionReader`、`NormalTabletSearcher` 和 `NormalTabletDocIterator`，揭示它们如何协同工作，将底层的索引数据结构封装成强大而灵活的查询服务。

## 2. 核心组件协同工作流程

读取与查询模块的组件关系紧密，形成了一个层次分明、职责清晰的架构。

```mermaid
graph TD
    subgraph "用户/外部调用"
        A[外部查询请求 (JSON/Protobuf)]
        B[数据导出请求 (Export)]
    end

    A --> C{NormalTabletSessionReader};
    B --> E(NormalTabletDocIterator);

    subgraph "查询核心层"
        C -- 委托查询 --> D(NormalTabletReader);
        D -- 创建Searcher --> F(NormalTabletSearcher);
        F -- 执行查询 --> D;
    end

    subgraph "数据访问层"
        D -- 访问索引 --> G[各种IndexReader<br/>(MultiFieldIndexReader, AttributeReader, etc.)];
        E -- 访问索引 --> G;
    end

    subgraph "会话与资源管理"
        C -- 管理内存 --> H(IIndexMemoryReclaimer);
    end

    style C fill:#f9f,stroke:#333,stroke-width:2px
    style D fill:#ccf,stroke:#333,stroke-width:2px
    style F fill:#cfc,stroke:#333,stroke-width:2px
    style E fill:#fcf,stroke:#333,stroke-width:2px
```

- **NormalTabletReader**: 是最核心的读取器，它持有并管理所有底层 `IndexReader`（如倒排、正排、主键等）的实例。它提供了对索引数据的原子性、一致性的视图，是所有上层查询功能的基石。
- **NormalTabletSessionReader**: 是 `NormalTabletReader` 的一层封装，主要面向在线查询场景。它引入了会话（Session）和内存回收机制（`IIndexMemoryReclaimer`），确保在并发查询环境下，即使底层 `TabletReader` 发生 `Reopen` 切换，当前会话也能继续使用旧的 `Reader` 完成查询，保证了查询的稳定性和一致性。它是典型的“写时复制”（Copy-On-Write）思想的体现。
- **NormalTabletSearcher**: 作为一个辅助类，它封装了具体的查询逻辑。当 `NormalTabletReader` 接收到查询请求时，会创建一个临时的 `NormalTabletSearcher` 实例来处理这个请求。`Searcher` 负责解析查询参数、执行倒排检索、获取正排/Summary/Source 数据，并将结果序列化。
- **NormalTabletDocIterator**: 专用于数据导出场景。它提供了一个迭代器接口，可以按照指定的范围（`rangeInRatio`）或条件（`params`）高效地遍历全量或部分文档，并以 `RawDocument` 的形式返回，非常适合批量数据处理和离线分析任务。

## 3. `NormalTabletReader`：索引数据的统一访问入口

`NormalTabletReader` 是整个查询体系的基石，它聚合了所有类型的 `IndexReader`，提供了一个统一的、面向整个 `Tablet` 的数据视图。

### 3.1. 主要职责

- **初始化和持有 IndexReaders**: 在 `DoOpen` 方法中，根据 `Schema` 配置，通过 `IndexFactoryCreator` 创建并初始化所有需要的 `IndexReader`，如 `MultiFieldIndexReader`、`AttributeReader`、`SummaryReader`、`PrimaryKeyIndexReader` 等，并将它们存储在 `_indexReaderMap` 中。
- **提供访问接口**: 提供了 `GetXXXReader` 系列方法，供上层（主要是 `NormalTabletSearcher`）获取特定类型的 `IndexReader` 实例。
- **管理 Tablet 元信息**: 初始化并持有 `NormalTabletInfoHolder`，它包含了 `Tablet` 的元信息，如 `docid` 范围、排序信息等，这些信息对于查询优化（如排序优化）至关重要。
- **查询逻辑委托**: 当接收到 `Search` 请求时，它会创建一个 `NormalTabletSearcher` 实例，并将查询任务委托给它。这种设计将数据持有和查询逻辑进行了解耦。

### 3.2. 核心实现分析

#### IndexReader 的初始化

`DoOpen` 方法是 `NormalTabletReader` 的生命周期起点，其核心任务就是构建完整的 `IndexReader` 体系。

```cpp
// table/normal_table/NormalTabletReader.cpp

Status NormalTabletReader::DoOpen(const std::shared_ptr<framework::TabletData>& tabletData,
                                  const framework::ReadResource& readResource)
{
    // ...
    auto indexFactoryCreator = index::IndexFactoryCreator::GetInstance();
    auto indexConfigs = _schema->GetIndexConfigs();
    for (const auto& indexConfig : indexConfigs) {
        const auto& indexType = indexConfig->GetIndexType();
        const auto& indexName = indexConfig->GetIndexName();
        auto [status, indexFactory] = indexFactoryCreator->Create(indexType);
        // ... 错误处理 ...
        auto indexReader = indexFactory->CreateIndexReader(indexConfig, indexReaderParam);
        // ... 错误处理 ...
        status = indexReader->Open(indexConfig, tabletData.get());
        // ... 错误处理 ...
        _indexReaderMap[std::pair(indexType, indexName)] = std::move(indexReader);
    }

    // 初始化聚合型的 Reader
    auto status = InitInvertedIndexReaders();
    RETURN_IF_STATUS_ERROR(status, "init multi field index reader failed.");
    status = InitIndexAccessoryReader(tabletData.get(), readResource);
    RETURN_IF_STATUS_ERROR(status, "init index accessory reader failed.");
    _multiFieldIndexReader->SetAccessoryReader(_indexAccessoryReader);

    // 初始化 NormalTabletInfo
    RETURN_IF_STATUS_ERROR(InitNormalTabletInfoHolder(tabletData, readResource), "init tablet info failed");

    // 初始化 SummaryReader (依赖其他 Reader)
    status = InitSummaryReader();
    RETURN_IF_STATUS_ERROR(status, "init summary reader failed.");
    return status;
}
```

这段代码展示了一个典型的工厂模式和依赖注入的应用。`Reader` 本身不关心 `IndexReader` 的具体实现，而是通过工厂创建。同时，像 `SummaryReader` 这样依赖其他 `Reader` 的组件，会在所有基础 `Reader` 初始化完成后再进行初始化，保证了依赖关系的正确性。

#### 查询的委托

`Search` 方法体现了职责的分离。

```cpp
// table/normal_table/NormalTabletReader.cpp

Status NormalTabletReader::Search(const std::string& jsonQuery, std::string& result) const
{
    return NormalTabletSearcher(this).Search(jsonQuery, result);
}
```

这种设计非常简洁，`Reader` 只负责持有数据和状态，而将无状态的、一次性的查询逻辑完全交给 `Searcher`。这使得 `Reader` 的状态更加稳定，也让 `Searcher` 的实现可以更专注于查询逻辑本身，易于测试和维护。

### 3.3. 设计动机与技术风险

- **设计动机**:
    - **统一视图**: 为上层提供一个统一的、包含所有索引数据的数据视图，屏蔽底层 `Segment` 和各种 `IndexReader` 的复杂性。
    - **原子性**: 一个 `NormalTabletReader` 实例对应一个特定版本的 `TabletData`，提供了该版本数据的原子性快照。
    - **解耦**: 将数据持有（`Reader`）与查询逻辑（`Searcher`）分离，使得代码结构更清晰，职责更单一。

- **技术风险**:
    - **初始化开销**: `DoOpen` 过程需要初始化所有的 `IndexReader`，对于一个包含大量索引的 `Schema`，这个过程可能会比较耗时，影响 `Reopen` 的速度。
    - **内存占用**: `Reader` 会持有所有 `IndexReader` 的实例，这些实例本身会占用一定的内存。在多版本并存的场景下（由 `SessionReader` 管理），内存占用需要被精确控制。

## 4. `NormalTabletSessionReader`：并发查询的稳定器

`NormalTabletSessionReader` 是为了解决在线服务中一个核心痛点而设计的：如何在后台数据持续更新、`TabletReader` 频繁 `Reopen` 的情况下，保证正在进行的查询能够稳定、一致地完成？

### 4.1. 主要职责

- **生命周期管理**: 它的核心是内部持有一个 `std::shared_ptr<NormalTabletReader>`。当外部获取一个 `SessionReader` 实例时，实际上是获取了这个 `shared_ptr` 的一个拷贝，从而延长了其指向的 `NormalTabletReader` 的生命周期。
- **查询稳定性**: 即使后端的 `Tablet` 完成了 `Reopen`，生成了新的 `NormalTabletReader`，旧的查询会话仍然持有旧的 `Reader` 的 `shared_ptr`，因此可以继续在旧的数据快照上完成查询，不受 `Reopen` 的影响。
- **内存回收**: 通过 `IIndexMemoryReclaimer`（通常是 `EpochBasedMemReclaimer`），它能确保当一个 `NormalTabletReader` 不再被任何查询会话引用时，其占用的资源能够被安全、及时地回收。
- **接口代理**: 它将所有查询相关的接口（如 `Search`, `GetAttributeReader` 等）直接代理（delegate）到底层的 `_impl`（即 `NormalTabletReader` 实例）上。

### 4.2. 核心实现分析

`NormalTabletSessionReader` 的实现非常简洁，其本质是一个代理模式和 `std::shared_ptr` 引用计数的巧妙运用。

```cpp
// table/common/CommonTabletSessionReader.h (模板基类)

template <typename TabletReaderType>
class CommonTabletSessionReader : public framework::ITabletReader
{
public:
    // ...
    // 构造函数接收一个 TabletReader 的 shared_ptr
    CommonTabletSessionReader(const std::shared_ptr<TabletReaderType>& tabletReader,
                              const std::shared_ptr<framework::IIndexMemoryReclaimer>& memReclaimer)
        : _impl(tabletReader)
        , _memReclaimer(memReclaimer)
    {
        // ...
    }

    // 析构时，通过 Reclaimer 通知资源可以被回收
    ~CommonTabletSessionReader() override
    {
        if (_memReclaimer) {
            _memReclaimer->Leave();
        }
    }

protected:
    std::shared_ptr<TabletReaderType> _impl; // 核心：持有 Reader 的 shared_ptr
    std::shared_ptr<framework::IIndexMemoryReclaimer> _memReclaimer;
};

// table/normal_table/NormalTabletSessionReader.cpp
// 将查询接口直接代理给 _impl
Status NormalTabletSessionReader::QueryIndex(const base::PartitionQuery& query,
                                             base::PartitionResponse& partitionResponse) const
{
    return _impl->QueryIndex(query, partitionResponse);
}
```

这种设计的优雅之处在于，它用非常少的代码，就解决了一个非常棘手的并发控制问题。它没有使用任何重量级的锁，而是利用 C++ 语言本身的特性（RAII 和 `shared_ptr`）来保证线程安全和资源管理。

### 4.3. 设计动机与技术风险

- **设计动机**:
    - **无锁查询**: 避免在查询路径上使用锁，最大化查询性能。
    - **查询隔离**: 保证单个查询的执行过程不会被后台的 `Reopen` 操作干扰，提供一致性的数据视图。
    - **优雅的资源管理**: 结合 `Epoch` 机制，实现了对多版本 `Reader` 资源的自动、安全回收。

- **技术风险**:
    - **长查询导致的资源占用**: 如果某个查询耗时非常长，它会一直持有旧版本 `Reader` 的引用，导致旧版本的资源（内存、文件句柄）无法被及时释放。在极端情况下，如果长查询持续存在，可能会导致资源耗尽。
    - **内存回收延迟**: `Epoch` 机制本身存在一定的延迟，资源不会在引用计数归零后立即被回收，这在内存极度紧张的情况下可能是一个问题。

## 5. `NormalTabletSearcher` & `NormalTabletDocIterator`：查询逻辑的执行者

这两个类是具体查询和导出逻辑的实现者。

### 5.1. `NormalTabletSearcher`：在线查询的处理器

`Searcher` 是一个无状态的工具类，它的生命周期仅限于一次查询。

- **职责**:
    - **解析查询**: 将 JSON 或 Protobuf 格式的查询请求解析成内部可处理的数据结构。
    - **执行检索**: 根据查询条件（`condition`），调用 `MultiFieldIndexReader` 查找倒排拉链，获取 `docid` 列表。
    - **主键/DocId查询**: 处理通过主键（`pk`）或 `docid` 直接查询的请求。
    - **数据获取**: 遍历检索到的 `docid` 列表，按需从 `AttributeReader`、`SummaryReader`、`SourceReader` 中获取字段值。
    - **结果封装**: 将获取到的数据封装到 `PartitionResponse` 这个 Protobuf 结构中，并可序列化为 JSON 返回。

- **核心实现**:
  `QueryIndexByDocId` 方法展示了如何根据 `docid` 获取各种数据。

  ```cpp
  // table/normal_table/NormalTabletSearcher.cpp
  Status NormalTabletSearcher::QueryIndexByDocId(const std::vector<std::string>& attrs,
                                                 const std::vector<std::string>& summarys,
                                                 const std::vector<std::string>& sources,
                                                 const std::vector<docid_t>& docids, const int64_t limit,
                                                 base::PartitionResponse& partitionResponse) const
  {
      for (int docCount = 0; docCount < std::min(docids.size(), size_t(limit)); ++docCount) {
          auto row = partitionResponse.add_rows();
          auto docid = docids[docCount];
          row->set_docid(docid);
          // 分别从 Attribute, Summary, Source 中获取数据
          RETURN_STATUS_DIRECTLY_IF_ERROR(QueryRowAttrByDocId(docid, attrs, *row));
          RETURN_STATUS_DIRECTLY_IF_ERROR(QueryRowSummaryByDocId(docid, summarys, *row));
          RETURN_STATUS_DIRECTLY_IF_ERROR(QueryRowSourceByDocId(docid, sources, *row));
      }
      // ... 填充元信息 ...
      return Status::OK();
  }
  ```

### 5.2. `NormalTabletDocIterator`：数据导出的迭代器

`DocIterator` 专为批量导出场景设计，提供了高效的全量或增量数据遍历能力。

- **职责**:
    - **初始化范围**: 根据 `rangeInRatio`（百分比范围）或 `__buildin_docid_range__`（绝对 `docid` 范围）参数确定要遍历的 `docid` 区间。
    - **初始化条件**: 可以接受 `user_define_index_param` 参数，构建一个 `PostingExecutor` 来按条件过滤文档，而不仅仅是遍历。
    - **迭代访问**: `Next` 方法负责获取下一个符合条件的文档，并从 `Source`、`Attribute`、`Summary` 中按需读取字段，填充到 `RawDocument` 中。
    - **状态管理**: 内部维护 `_currentDocId`，并通过 `checkpoint` 机制支持断点续传。

- **核心实现**:
  `Next` 方法是其核心逻辑，体现了数据获取的优先级：Source > Attribute > Summary。

  ```cpp
  // table/normal_table/NormalTabletDocIterator.cpp
  Status NormalTabletDocIterator::Next(indexlibv2::document::RawDocument* rawDocument, std::string* checkpoint,
                                       framework::Locator::DocInfo* docInfo)
  {
      // ...
      // 1. 优先从 Source 读取
      if (_sourceReader) {
          // ... 从 Source 获取数据并填充到 rawDocument ...
      }

      // 2. 如果字段不在 Source 中，尝试从 Attribute 读取
      for (auto& [fieldName, attrFieldIter] : _attrFieldIters) {
          if (rawDocument->exist(fieldName)) {
              continue;
          }
          // ... 从 Attribute 获取数据 ...
      }

      // 3. 最后尝试从 Summary 读取
      if (_summaryReader) {
          // ... 从 Summary 获取数据 ...
      }

      // 移动到下一个 docid
      auto status = SeekByDocId(_currentDocId + 1);
      // ... 更新 checkpoint ...
      return Status::OK();
  }
  ```

### 5.3. 设计动机与技术风险

- **设计动机**:
    - **场景分离**: 将在线的、低延迟的随机查询（`Searcher`）和离线的、高吞吐的批量导出（`DocIterator`）分离开来，使得两者可以针对各自的场景进行独立优化。
    - **逻辑封装**: `Searcher` 封装了复杂的查询解析和执行逻辑，使得 `Reader` 的接口非常简洁。
    - **高效遍历**: `DocIterator` 提供了灵活的范围和条件控制，以及高效的顺序访问模式，非常适合 MapReduce 等批处理框架。

- **技术风险**:
    - **查询语法解析**: `Searcher` 中对 JSON 查询的解析逻辑如果不够健壮，可能会有安全风险（如注入攻击）或稳定性问题。使用 Protobuf 是更安全的选择。
    - **迭代器性能**: `DocIterator` 的性能直接影响导出任务的效率。其内部实现的 `SeekByDocId` 如果不够高效（例如，在有大量删除的情况下），可能会成为瓶颈。

## 6. 总结

Havenask `Normal Table` 的数据读取与查询模块是一个层次清晰、职责分明、设计优雅的系统。它通过 `Reader`、`SessionReader`、`Searcher` 和 `DocIterator` 的精妙组合，成功地应对了高性能、高并发、高可用的挑战。

- **`NormalTabletReader`** 作为数据访问的基石，提供了统一、原子的数据视图。
- **`NormalTabletSessionReader`** 通过巧妙的引用计数机制，解决了并发查询与数据更新之间的核心矛盾，保证了服务的稳定性。
- **`NormalTabletSearcher`** 和 **`NormalTabletDocIterator`** 则分别针对在线查询和离线导出这两个典型场景，提供了高效、灵活的逻辑执行层。

对该模块的深入理解，不仅能帮助我们更好地进行性能调优和问题排查，也为我们设计自己的查询引擎或数据服务提供了优秀的架构范例。
