
# Indexlib Summary 模块磁盘I/O与合并机制深度解析

## 1. 概述

**涉及文件:**

*   `indexlib/index/normal/summary/local_disk_summary_writer.cpp`
*   `indexlib/index/normal/summary/local_disk_summary_writer.h`
*   `indexlib/index/normal/summary/summary_writer_impl.cpp`
*   `indexlib/index/normal/summary/summary_writer_impl.h`
*   `indexlib/index/normal/summary/local_disk_summary_segment_reader.cpp`
*   `indexlib/index/normal/summary/local_disk_summary_segment_reader.h`
*   `indexlib/index/normal/summary/local_disk_summary_reader.cpp`
*   `indexlib/index/normal/summary/local_disk_summary_reader.h`
*   `indexlib/index/normal/summary/summary_merger.h`
*   `indexlib/index/normal/summary/local_disk_summary_merger.cpp`
*   `indexlib/index/normal/summary/local_disk_summary_merger.h`

本文档聚焦于 Indexlib Summary 模块与磁盘交互的实现细节，涵盖了从数据写入、持久化、段内读取、跨段查询到最终合并的全过程。这部分是 Summary 功能的核心，其实现效率直接决定了索引的构建速度和查询性能。我们将深入分析其文件结构、I/O 模型、异步化改造以及多组 Summary 的处理逻辑，揭示其在高性能存储和检索方面的设计精髓。

## 2. 数据写入与持久化：从内存到磁盘

数据写入是索引构建的起点。`SummaryWriter` 接口的实现类负责将文档的 Summary 数据流式写入内存，并在适当时机（`Dump`）将其持久化到磁盘，形成一个独立的段（Segment）。

### 2.1. `LocalDiskSummaryWriter`：单组 Summary 的写入核心

`LocalDiskSummaryWriter` 是针对**单个 Summary 组**的具体写入实现。它并不直接实现 `SummaryWriter` 接口，而是作为一个功能单元，被上层的 `SummaryWriterImpl` 所调用。

其核心依赖于 `GroupFieldDataWriter`，这是一个更通用的组件，专门用于处理按组存储的变长字段（如 Summary、部分 Attribute）。

**核心逻辑与实现：**

```cpp
// indexlib/index/normal/summary/local_disk_summary_writer.h
class LocalDiskSummaryWriter
{
public:
    // ...
    void Init(const config::SummaryGroupConfigPtr& summaryGroupConfig,
              util::BuildResourceMetrics* buildResourceMetrics);
    void AddDocument(const autil::StringView& data);
    void Dump(const file_system::DirectoryPtr& directory, autil::mem_pool::PoolBase* dumpPool,
              const std::string& temperatureLayer);
    // ...
private:
    config::SummaryGroupConfigPtr mSummaryGroupConfig;
    GroupFieldDataWriterPtr mDataWriter;
    // ...
};

// indexlib/index/normal/summary/local_disk_summary_writer.cpp
void LocalDiskSummaryWriter::Init(const SummaryGroupConfigPtr& summaryGroupConfig,
                                  BuildResourceMetrics* buildResourceMetrics)
{
    mSummaryGroupConfig = summaryGroupConfig;
    mDataWriter.reset(
        new GroupFieldDataWriter(mSummaryGroupConfig->GetSummaryGroupDataParam().GetFileCompressConfig()));
    mDataWriter->Init(SUMMARY_DATA_FILE_NAME, SUMMARY_OFFSET_FILE_NAME,
                      VarLenDataParamHelper::MakeParamForSummary(mSummaryGroupConfig), buildResourceMetrics);
}

void LocalDiskSummaryWriter::AddDocument(const StringView& data) 
{
    mDataWriter->AddDocument(data); 
}

void LocalDiskSummaryWriter::Dump(const file_system::DirectoryPtr& directory, autil::mem_pool::PoolBase* dumpPool,
                                  const std::string& temperatureLayer)
{
    mDataWriter->Dump(directory, dumpPool, temperatureLayer);
}
```

1.  **初始化 (`Init`)**: 在初始化阶段，它会创建一个 `GroupFieldDataWriter` 实例。关键在于 `Init` 调用中传入了两个核心文件名：`SUMMARY_DATA_FILE_NAME` ("summary.data") 和 `SUMMARY_OFFSET_FILE_NAME` ("summary.offset")。这明确了 Summary 的磁盘存储结构。
2.  **添加文档 (`AddDocument`)**: 此方法极其轻量，直接将序列化好的二进制数据（`StringView`）透传给 `mDataWriter`。`mDataWriter` 内部的 `VarLenDataAccessor` 会将这些数据追加到内存缓冲区中。
3.  **持久化 (`Dump`)**: `Dump` 操作同样委托给 `mDataWriter`。`mDataWriter` 会执行两个关键步骤：
    *   将内存中所有文档的变长数据（data）连续写入到 `summary.data` 文件。
    *   为每个文档生成一个偏移量（offset），指向其在 `summary.data` 文件中的起始位置，并将这些偏移量紧凑地写入到 `summary.offset` 文件中。

**磁盘文件结构：**

*   **`summary.data`**: 存储所有文档的、经过序列化和（可选的）压缩后的二进制 Summary 数据。数据是紧凑排列的。
*   **`summary.offset`**: 这是一个定长数据文件，可以看作一个 `uint64_t` 数组。数组的下标是 `localDocId`，其存储的值是该 `docid` 对应的数据在 `summary.data` 文件中的起始偏移量。通过 `offset[i+1] - offset[i]` 即可得到第 `i` 个文档的数据长度。

这种**数据与索引（偏移量）分离**的存储方式是典型的变长数据存储优化，它使得我们可以通过一次 O(1) 的偏移量查找和一次磁盘 seek 就能定位任意文档的数据。

### 2.2. `SummaryWriterImpl`：多组 Summary 的管理者

`SummaryWriterImpl` 是 `SummaryWriter` 接口的真正实现者。它负责管理一个 Schema 中定义的所有 Summary 组，并为每个需要存储的组创建一个 `LocalDiskSummaryWriter`。

**核心逻辑与实现：**

```cpp
// indexlib/index/normal/summary/summary_writer_impl.h
class SummaryWriterImpl : public SummaryWriter
{
    // ...
private:
    typedef std::vector<LocalDiskSummaryWriterPtr> GroupWriterVec;
    GroupWriterVec mGroupWriterVec;
    config::SummarySchemaPtr mSummarySchema;
    // ...
};

// indexlib/index/normal/summary/summary_writer_impl.cpp
void SummaryWriterImpl::Init(const SummarySchemaPtr& summarySchema, BuildResourceMetrics* buildResourceMetrics)
{
    // ...
    for (summarygroupid_t groupId = 0; groupId < summarySchema->GetSummaryGroupConfigCount(); ++groupId) {
        const SummaryGroupConfigPtr& groupConfig = mSummarySchema->GetSummaryGroupConfig(groupId);
        if (groupConfig->NeedStoreSummary()) {
            mGroupWriterVec.push_back(LocalDiskSummaryWriterPtr(new LocalDiskSummaryWriter()));
            mGroupWriterVec[groupId]->Init(groupConfig, buildResourceMetrics);
        } else {
            mGroupWriterVec.push_back(LocalDiskSummaryWriterPtr()); // push null ptr
        }
    }
}

void SummaryWriterImpl::AddDocument(const SerializedSummaryDocumentPtr& document)
{
    // ...
    const char* cursor = document->GetValue();
    for (size_t i = 0; i < mGroupWriterVec.size(); ++i) {
        if (!mGroupWriterVec[i]) continue;
        uint32_t len = GroupFieldFormatter::ReadVUInt32(cursor);
        StringView data(cursor, len);
        cursor += len;
        mGroupWriterVec[i]->AddDocument(data);
    }
    // ...
}

void SummaryWriterImpl::Dump(const DirectoryPtr& directory, PoolBase* dumpPool)
{
    // ...
    if (mGroupWriterVec[0]) { // default group
        mGroupWriterVec[0]->Dump(directory, dumpPool, mTemperatureLayer);
    }

    for (size_t i = 1; i < mGroupWriterVec.size(); ++i) {
        if (!mGroupWriterVec[i]) continue;
        const string& groupName = mGroupWriterVec[i]->GetGroupName();
        DirectoryPtr realDirectory = directory->MakeDirectory(groupName);
        mGroupWriterVec[i]->Dump(realDirectory, dumpPool, mTemperatureLayer);
    }
}
```

1.  **初始化 (`Init`)**: 遍历 `SummarySchema` 中的所有 `SummaryGroupConfig`，为每个需要存储的组（`NeedStoreSummary()` 为 true）创建一个 `LocalDiskSummaryWriter` 实例并初始化。
2.  **添加文档 (`AddDocument`)**: `SerializedSummaryDocument` 中包含的是一个**打包后**的数据块。这个数据块的格式是 `[len1][group_data1][len2][group_data2]...`。`AddDocument` 的逻辑就是按顺序解析这个数据块，将属于每个组的数据（`group_data`）分发给对应的 `LocalDiskSummaryWriter`。
3.  **持久化 (`Dump`)**: `Dump` 操作会为不同的 Summary 组创建不同的目录。默认组的数据直接存储在 `directory` 下，而其他命名组的数据则存储在 `directory` 下以组名命名的子目录中。这保证了不同组的数据在物理上是隔离的。

**设计动机**：这种分层设计（`SummaryWriterImpl` -> `LocalDiskSummaryWriter` -> `GroupFieldDataWriter`）非常清晰。顶层负责管理和分发，中层负责单组的逻辑，底层负责具体的读写文件和内存管理。这使得代码易于维护和扩展。

## 3. 磁盘数据读取：从 Segment 到 Partition

数据读取与写入过程相对应，也分为段内读取和跨段读取两个层次。

### 3.1. `LocalDiskSummarySegmentReader`：段内数据的直接访问者

`LocalDiskSummarySegmentReader` 实现了 `SummarySegmentReader` 接口，负责从磁盘上一个已经持久化好的 Segment 中读取指定 `localDocId` 的 Summary 数据。

**核心逻辑与实现：**

```cpp
// indexlib/index/normal/summary/local_disk_summary_segment_reader.h
class LocalDiskSummarySegmentReader : public SummarySegmentReader
{
    // ...
protected:
    VarLenDataReaderPtr mDataReader;
    config::SummaryGroupConfigPtr mSummaryGroupConfig;
    // ...
};

// indexlib/index/normal/summary/local_disk_summary_segment_reader.cpp
bool LocalDiskSummarySegmentReader::Open(const index_base::SegmentData& segmentData)
{
    // ... get directory by group name ...
    mDataReader->Init(segmentData.GetSegmentInfo()->docCount, directory, 
                      SUMMARY_OFFSET_FILE_NAME, SUMMARY_DATA_FILE_NAME);
    return true;
}

bool LocalDiskSummarySegmentReader::GetDocument(docid_t localDocId, SearchSummaryDocument* summaryDoc) const
{
    autil::StringView data;
    if (!mDataReader->GetValue(localDocId, data, summaryDoc->getPool())) {
        return false;
    }
    // ...
    SummaryGroupFormatter formatter(mSummaryGroupConfig);
    return formatter.DeserializeSummary(summaryDoc, (char*)data.data(), data.size());
}
```

1.  **打开 (`Open`)**: `Open` 方法的核心是初始化 `VarLenDataReader`。`VarLenDataReader` 是 `VarLenDataWriter` 的读取对应体，它会打开 `summary.offset` 和 `summary.data` 文件。通常，它会通过 `mmap` 将 `offset` 文件完整映射到内存，因为 `offset` 文件访问频繁且相对较小。`data` 文件则根据需要进行读取。
2.  **获取文档 (`GetDocument`)**: 当请求一个 `localDocId` 时，`GetDocument` 委托 `mDataReader->GetValue` 执行。`GetValue` 的逻辑是：
    a.  通过内存中的 `offset` 数组，直接计算出 `offset[localDocId]` 和 `offset[localDocId+1]`。
    b.  根据这两个偏移量，计算出数据在 `summary.data` 文件中的位置和长度。
    c.  从 `summary.data` 文件中读取出这段二进制数据（`autil::StringView`）。
    d.  获取到原始数据后，调用 `SummaryGroupFormatter::DeserializeSummary` 将二进制数据反序列化，填充到 `SearchSummaryDocument` 中。

### 3.2. 异步化改造：`GetDocument` 的 Coroutine 版本

为了应对高并发场景，`LocalDiskSummarySegmentReader` 提供了 `GetDocument` 的异步版本，使用了 C++20 的 Coroutine（通过 `future_lite` 库）。

```cpp
// indexlib/index/normal/summary/local_disk_summary_segment_reader.cpp
future_lite::coro::Lazy<vector<index::ErrorCode>>
LocalDiskSummarySegmentReader::GetDocument(const std::vector<docid_t>& docIds, autil::mem_pool::Pool* sessionPool,
                                           file_system::ReadOption readOption, const SearchSummaryDocVec* docs) noexcept
{
    // ...
    auto ret = co_await mDataReader->GetValue(docIds, sessionPool, readOption, &datas);
    // ...
    for (size_t i = 0; i < ret.size(); ++i) {
        // ... deserialize ...
    }
    co_return ec;
}
```

这里的关键是 `co_await mDataReader->GetValue(...)`。`mDataReader` 的异步 `GetValue` 会将底层的 `file_system::FileReader` 的异步读操作（`BatchRead`）包装成一个 `Lazy` 对象。当这个 `Lazy` 对象被 `co_await` 时，它会发起异步 I/O 请求，然后将当前协程挂起。当 I/O 操作完成后，执行线程会恢复该协程，并从 `co_await` 处继续执行后续的反序列化逻辑。这使得 I/O 等待期间，线程可以去执行其他任务，极大地提升了系统的吞吐能力。

### 3.3. `LocalDiskSummaryReader`：用户视角的全局读取器

`LocalDiskSummaryReader` 实现了 `SummaryReader` 接口，是用户直接交互的类。它管理了分区（Partition）中所有已构建（Built）段的 `LocalDiskSummarySegmentReader` 和正在构建（Building）段的 `BuildingSummaryReader`。

**核心逻辑与实现：**

```cpp
// indexlib/index/normal/summary/local_disk_summary_reader.h
class LocalDiskSummaryReader : public SummaryReader
{
    // ...
private:
    std::vector<LocalDiskSummarySegmentReaderPtr> mSegmentReaders;
    std::vector<uint64_t> mSegmentDocCount;
    BuildingSummaryReaderPtr mBuildingSummaryReader;
    docid_t mBuildingBaseDocId;
    // ...
};

// indexlib/index/normal/summary/local_disk_summary_reader.cpp
bool LocalDiskSummaryReader::Open(const PartitionDataPtr& partitionData, ...)
{
    // 1. Iterate over built segments
    while (builtSegIter->IsValid()) {
        // ... LoadSegmentReader for each segment ...
        mSegmentReaders.push_back(...);
        mSegmentDocCount.push_back(segmentInfo->docCount);
        // ...
    }

    // 2. Get building base docid
    mBuildingBaseDocId = segIter->GetBuildingBaseDocId();

    // 3. Init building summary reader
    InitBuildingSummaryReader(buildingSegIter);
    return true;
}

bool LocalDiskSummaryReader::GetDocumentFromSummary(docid_t docId, ...) const
{
    if (docId >= mBuildingBaseDocId) { // Is in building segment?
        return mBuildingSummaryReader && mBuildingSummaryReader->GetDocument(docId, summaryDoc);
    }
    
    docid_t baseDocId = 0;
    for (uint32_t i = 0; i < mSegmentDocCount.size(); i++) {
        if (docId < baseDocId + (docid_t)mSegmentDocCount[i]) {
            return mSegmentReaders[i]->GetDocument(docId - baseDocId, summaryDoc);
        }
        baseDocId += mSegmentDocCount[i];
    }
    return false;
}
```

1.  **打开 (`Open`)**: `Open` 方法会遍历 `PartitionData` 提供的所有 Segment。对于已构建的 Segment，它会创建一个 `LocalDiskSummarySegmentReader` 并加载。同时，它会累加每个 Segment 的文档数，用于后续 `docid` 的定位。对于构建中的 Segment，它会初始化 `BuildingSummaryReader`（我们将在下一章节分析）。
2.  **获取文档 (`GetDocumentFromSummary`)**: 这是 `docid` 路由的核心逻辑。
    *   首先判断 `docId` 是否大于等于 `mBuildingBaseDocId`。如果是，说明该文档在内存中的构建段里，请求被转发给 `mBuildingSummaryReader`。
    *   如果不是，则遍历 `mSegmentReaders` 列表。通过 `docId` 与累加的 `mSegmentDocCount` 比较，找到 `docId` 所属的 Segment，计算出其 `localDocId` (`docId - baseDocId`)，然后调用对应 `LocalDiskSummarySegmentReader` 的 `GetDocument` 方法。

**异步化 (`GetDocumentAsync`)** 的逻辑与同步版本类似，但它会将对不同 Segment 的批量请求构建成多个并行的 `Lazy` 任务，并通过 `future_lite::coro::collectAll` 并发执行，最大化 I/O 并行度。

## 4. 数据合并：`LocalDiskSummaryMerger`

随着增量构建的进行，会产生大量的小 Segment，影响查询性能。`LocalDiskSummaryMerger` 负责将这些小段合并成一个大段。

它继承自 `SummaryMerger`，而 `SummaryMerger` 又继承自通用的 `GroupFieldDataMerger`。合并的核心逻辑在 `GroupFieldDataMerger::Merge` 中实现，`LocalDiskSummaryMerger` 主要负责提供一些定制化的参数和目录。

**核心逻辑（在父类中）：**

1.  **初始化**: 创建一个 `VarLenDataWriter` 用于向新段写入数据。
2.  **创建段读取器**: 为每一个待合并的旧段（source segment）创建一个 `SummarySegmentReader`。
3.  **遍历与数据拷贝**: 遍历新段的 `docid`（从 0 到 `newSegDocCount - 1`）。对于每个 `docid`：
    a.  使用 `ReclaimMap` 或 `SegmentMapper` 找到这个新 `docid` 对应的旧 `docid` 和旧段索引。
    b.  如果 `docid` 有效（未被删除），则使用对应旧段的 `SummarySegmentReader` 读取出完整的 Summary 二进制数据。
    c.  将读取出的数据通过 `VarLenDataWriter` 写入到新段的 `summary.data` 和 `summary.offset` 文件中。
4.  **持久化**: 完成所有 `docid` 的拷贝后，调用 `VarLenDataWriter` 的 `Dump` 方法，完成新段的持久化。

`LocalDiskSummaryMerger` 的**特化之处**在于 `CreateVarLenDataParam`、`CreateInputDirectory` 和 `CreateOutputDirectory` 这几个虚函数。它根据 `SummaryGroupConfig` 来正确地创建数据参数，并处理带组名（Group Name）的子目录逻辑，确保合并过程能正确地找到输入文件和创建输出目录。

## 5. 技术风险与考量

*   **I/O 瓶颈**: Summary 数据通常较大，磁盘 I/O 是主要瓶颈。当前的异步化改造能有效提升吞吐，但对于延迟敏感的应用，仍需关注磁盘性能和缓存策略（如 `FileBlockCache`）。
*   **文件句柄与 MMap 限制**: 在一个拥有大量 Segment 的索引中，`LocalDiskSummaryReader` 会为每个 Segment 打开至少两个文件句柄（data 和 offset）。这可能超出系统的文件句柄限制。同时，大量使用 `mmap` 会消耗大量虚拟内存地址空间，在 32 位系统或资源受限的环境下是风险点。
*   **数据压缩与解压开销**: 虽然压缩能减少磁盘空间和 I/O 流量，但查询时的解压会消耗 CPU。需要根据数据特征和查询模式，在压缩率和解压速度之间做出权衡。`SummaryGroupConfig` 中提供了多种压缩算法选项。
*   **合并放大（Merge Amplification）**: 合并过程本身会读写大量数据，造成 I/O 放大。不合理的合并策略可能导致系统大部分时间都在进行内部数据整理，影响服务稳定性。Indexlib 的 Merge Policy 机制是应对此问题的关键。

通过对磁盘 I/O 和合并机制的深入分析，我们可以看到 Indexlib 在经典存储结构之上，通过异步化、分层设计和组件化，构建了一个兼具高性能和高扩展性的 Summary 系统。
