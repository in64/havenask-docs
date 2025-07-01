
# 核心文档表示：`Source` 与 `Summary` 文档深度解析

**涉及文件:**
* `document/normal/SourceDocument.h`
* `document/normal/SourceDocument.cpp`
* `document/normal/SummaryDocument.h`
* `document/normal/SummaryDocument.cpp`
* `document/normal/SearchSummaryDocument.h`
* `document/normal/SearchSummaryDocument.cpp`

## 1. 引言：文档——搜索引擎的基石

在信息检索系统中，"文档" 是最核心、最基本的单元。它不仅是信息的载体，也是整个索引、检索、摘要生成流程的起点。`indexlib` 作为 `Havenask` 的核心存储和索引引擎，其对文档的抽象和实现直接决定了系统的性能、灵活性和可扩展性。本文将深入剖析 `indexlib` 中三种核心的文档表示形式：`SourceDocument`、`SummaryDocument` 和 `SearchSummaryDocument`，揭示它们的设计哲学、内部实现、以及它们之间既分工明确又紧密协作的复杂关系。

理解这三种文档的本质，是理解 `indexlib` 如何将非结构化的原始数据转化为高度优化的、可供毫秒级检索的索引结构的关键。我们将从系统设计的角度出发，探讨这些 C++ 类如何通过精巧的数据结构和高效的内存管理，支撑起一个高性能的搜索引擎后端。

## 2. `SourceDocument`：数据的源头与契约

`SourceDocument` 是 `indexlib` 数据处理管道的入口。它代表了从外部数据源（如数据库、日志文件、消息队列）接收到的最原始的、未经处理的文档数据。它的核心职责是 **完整地、无损地** 承载一份文档的所有字段和信息。

### 2.1 设计目标与核心思想

`SourceDocument` 的设计目标非常明确：

*   **完整性 (Completeness):** 必须能够容纳一个文档的所有字段，无论这些字段最终是否会被索引、存储或用于摘要。它是一份文档的 "全貌"。
*   **灵活性 (Flexibility):** 设计上需要能够处理不同类型、不同数量的字段，适应业务 schema 的变化。
*   **解耦 (Decoupling):** 作为数据源和索引处理流程之间的 "契约"，`SourceDocument` 将上游数据生产方和下游 `indexlib` 内部处理逻辑解耦开来。上游只需要按照约定构造 `SourceDocument`，而无需关心 `indexlib` 内部复杂的索引构建细节。

### 2.2 关键实现细节

`SourceDocument` 的核心是其内部的数据存储机制。它通常使用一个 `std::vector` 或类似的动态数组来存储 `autil::StringView`，每个 `StringView` 代表一个字段的值。

```cpp
// document/normal/SourceDocument.h

class SourceDocument
{
public:
    // ...
    void SetField(fieldid_t fieldId, const autil::StringView& value);
    const autil::StringView& GetField(fieldid_t fieldId) const;
    // ...

private:
    std::vector<autil::StringView> _fields;
    // ...
};
```

这种设计的精妙之处在于：

1.  **`autil::StringView` 的妙用:** `StringView` 本身并不拥有字符串数据，而是指向一个已存在的内存区域。这意味着在创建 `SourceDocument` 时，字段数据可以 **零拷贝 (Zero-Copy)** 地从原始数据缓冲区（例如，从网络或文件中读取的 buffer）"借用" 过来，极大地提升了数据接入的性能。数据的实际生命周期由外部的内存池（`autil::mem_pool::Pool`）管理。
2.  **`fieldid_t` 作为索引:** 字段的访问不是通过字段名（`std::string`），而是通过一个整数类型的 `fieldid_t`。这是一个至关重要的性能优化。在 `indexlib` 中，`TabletSchema` 会预先将每个字段名映射到一个唯一的 `fieldid_t`。通过使用整数 ID 作为 `std::vector` 的下标，字段的查找操作从 `O(logN)` 或 `O(N)`（如果用 `std::map` 或遍历查找）降低到了 **`O(1)`** 的时间复杂度。

### 2.3 技术风险与考量

*   **内存管理:** `SourceDocument` 的零拷贝特性是一把双刃剑。它虽然高效，但也引入了复杂的内存生命周期管理问题。持有 `SourceDocument` 的代码必须确保其引用的内存池（`Pool`）在 `SourceDocument` 被销毁之前一直有效。如果内存池被提前释放，`StringView` 将指向无效内存，导致悬挂指针和程序崩溃。`indexlib` 通过精细的 `document::DocumentBatch` 和内存池管理机制来规避这一风险。
*   **数据一致性:** `SourceDocument` 自身不包含 schema 信息。这意味着，创建和使用 `SourceDocument` 的代码必须保证 `fieldid_t` 的使用与 `TabletSchema` 的定义严格一致。任何不匹配都可能导致数据错乱，例如，将一个 `INT` 类型字段的数据错误地赋给了 `STRING` 类型的字段。

## 3. `SummaryDocument`：为 "摘要" 而生

当一个 `SourceDocument` 进入 `indexlib` 后，它会被 "分发" 到不同的处理流程中。其中一个最重要的流程就是构建 "摘要"（Summary）。摘要是用户在搜索结果列表中看到的内容片段，它需要快速返回。`SummaryDocument` 正是为此目的而设计的。

### 3.1 设计目标与核心思想

`SummaryDocument` 的核心思想是 **"按需提取，紧凑存储"**。

*   **数据子集 (Subset of Data):** 与 `SourceDocument` 不同，`SummaryDocument` 只包含那些在 `TabletSchema` 中被标记为需要存储在摘要（`summary`）中的字段。它丢弃了那些只用于索引而不用于展示的字段。
*   **紧凑存储 (Compact Storage):** 它的内部表示被高度优化，以便能够被高效地序列化并写入磁盘。目标是最小化磁盘占用和 I/O 开销。
*   **快速访问 (Fast Access):** 在查询阶段，能够从磁盘快速反序列化，并提供高效的字段访问接口。

### 3.2 关键实现细节

`SummaryDocument` 的内部通常是一个连续的内存块，其中包含了所有摘要字段的数据。这种设计非常有利于序列化和反序列化。

```cpp
// document/normal/SummaryDocument.h

class SummaryDocument
{
public:
    // ...
    char* GetBuffer() { return _buffer; }
    size_t GetBufferLen() const { return _len; }

    // ... (serialize/deserialize methods)

private:
    char* _buffer;
    size_t _len;
    // ...
};
```

它的序列化格式大致如下：

1.  **头部 (Header):** 包含字段数量、每个字段的偏移量等元信息。
2.  **数据体 (Body):** 紧凑地存储着每个摘要字段的二进制数据。

这种 "指针+偏移量" 的设计使得在反序列化后，可以 `O(1)` 地定位到任何一个字段的数据，而无需进行字符串解析或查找。

### 3.3 与 `SourceDocument` 的转换

从 `SourceDocument` 到 `SummaryDocument` 的转换是一个关键步骤，通常由 `SummaryFormatter` 类来完成。这个过程涉及到：

1.  遍历 `TabletSchema` 中定义的所有摘要字段。
2.  根据 `fieldid_t` 从 `SourceDocument` 中获取对应字段的 `StringView`。
3.  将这些 `StringView` 指向的数据拷贝到一个新的、连续的内存缓冲区中。
4.  构建 `SummaryDocument` 的头部信息（字段数量、偏移量等）。
5.  最终创建一个指向这个新内存缓冲区的 `SummaryDocument` 对象。

这个过程虽然涉及数据拷贝，但是是必要的，因为它将数据从 `SourceDocument` 的 "零散" 引用状态，转换为了 `SummaryDocument` 的 "集中" 持有状态，为后续的独立存储和检索做好了准备。

## 4. `SearchSummaryDocument`：查询时的高效代理

当用户发起一次搜索请求后，`indexlib` 的 `Searcher` 会根据查询条件找到匹配的 `docid` 列表。接着，需要根据这些 `docid` 去获取每个文档的摘要信息。`SearchSummaryDocument` 在这个阶段扮演了关键角色。

### 4.1 设计目标与核心思想

`SearchSummaryDocument` 的设计目标是 **"查询时的高效、只读访问"**。

*   **懒加载 (Lazy Loading / On-demand Access):** 它本身不存储摘要数据。相反，它持有一个指向磁盘或内存中序列化 `SummaryDocument` 数据的指针（通常通过 `file_system::FileReader` 或内存缓存）。只有当上层应用（如 `ha3` 的 `SummaryExtractor`）实际需要某个字段时，它才会去解析那部分数据。
*   **只读接口 (Read-Only Interface):** 它提供与 `SummaryDocument` 类似的 `GetField` 接口，但这些接口都是 `const` 的，明确表示其只读属性。
*   **与 `SummaryDocument` 的统一接口:** 它提供了与 `SummaryDocument` 相似的接口，使得上层代码可以透明地处理这两种类型的摘要文档，而无需关心数据是来自内存中的 `SummaryDocument` 还是通过 `SearchSummaryDocument` 代理的磁盘数据。

### 4.2 关键实现细节

`SearchSummaryDocument` 的核心是其内部的解析逻辑。它知道 `SummaryDocument` 的序列化格式。

```cpp
// document/normal/SearchSummaryDocument.cpp

const autil::StringView& SearchSummaryDocument::GetField(fieldid_t fieldId) const
{
    // 1. Check if the fieldId is valid for summary
    // 2. Find the offset for this fieldId from the header of the serialized data
    // 3. Calculate the start and end pointers for the field data
    // 4. Return a StringView pointing to that data slice
    // ...
}
```

当 `GetField` 被调用时，它会执行以下操作：

1.  **查找元信息:** 从序列化的数据块的头部读取元信息，找到 `fieldId` 对应的偏移量（offset）和长度（length）。
2.  **计算指针:** 根据基地址、偏移量和长度，计算出该字段数据在内存中的起始和结束位置。
3.  **返回 `StringView`:** 创建一个 `autil::StringView`，使其指向这块内存区域，并返回给调用者。

整个过程同样是 **零拷贝** 的。`SearchSummaryDocument` 就像一个高效的 "翻译官"，将上层的字段访问请求 "翻译" 成对底层二进制数据块的指针操作，避免了任何不必要的数据拷贝和反序列化开销。

## 5. 系统架构中的协作流程

现在，让我们将这三者串联起来，看看它们在 `indexlib` 的整个生命周期中是如何协作的：

**阶段一：索引构建 (Build Phase)**

1.  **数据注入:** 外部数据源产生原始数据，`indexlib` 的 `builder` 模块接收数据，并将其封装到一个 `autil::mem_pool::Pool`管理的内存 `Buffer` 中。
2.  **创建 `SourceDocument`:** `DocumentFactory` 根据 `TabletSchema` 创建 `SourceDocument`。通过 `SetField` 方法，将 `Buffer` 中的数据以 `StringView` 的形式 "借用" 给 `SourceDocument`。这是一个零拷贝的过程。
3.  **分发处理:**
    *   **索引字段 (Index Fields):** `SourceDocument` 被传递给 `IndexFieldParser`，后者解析出需要建立倒排索引、正排索引的字段，并将其送入相应的索引写入器（`IndexWriter`）。
    *   **摘要字段 (Summary Fields):** `SourceDocument` 被传递给 `SummaryFormatter`。
4.  **创建 `SummaryDocument`:** `SummaryFormatter` 从 `SourceDocument` 中提取出所有摘要字段，将它们拷贝并序列化到一个紧凑的内存块中，然后创建 `SummaryDocument` 对象。
5.  **写入磁盘:** `SummaryWriter` 将 `SummaryDocument` 的序列化数据写入磁盘上的 `summary` 文件中。同时，记录下该文档的 `docid` 及其在文件中的偏移量，这个映射关系会保存在 `summary.offset` 文件中。

**阶段二：查询服务 (Search Phase)**

1.  **检索 `docid`:** `Searcher` 根据用户查询，通过倒排索引等找到匹配的文档 `docid` 列表。
2.  **获取摘要:** `Searcher` 需要为这些 `docid` 获取摘要。它会调用 `SummaryReader`。
3.  **定位数据:** `SummaryReader` 根据 `docid`，从 `summary.offset` 文件中查到该文档摘要数据在 `summary` 文件中的偏移量和长度。
4.  **读取数据:** `SummaryReader` 从 `summary` 文件中将对应的数据块读入内存（或直接利用 `mmap`）。
5.  **创建 `SearchSummaryDocument`:** `SummaryReader` 使用刚刚读入内存的数据块和 `TabletSchema` 来创建一个 `SearchSummaryDocument` 对象。这个对象现在是磁盘上摘要数据的一个高效、只读的内存代理。
6.  **摘要提取:** `SearchSummaryDocument` 被传递给上层应用（如 `ha3` 的 `SummaryExtractor`）。`SummaryExtractor` 调用 `GetField` 方法，`SearchSummaryDocument` 执行其零拷贝的指针解析逻辑，返回字段的 `StringView`。
7.  **生成片段:** `SummaryExtractor` 利用获取到的字段内容，结合查询的关键词，生成最终呈现给用户的搜索结果摘要片段。

## 6. 结论：精心设计的文档抽象层次

`indexlib` 的 `SourceDocument`, `SummaryDocument`, 和 `SearchSummaryDocument` 共同构成了一个层次分明、职责清晰的文档处理体系。

*   **`SourceDocument`** 是通用的、面向 **数据生产者** 的接口，以其灵活性和零拷贝的接入效率，构成了数据管道的入口。
*   **`SummaryDocument`** 是专用的、面向 **持久化存储** 的数据结构，以其紧凑的序列化格式，优化了磁盘 I/O 和存储成本。
*   **`SearchSummaryDocument`** 是高效的、面向 **查询消费者** 的代理，以其懒加载和零拷贝的访问模式，保证了摘要获取的低延迟。

这个三层抽象的设计，是典型的高性能后台系统设计思想的体现：**在系统的不同阶段，针对不同的性能瓶颈，采用不同的数据表示和优化策略。** 通过在 `Source` -> `Summary` -> `SearchSummary` 的转换链条中精心设计数据结构和算法，`indexlib` 成功地将一份原始、非结构化的数据，高效地转化为了服务于高性能检索的、高度优化的索引和摘要数据，最终为用户提供了毫秒级的搜索体验。对这套机制的理解，是深入掌握 `Havenask` 乃至现代搜索引擎内核实现的关键所在。
