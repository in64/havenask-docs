
# 文档序列化：`Source` 与 `Summary` 的持久化与传输机制

**涉及文件:**
* `document/normal/SerializedSourceDocument.h`
* `document/normal/SerializedSourceDocument.cpp`
* `document/normal/SerializedSummaryDocument.h`
* `document/normal/SerializedSummaryDocument.cpp`

## 1. 引言：为何需要序列化？

在分布式系统中，数据需要在不同节点间传输，也需要被持久化到磁盘上。`indexlib` 作为 `Havenask` 的存储引擎，其处理的文档对象（如 `SourceDocument` 和 `SummaryDocument`）本质上是内存中的 C++ 对象，包含指针、引用等复杂结构。这些内存对象无法被直接写入文件或通过网络发送。因此，必须将它们转换为一种线性的、可存储、可传输的字节流格式——这个过程就是 **序列化 (Serialization)**。反之，从字节流恢复为内存对象的过程，则称为 **反序列化 (Deserialization)**。

本文将深入剖析 `indexlib` 中专门负责此项任务的两个关键类：`SerializedSourceDocument` 和 `SerializedSummaryDocument`。我们将揭示它们如何将内存中的文档对象高效地转化为紧凑的二进制格式，其背后的设计考量、技术实现以及它们在系统中的关键作用。

## 2. `SerializedSummaryDocument`：紧凑的摘要快照

`SerializedSummaryDocument` 的目标非常明确：将 `SummaryDocument`（或 `SearchSummaryDocument`）的内容进行序列化，以便存储或传输。它的设计核心是 **效率** 和 **紧凑性**。

### 2.1 设计目标与核心思想

*   **数据保真性:** 序列化后的数据必须能完整、精确地还原出原始 `SummaryDocument` 的所有字段信息。
*   **空间效率:** 序列化格式需要尽可能紧凑，以减少磁盘占用和网络带宽消耗。这对于大规模搜索引擎至关重要，因为摘要数据通常会占据相当大的存储空间。
*   **时间效率:** 序列化和反序列化的过程必须非常快，以降低索引构建的延迟和查询时恢复摘要的开销。

### 2.2 序列化格式剖析

`SerializedSummaryDocument` 的实现非常精巧，它并没有定义一种全新的序列化格式，而是 **复用并固化了 `SummaryDocument` 本身的内存布局**。正如在核心文档表示分析中所述，`SummaryDocument` 内部就是一个连续的内存块，其结构天然适合序列化。

这个内存块的格式通常如下：

```
[ Header (元数据) | Field 1 Data | Field 2 Data | ... | Field N Data ]
```

*   **Header:** 头部区域存储了元信息，这是反序列化的关键。它通常包含：
    *   `docid_t docid`: 文档的 ID。
    *   `int64_t version`: 文档的版本时间戳。
    *   `uint32_t count`: 摘要字段的数量。
    *   `uint32_t offsets[count]`: 一个偏移量数组，记录每个字段数据相对于数据区起点的字节偏移量。
    *   `uint32_t totalLen`: 整个数据区的总长度。

*   **Data Area:** 紧跟在 Header 之后，是所有摘要字段的值被紧密地拼接在一起的二进制数据。

`SerializedSummaryDocument` 的 `serialize` 方法，本质上就是将 `SummaryDocument` 的内部 `_buffer` 内容（已经符合上述格式）拷贝出来。而 `deserialize` 方法，则是将外部传入的字节流直接加载，并提供方法来访问其中的各个部分。

### 2.3 关键实现细节

`SerializedSummaryDocument` 自身并不持有庞大的数据实体，它更像是一个轻量级的 **视图 (View)** 或 **包装器 (Wrapper)**。它内部只存储指向序列化数据块的指针和长度。

```cpp
// document/normal/SerializedSummaryDocument.h

class SerializedSummaryDocument
{
public:
    // ...
    // Attach to an existing serialized data buffer
    void SetData(const char* data, size_t len);

    // Accessor methods
    docid_t GetDocId() const;
    int64_t GetVersion() const;
    const char* GetField(fieldid_t fieldId, size_t& length) const;
    // ...

private:
    const char* _data; // Pointer to the serialized data
    size_t _size;      // Size of the data
};
```

其 `GetField` 方法的实现逻辑与 `SearchSummaryDocument` 类似，都是通过解析头部的偏移量数组来定位字段数据，并返回一个指向该数据的指针和长度。这是一个零拷贝的读取过程。

```cpp
// document/normal/SerializedSummaryDocument.cpp

const char* SerializedSummaryDocument::GetField(fieldid_t fieldId, size_t& length) const
{
    // 1. Parse header from _data to get field count and offsets array pointer.
    // 2. Find the index for the given fieldId in the schema's summary fields.
    // 3. Use the index to look up the offset in the offsets array.
    // 4. Calculate the start position and length of the field data.
    // 5. Set the output 'length' and return the pointer to the start of the data.
    // ...
}
```

这种设计使得 `SerializedSummaryDocument` 成为一个在序列化数据和结构化访问之间的完美桥梁，无需进行完整的反序列化（即重新创建 `SummaryDocument` 对象），就能高效地读取所需信息。

## 3. `SerializedSourceDocument`：为跨集群传输而生

`SerializedSourceDocument` 的应用场景与 `SerializedSummaryDocument` 有所不同。它主要用于 **全量数据** 的移动，尤其是在 `Havenask` 的 `BS` (Build Service) 架构中，当 `Builder` 节点需要将处理好的 `DocumentBatch` 发送给 `Merger` 节点时。

### 3.1 设计目标与核心思想

*   **完整性:** 必须能够序列化 `SourceDocument` 的 **所有字段**，而不仅仅是摘要字段。这是因为它服务于后续的索引构建或合并流程，这些流程可能需要访问任何字段。
*   **批量处理:** `SourceDocument` 通常以 `DocumentBatch` 的形式出现。因此，序列化方案需要高效地处理一批文档，而不是单个文档。
*   **自包含性 (Self-Contained):** 序列化后的数据应该包含所有必要的信息，包括文档数据、操作类型（ADD/DELETE/UPDATE）等，以便接收方可以完全重建 `DocumentBatch`。

### 3.2 序列化格式剖析

`SerializedSourceDocument` 的序列化格式比 `SerializedSummaryDocument` 更复杂，因为它需要处理可变数量、可变长度的字段，并且需要保留每个字段的 `fieldid_t`。

一个 `SerializedSourceDocument` 的二进制流大致结构如下：

```
[ Header | Field 1 Meta | Field 2 Meta | ... | Field N Meta | Field 1 Data | Field 2 Data | ... | Field N Data ]
```

1.  **Header:** 包含文档级别的元信息，如 `docid`, `version`, 操作类型 (`DocOperateType`)，以及最重要的——字段数量 (`field_count`)。
2.  **Field Meta Section:** 这是一个元数据数组，每个元素对应一个字段，包含了：
    *   `fieldid_t fieldId`: 字段的 ID。
    *   `uint32_t offset`: 该字段数据在后面数据区的偏移量。
    *   `uint32_t length`: 该字段数据的长度。
3.  **Field Data Section:** 所有字段的 `StringView` 内容被依次拷贝、拼接在这里。

这种 **"元数据与数据分离"** 的设计是处理异构、可变长度数据的经典方法。它允许接收方首先读取固定大小的元数据区，从而了解整个数据块的布局（有哪些字段，它们在哪里，有多长），然后再按需、高效地访问实际的数据区。

### 3.3 关键实现细节

`SerializedSourceDocument` 同样采用轻量级包装器的设计，内部持有指向序列化数据的指针。

```cpp
// document/normal/SerializedSourceDocument.h

class SerializedSourceDocument
{
public:
    // ...
    // Methods to serialize a SourceDocument into a buffer
    static bool serialize(const SourceDocument* doc, autil::DataBuffer& dataBuffer);

    // Attach to existing serialized data
    void fromBuffer(const char* buffer, size_t len);

    // Accessor methods
    const autil::StringView& getField(fieldid_t fieldId) const;
    // ...

private:
    const char* _data = nullptr;
    size_t _size = 0;
    // ... maybe some parsed header info for quick access
};
```

*   **序列化 (`serialize`):** 这个静态方法是实现的核心。它会：
    1.  遍历 `SourceDocument` 中的所有有效字段。
    2.  为每个字段准备一个元数据条目（`fieldId`, `offset`, `length`）。
    3.  将所有字段的数据拷贝到一个临时的连续 `Buffer` 中。
    4.  计算每个字段在 `Buffer` 中的 `offset` 和 `length`，填充元数据。
    5.  最后，将文档 Header、所有字段的元数据、以及 `Buffer` 中的数据，依次写入到 `autil::DataBuffer` 中。

*   **反序列化/访问 (`getField`):** 当 `getField` 被调用时，它会：
    1.  遍历头部的字段元数据区。
    2.  通过 `fieldId` 匹配找到对应的元数据条目。
    3.  从元数据中获得 `offset` 和 `length`。
    4.  基于 `_data` 指针、`offset` 和 `length`，构造一个 `autil::StringView` 并返回。这同样是零拷贝的。

## 4. 系统中的应用场景

*   **`SerializedSummaryDocument` 的应用:**
    *   **行存 (Row Store) 模式:** 在 `indexlib` 的 `KV` 或 `KKV` 索引中，`value` 部分通常就存储着 `SerializedSummaryDocument` 的数据。当通过 `key` 查找到 `value` 后，可以直接用 `SerializedSummaryDocument` 进行包装和访问。
    *   **摘要缓存:** 在查询服务中，从磁盘读取的 `summary` 数据块可以被缓存起来。`SerializedSummaryDocument` 可以直接作用于这些缓存的内存块，避免了每次都从磁盘 I/O。

*   **`SerializedSourceDocument` 的应用:**
    *   **分布式构建:** 在 `Havenask` 的 `BS` (Build Service) 中，`Builder` 进程完成对 `DocumentBatch` 的初步处理（例如 tokenization）后，会将整个 `DocumentBatch` 序列化，并通过网络发送给 `Merger` 进程。这个序列化的核心就是 `SerializedSourceDocument`。`Merger` 接收到字节流后，可以快速地反序列化出 `DocumentBatch`，并继续后续的索引合并操作。
    *   **Redo Log / WAL:** 在需要高可靠性的场景下，可以将接收到的 `DocumentBatch` 以序列化的形式写入预写日志 (Write-Ahead Log)。当系统发生故障重启时，可以通过读取 WAL 并反序列化 `SerializedSourceDocument` 来恢复数据，确保数据不丢失。

## 5. 结论：为持久化和移动而生的二进制契约

`SerializedSummaryDocument` 和 `SerializedSourceDocument` 是 `indexlib` 文档体系中负责 **"落地"** 和 **"远行"** 的关键组件。它们共同定义了一套高效的二进制契约，将内存中复杂的 C++ 对象转换为了可持久化、可传输的字节流。

*   **`SerializedSummaryDocument`** 通过复用 `SummaryDocument` 的内存布局，实现了极致的紧凑性和访问效率，完美服务于摘要的存储和快速读取场景。
*   **`SerializedSourceDocument`** 通过 "元数据+数据" 的灵活格式，实现了对 `SourceDocument` 的完整信息编码，是分布式构建和数据复制等场景下不可或缺的一环。

这两个类的设计充分体现了高性能存储引擎的普遍原则：**针对不同数据的特性和应用场景，设计专门的序列化方案。** 通过对数据布局的精心安排，它们在保证数据完整性的前提下，最大限度地减少了空间占用和计算开销，为 `Havenask` 系统整体的稳定性、高性能和可扩展性提供了坚实的基础。
