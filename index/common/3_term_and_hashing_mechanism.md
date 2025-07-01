# Indexlib `common` 模块索引词元与哈希机制深度解析

**涉及文件:**
*   `index/common/Term.h`, `index/common/Term.cpp`
*   `index/common/LiteTerm.h`
*   `index/common/NumberTerm.h`
*   `index/common/DictKeyInfo.h`, `index/common/DictKeyInfo.cpp`
*   `index/common/DictHasher.h`, `index/common/DictHasher.cpp`
*   `index/common/KeyHasherWrapper.h`, `index/common/KeyHasherWrapper.cpp`
*   `index/common/PrimaryKeyHashType.h`
*   `index/common/SortValueConvertor.h`, `index/common/SortValueConvertor.cpp`
*   `index/common/ShardPartitioner.h`, `index/common/ShardPartitioner.cpp`

## 1. 系统概述

在倒排索引（Inverted Index）的体系中，“词元”（Term）是信息检索的基本单位，而如何高效、一致地处理和标识这些词元，则是整个索引引擎性能的关键。Indexlib 的 `index/common` 模块中，与词元和哈希相关的组件共同构建了一个强大而灵活的系统，它不仅负责词元的表示，还涵盖了从词元到词典键（Dictionary Key）的转换、哈希计算、排序键生成以及分片策略等一系列核心功能。

这个系统的核心设计思想可以概括为**“分层表示与策略化哈希”**：

1.  **分层表示**: 系统提供了多种 `Term` 的表示形式。`Term` 类是功能最完整的表示，包含了词元文本、索引名等丰富信息。`LiteTerm` 则是其轻量化版本，在性能敏感的路径上，用数值ID（`indexid_t`, `dictkey_t`）替代字符串，以减少开销。`NumberTerm` 则专门用于表示数值范围，是范围查询的基础。这种分层设计使得系统可以在不同场景下选择最合适的表示方式，在功能完备性和极致性能之间取得平衡。

2.  **策略化哈希**: 系统将词元的哈希计算过程抽象成一种可配置的“策略”。`DictHasher` 和 `KeyHasherWrapper` 允许根据索引类型（如 `text`, `number`）或用户配置（如 `default`, `murmur`, `layerhash`）选择不同的哈希函数。这种策略模式使得哈希机制非常灵活，能够适应不同的业务需求和性能要求，例如，为数值类型使用专门的 Number-Hash，或为文本类型使用更复杂的 Layer-Hash。

除此之外，该系统还整合了 `SortValueConvertor` 和 `ShardPartitioner`，将排序和分片这两个与词元处理紧密相关的逻辑纳入统一的管理范畴，形成了一个功能内聚、设计完善的词元处理中枢。

## 2. 核心组件剖析

### 2.1. `Term` 家族: 词元的多样化表示

`Term` 是查询和索引的基本单元。Indexlib 提供了多种 `Term` 的实现以适应不同场景。

#### 2.1.1. `Term.h`: 通用词元表示

`Term` 类是所有词元表示的基类和最完整的形式。它包含了描述一个词元所需的全部信息：
*   `_word`: 词元的文本表示（`std::string`）。
*   `_indexName`: 该词元所属的索引字段名。
*   `_truncateName`: 用于截断索引（Truncate Index），表示该词元属于哪个截断链。
*   `_isNull`: 标记是否为空 Term。
*   `_liteTerm`: 内嵌一个 `LiteTerm` 对象，用于存储轻量化的表示。
*   `_hasValidHashKey`: 标记 `_liteTerm` 中的哈希键是否有效。

`Term` 类是系统对外接口和需要完整信息的场景下的标准选择。

#### 2.1.2. `LiteTerm.h`: 轻量化性能优化

`LiteTerm` 是 `Term` 的一个关键性能优化。在 Indexlib 内部的许多高性能路径上，反复创建和比较包含 `std::string` 的 `Term` 对象会带来显著开销。`LiteTerm` 通过将字符串标识替换为数值标识来解决这个问题：
*   `_termHashKey`: 词元的哈希值（`dictkey_t`），即其在词典中的唯一标识。
*   `_indexId`: 索引字段的数字 ID。
*   `_truncateIndexId`: 截断链的数字 ID。

在索引构建或查询解析的早期阶段，`Term` 对象中的字符串信息会被转换为 `LiteTerm` 中的数值 ID。在后续的处理流程中，系统就可以只传递轻量的 `LiteTerm` 对象，极大地提升了效率。`Term` 类中的 `EnableLiteTerm` 和 `GetLiteTerm` 方法，正是连接这两种表示的桥梁。

**代码示例: `Term` 与 `LiteTerm` 的关系**
```cpp
// In Term.h
class Term {
    // ...
protected:
    std::string _word;
    std::string _indexName;
    LiteTerm _liteTerm; // 内嵌 LiteTerm
    bool _hasValidHashKey;
};

// In LiteTerm.h
class LiteTerm {
public:
    LiteTerm(dictkey_t termHashKey, indexid_t indexId, ...);
    dictkey_t GetTermHashKey() const; 
    // ...
private:
    indexid_t _indexId;
    indexid_t _truncateIndexId;
    dictkey_t _termHashKey;
};
```

#### 2.1.3. `NumberTerm.h`: 数值与范围查询的特化

`NumberTerm` 是 `Term` 的一个子类，专门用于表示数值类型。它除了继承 `Term` 的基本属性外，还增加了：
*   `_leftNum`, `_rightNum`: 表示一个数值区间 `[left, right]`。

这使得 `NumberTerm` 不仅可以表示单个数值（此时 `_leftNum == _rightNum`），还可以表示一个查询范围。这是实现范围查询（Range Query）功能的基础。模板 `NumberTerm<T>` 的设计使其可以支持 `int32_t`, `int64_t` 等多种数值类型。

### 2.2. 哈希机制: 从 `Term` 到 `DictKeyInfo`

词典（Dictionary）是倒排索引的核心，它存储了所有词元到倒排拉链（Posting List）的映射。为了在词典中快速查找，需要将 `Term` 转换为一个固定长度的键，即 `DictKeyInfo`。

#### 2.2.1. `DictKeyInfo.h`: 词典键的最终形态

`DictKeyInfo` 是词元在词典中的唯一标识。它非常简单，主要包含：
*   `_dictKey`: 一个 `dictkey_t` (通常是 `uint64_t`) 类型的哈希值。
*   `_isNull`: 标记是否为空。

它是哈希计算流程的最终输出。

#### 2.2.2. `KeyHasherWrapper.h` 和 `DictHasher.h`: 策略化的哈希计算

这是哈希机制的核心，体现了策略模式的灵活性。

*   **`KeyHasherWrapper.cpp`**: 这是一个静态工具类，它封装了最底层的哈希算法调用。它根据 `FieldType` 或 `InvertedIndexType`，选择不同的哈希实现（如 `DefaultHasher`, `NumberHasher`, `MurmurHasher`）。这是哈希策略的“执行层”。

    **代码示例: `KeyHasherWrapper::GetHashKeyByFieldType`**
    ```cpp
    bool KeyHasherWrapper::GetHashKeyByFieldType(FieldType fieldType, const char* key, size_t size, dictkey_t& hashKey)
    {
        switch (fieldType) {
        case ft_int64:
            return util::Int64NumberHasher::GetHashKey(key, size, hashKey);
        // ... other number types
        default:
            // 默认使用 DefaultHasher
            return util::DefaultHasher::GetHashKey(key, size, hashKey);
        }
        assert(false);
        return false;
    }
    ```

*   **`DictHasher.h`**: 这是对 `KeyHasherWrapper` 的一层封装，它是有状态的，并且可以直接从用户配置（`KeyValueMap`）中初始化。它引入了更高级的哈希策略，如 `LayerTextHasher`，这种哈希器可能用于处理带有层次关系的文本。`DictHasher` 是哈希策略的“决策层”。

    **`DictHasher::GetHashKey` 的逻辑**: 
    1.  首先尝试从 `Term` 对象中直接获取已经计算好的哈希值（`TryGetHashInTerm`）。这是一个重要的缓存优化。
    2.  如果获取失败，则调用 `CalcHashKey`。
    3.  `CalcHashKey` 会检查是否存在 `_layerHasher`，如果存在则使用它；否则，回退到调用 `KeyHasherWrapper` 的静态方法。
    4.  计算出哈希值后，会将其写回到 `Term` 对象中（`term.SetTermHashKey(hashKey)`），以便下次直接使用。

这个“计算一次，到处使用”的模式，是 Indexlib 中常见的性能优化手段。

### 2.3. `SortValueConvertor.h`: 生成可比较的排序键

在很多场景下，需要根据字段的原始值进行排序。然而，不同数据类型（如 `int`, `float`, `string`）和不同排序方式（升序/降序）使得直接比较变得复杂。`SortValueConvertor` 的作用就是将各种类型的字段值，统一转换成**可直接按字节序比较**的字符串格式。

**实现机制**: 
*   它利用了 `autil` 库中的 `BinaryStringUtil`。该工具能将数值类型转换为保留其原始大小顺序的二进制字符串。
*   对于**升序（`sp_asc`）**，直接转换。
*   对于**降序（`sp_desc`）**，它会对转换后的二进制字符串按位取反（`toInvertString`）。这样，原本大的值在字符串比较时就会变小，从而巧妙地实现了降序排列。
*   `GenerateConvertor` 函数是一个工厂方法，它根据字段类型和排序模式（`sp_asc`/`sp_desc`），返回一个对应的 `std::function` 闭包。这使得上层代码可以持有一系列转换函数，对多重排序键进行统一处理。

### 2.4. `ShardPartitioner.h`: 分片策略

在分布式索引或分片索引的场景下，需要根据一个键（通常是主键）来决定一条文档应该被分配到哪个分片（Shard）。`ShardPartitioner` 负责这个逻辑。

**实现机制**: 
*   **初始化**: `Init` 方法接收 `shardCount`（必须是 2 的幂）和可选的哈希函数类型。
*   **计算**: `GetShardIdx` 方法首先使用 `indexlib::util::GetHashKey` 计算输入键的哈希值，然后通过一个简单的位运算 `hashValue & _shardMask` 来得到分片索引。`_shardMask` 的值是 `shardCount - 1`，这是一种最高效的取模运算，前提是 `shardCount` 必须是 2 的幂。

## 3. 总结与技术价值

Indexlib 的词元与哈希机制是一个设计精良、高度优化的系统。它通过分层表示、策略化哈希和功能组件的内聚，实现了功能、性能和灵活性的高度统一。

*   **性能**: `LiteTerm` 的使用、哈希值的缓存、基于位运算的分片和高效的排序键生成，无一不体现了对性能的极致追求。
*   **灵活性**: `DictHasher` 和 `KeyHasherWrapper` 的设计使得哈希策略可以轻松替换和扩展，以适应不同的业务需求。
*   **内聚性**: 将词元表示、哈希、排序、分片这些紧密相关的逻辑集中在 `index/common` 模块，形成了一个职责明确、易于理解和维护的功能单元。

这个系统是 Indexlib 实现高性能倒排索引的核心基础。理解其设计，特别是 `Term` 的分层表示和哈希计算的策略化与缓存机制，对于深入掌握 Indexlib 的工作原理至关重要。
