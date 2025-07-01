
# 数据格式化与提取：从 `Source` 到 `Summary` 和 `Group` 的变形记

**涉及文件:**
* `document/normal/SourceFormatter.h`
* `document/normal/SourceFormatter.cpp`
* `document/normal/SummaryFormatter.h`
* `document/normal/SummaryFormatter.cpp`
* `document/normal/GroupFieldFormatter.h`
* `document/normal/GroupFieldFormatter.cpp`
* `document/normal/SummaryGroupFormatter.h`
* `document/normal/SummaryGroupFormatter.cpp`

## 1. 引言：数据处理的十字路口

在 `indexlib` 的数据处理流水线中，`SourceDocument` 如同一个满载各种原材料的货车，它包含了所有可能用到的信息。然而，不同的下游工序——如正排索引、倒排索引、摘要存储、分组聚合——需要的 “零件” 各不相同。直接让每个工序都从 `SourceDocument` 这辆大货车上自行翻找，无疑是低效且混乱的。

因此，`indexlib` 设计了一系列 **`Formatter` (格式化器)** 类。它们如同一个个智能分拣机器人，站在数据处理的十字路口，其核心职责就是：**根据预设的规则 (Schema)，从 `SourceDocument` 中精准地提取出特定的字段子集，并将它们组装成特定格式的、服务于专门目的的 “物料包”**。本文将深入剖析 `SourceFormatter`、`SummaryFormatter`、`GroupFieldFormatter` 等核心格式化器的设计理念、工作流程和技术实现，揭示它们如何高效地完成数据的 “变形” 与分发。

## 2. `SummaryFormatter`：构建摘要的工匠

`SummaryFormatter` 是最核心、最常用的格式化器之一。它的任务是从 `SourceDocument` 中提取出所有需要在搜索结果中展示的字段，并创建一个紧凑的、可供持久化存储的 `SummaryDocument`。

### 2.1 设计目标与核心思想

*   **按需提取:** 只关心在 `Schema` 中被标记为 “需要存储在摘要中” 的字段，忽略所有其他字段。
*   **格式转换:** 将提取出的字段数据，从 `SourceDocument` 的 `StringView` 引用形式，转换为 `SummaryDocument` 所需的、由其自身内存池管理的连续内存块格式。
*   **Schema 驱动:** 整个提取和格式化过程完全由 `indexlib::config::SummarySchema` 来驱动和定义。

### 2.2 工作流程详解

`SummaryFormatter` 的核心方法是 `format(const SourceDocument& sourceDoc, autil::mem_pool::Pool* pool)`。其内部执行流程如下：

1.  **获取 Schema:** 首先，`SummaryFormatter` 会持有 `SummarySchema` 的引用，这是它所有操作的依据。
2.  **计算所需空间:** 遍历 `SummarySchema` 中定义的所有摘要字段。对于每个字段，通过 `fieldId` 从 `sourceDoc` 中获取其 `StringView`，并累加所有字段值的长度。同时，还要加上存储元数据（如字段偏移量数组）所需的头部空间。
3.  **分配内存:** 使用传入的 `pool`，一次性分配一个足够大的、连续的内存块，用于容纳最终的 `SummaryDocument` 数据。
4.  **填充数据与元数据:**
    *   再次遍历 `SummarySchema` 中的摘要字段。
    *   将每个字段的数据从 `sourceDoc` 的 `StringView` **拷贝** 到新分配的内存块的指定位置。
    *   同时，在内存块的头部区域，记录下每个字段的偏移量（offset）。
5.  **创建 `SummaryDocument`:** 最后，使用这个填充好的内存块的地址和总长度，创建一个 `SummaryDocument` 对象并返回。

### 2.3 关键实现代码片段

```cpp
// document/normal/SummaryFormatter.cpp (Conceptual)

void SummaryFormatter::format(const SourceDocument& sourceDoc, 
                              autil::mem_pool::Pool* pool, 
                              SummaryDocument* summaryDoc)
{
    // 1. Get summary schema
    const auto& summarySchema = _schema->GetSummarySchema();
    size_t fieldCount = summarySchema->GetFieldCount();

    // 2. Calculate total length
    size_t totalLength = calculateHeaderLength(fieldCount);
    std::vector<size_t> fieldLengths;
    for (size_t i = 0; i < fieldCount; ++i) {
        fieldid_t fieldId = summarySchema->GetFieldId(i);
        const auto& fieldValue = sourceDoc.GetField(fieldId);
        fieldLengths.push_back(fieldValue.length());
        totalLength += fieldValue.length();
    }

    // 3. Allocate buffer
    char* buffer = (char*)pool->allocate(totalLength);

    // 4. Fill header and data
    char* headerCursor = buffer;
    fillHeader(headerCursor, fieldCount, ...);

    char* dataCursor = buffer + getHeaderLength(fieldCount);
    char* headerOffsetCursor = getOffsetsAddress(buffer);
    for (size_t i = 0; i < fieldCount; ++i) {
        fieldid_t fieldId = summarySchema->GetFieldId(i);
        const auto& fieldValue = sourceDoc.GetField(fieldId);
        
        // Copy data
        memcpy(dataCursor, fieldValue.data(), fieldValue.length());
        
        // Record offset
        setOffset(headerOffsetCursor, i, dataCursor - (buffer + getHeaderLength(fieldCount)));

        dataCursor += fieldValue.length();
    }

    // 5. Set to summaryDoc
    summaryDoc->SetBuffer(buffer, totalLength);
}
```

这个过程中的 **数据拷贝是不可避免且至关重要的**。它实现了数据的 “所有权转移”，将数据从 `SourceDocument` 临时的、由外部管理的生命周期中解放出来，固化到了 `SummaryDocument` 自己的内存空间里，为后续的独立存储和检索做好了准备。

## 3. `GroupFieldFormatter` 与 `SummaryGroupFormatter`：为分组聚合定制

在搜索引擎中，分组（`GROUP BY`）和聚合（`AGGREGATE`）是常见的查询需求。为了优化这类查询的性能，`indexlib` 允许将某些字段的数据以特殊格式存储，`GroupFieldFormatter` 和 `SummaryGroupFormatter` 就是为此而生的。

### 3.1 设计理念：数据冗余换性能

通常，分组操作需要依赖正排索引。但如果一个字段既要用于分组，又要在摘要中展示，每次都去分别读取正排索引和摘要，会产生两次随机 I/O，性能较差。`GroupFieldFormatter` 的设计理念就是 **通过适度的数据冗余来换取查询性能的提升**。

*   **`GroupFieldFormatter`:** 它的任务是从 `SourceDocument` 中提取出 `Schema` 中为分组（`group`）指定的字段，并将它们序列化成一个独立的、紧凑的值。这个值会作为一个 **“附加字段”**，被存储在 **正排索引** 的数据区中。
*   **`SummaryGroupFormatter`:** 它的功能类似，但是将提取出的分组字段值，作为一个 “附加字段”，存储在 **摘要** 数据中。

这样，在进行分组查询时，可以直接从正排索引或摘要中一次性读出所有需要的分组字段值，避免了多次 I/O。

### 3.2 序列化格式

这两个 Formatter 生成的不是一个完整的文档，而是一个二进制的 `StringView`。其内部格式通常是 **多值（multi-value）** 编码格式，因为一个分组操作可能涉及多个字段。

一个常见格式如下：

```
[ Header | Value 1 | Value 2 | ... | Value N ]
```

*   **Header:** 包含值的数量，以及每个值的偏移量或长度信息。
*   **Values:** 各个分组字段的值被依次拼接。

这种格式的设计重点在于 **快速反序列化**。`GroupFieldIter` 或 `SummaryGroupFieldIter` 这类专用的迭代器，可以作用于这个二进制 `StringView` 上，高效地逐个解析出字段值，而无需完整的反序列化开销。

### 3.3 关键实现细节

`GroupFieldFormatter` 的 `format` 方法接收 `SourceDocument`，返回一个 `StringView`。

```cpp
// document/normal/GroupFieldFormatter.cpp (Conceptual)

autil::StringView GroupFieldFormatter::format(const SourceDocument& sourceDoc, 
                                              autil::mem_pool::Pool* pool)
{
    // 1. Get group field schema
    const auto& groupSchema = _schema->GetGroupFieldSchema();

    // 2. Calculate size and prepare data views
    std::vector<autil::StringView> fieldValues;
    size_t totalLength = calculateHeaderSize(groupSchema->GetFieldCount());
    for (fieldid_t fieldId : groupSchema->GetFieldIds()) {
        const auto& val = sourceDoc.GetField(fieldId);
        fieldValues.push_back(val);
        totalLength += val.length();
    }

    // 3. Allocate buffer and fill data
    char* buffer = (char*)pool->allocate(totalLength);
    char* cursor = buffer;
    // ... write header (count, offsets) ...
    cursor += headerSize;
    for (const auto& val : fieldValues) {
        memcpy(cursor, val.data(), val.length());
        cursor += val.length();
    }

    return autil::StringView(buffer, totalLength);
}
```

这个过程与 `SummaryFormatter` 类似，都涉及计算空间、分配内存、拷贝数据和构建元信息。不同之处在于，它的产出物不是一个 `Document` 对象，而是一个封装了序列化数据的 `StringView`。

## 4. `SourceFormatter`：保留原始面貌

`SourceFormatter` 的角色相对特殊。在某些场景下，用户可能希望将原始的 `SourceDocument` **原封不动地** 保存下来，用于数据回溯、重新索引（re-indexing）或其他特殊分析。`SourceFormatter` 就是为此服务的。

它的核心功能是将 `SourceDocument` 序列化。但与 `SerializedSourceDocument` 不同，`SourceFormatter` 的产出通常是作为一个特殊的 “字段” 被存储起来。例如，可以定义一个名为 `__source__` 的字段，其类型为 `RAW`，然后 `SourceFormatter` 会将整个 `SourceDocument` 序列化后的二进制内容作为这个字段的值。

这个过程实际上就是调用了 `SerializedSourceDocument` 的序列化逻辑，将其封装成一个 `Formatter` 接口，以便统一地在 `indexlib` 的文档处理框架下使用。

## 5. 结论：模块化的数据转换引擎

`indexlib` 的 `Formatter` 体系是其数据处理流水线中至关重要的一环。它们将 `SourceDocument` 的处理过程清晰地解耦为 **“提取 + 格式化”** 的标准步骤，体现了优秀的设计原则：

*   **单一职责原则:** 每个 `Formatter` 只负责一种特定的格式转换，如 `SummaryFormatter` 只关心摘要，`GroupFieldFormatter` 只关心分组字段。
*   **Schema 驱动:** 所有的格式化行为都由外部的 `Schema` 配置来定义，使得数据处理逻辑与业务规则分离，提高了系统的灵活性和可配置性。
*   **性能优化:** 通过为特定场景（如摘要、分组）生成高度优化的数据布局，`Formatter` 在数据写入时进行预处理，从而极大地提升了查询时的性能。

`SummaryFormatter`、`GroupFieldFormatter` 等类共同构成了一个模块化的数据转换引擎。它们协同工作，将一份通用的 `SourceDocument`，高效、准确地 “裁剪” 和 “包装” 成适用于不同存储和计算场景的、标准化的 “物料包”，为 `indexlib` 后续的索引构建、合并、查询等一系列复杂操作提供了坚实、高效的数据基础。
