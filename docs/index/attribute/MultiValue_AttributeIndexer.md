
# Indexlib 多值属性索引器实现深度解析

## 1. 引言

与单值属性不同，多值属性为一个文档存储一个或多个值，例如一篇文章的多个标签（tags）、一件商品的多个促销活动 ID 等。由于每个文档的属性值数量不固定，导致其底层存储是**变长（Variable-Length）**的。这种变长特性给数据的存储和读取带来了更大的挑战，也使得多值属性索引器的设计比单值索引器复杂得多。

本文将深入剖析 Indexlib 中多值属性索引器的核心实现，覆盖其从内存构建到磁盘服务的完整流程。我们将重点分析以下两个关键类：

*   **`MultiValueAttributeMemIndexer<T>`**: 负责在内存中构建多值属性索引。我们将探究其如何管理变长数据，以及其核心数据结构 `VarLenDataAccessor` 的设计。
*   **`MultiValueAttributeDiskIndexer<T>`**: 负责加载和查询磁盘上的多值属性数据。我们将分析其经典的“数据文件 + Offset 文件”存储结构，以及如何支持在线更新和碎片整理。

通过对这两个类的源码进行细致解读，我们将揭示 Indexlib 如何高效地处理变长数据，并理解其在存储效率、访问性能和在线维护能力之间的设计权衡。

## 2. 内存构建：`MultiValueAttributeMemIndexer<T>`

`MultiValueAttributeMemIndexer<T>` 继承自 `AttributeMemIndexer`，其核心职责是：**在内存中高效地存储和管理大量变长的多值数据。**

### 2.1. 核心数据结构与设计

多值属性索引器设计的最大挑战在于如何处理变长数据。如果为每个文档都独立分配内存，会产生大量的内存碎片，并且管理开销巨大。Indexlib 采用了一种集中的、类似于内存池的方案来解决这个问题，其核心实现是 `VarLenDataAccessor`。

`MultiValueAttributeMemIndexer` 将所有与数据存储相关的复杂逻辑都委托给了 `_accessor` 成员。

```cpp
// indexlib/index/attribute/MultiValueAttributeMemIndexer.h

template <typename T>
class MultiValueAttributeMemIndexer : public AttributeMemIndexer
{
    // ...
private:
    VarLenDataAccessor* _accessor;
    // ...
};
```

#### **`VarLenDataAccessor` 详解**

`VarLenDataAccessor` 是 Indexlib 中用于在内存中管理变长数据集合的核心组件。其内部主要由两部分构成：

1.  **数据区 (`_data`)**: 一个或多个连续的大内存块（通过 `TypedSliceList<char>` 管理），所有文档的变长数据（已经编码为二进制格式）被紧凑地追加（Append）到这个区域。
2.  **Offset 数组 (`_offsets`)**: 一个 `TypedSliceList<uint64_t>` 数组，其索引是 `docid_t`。`_offsets[docId]` 存储的是该文档对应的数据在数据区的起始偏移地址（Offset）。

![VarLenDataAccessor Architecture](https://mermaid.ink/svg/eyJjb2RlIjoiZ3JhcGggTFJcbiAgICBzdWJncmFwaCBcIlZhbHVlIEFyZWFcIlxuICAgICAgICB2YWx1ZXNcbiAgICBlbmRcblxuICAgIHN1YnNlY3Rpb24gT2Zmc2V0IEFycmF5XG4gICAgICAgIG9mZnNldHNcbiAgICBlbmRcblxuICAgIG9mZnNldHNbXCJkb2NJZFwiXSAtLT4gdmFsdWVzW1wiZGF0YVwiXVxuIiwibWVybWFpZCI6eyJ0aGVtZSI6ImRlZmF1bHQifSwidXBkYXRlRWRpdG9ciI6ZmFsc2V9)

当需要添加一个文档的属性值时：

*   数据被追加到数据区的末尾。
*   新的数据起始 Offset 被记录在 `_offsets` 数组的相应 `docId` 位置。

当需要读取一个文档的属性值时：

*   通过 `_offsets[docId]` 找到数据在数据区的起始地址。
*   从该地址开始解析数据。由于多值数据本身包含了长度信息（通常在数据的头部），因此可以知道需要读取多少字节。

这种设计将零散的变长数据整合到集中的存储区，极大地减少了内存碎片，提高了内存使用效率。

### 2.2. 关键实现分析

#### **添加字段 (`AddField`)**

```cpp
// indexlib/index/attribute/MultiValueAttributeMemIndexer.h

template <typename T>
inline void MultiValueAttributeMemIndexer<T>::AddField(
    docid_t docId, const autil::StringView& attributeValue, bool isNull)
{
    assert(_attrConvertor);
    // 1. 将字符串形式的输入值编码为二进制格式
    AttrValueMeta meta = isNull ? _attrConvertor->Decode(autil::StringView(_nullValue)) 
                                : _attrConvertor->Decode(attributeValue);
    // 2. 将编码后的二进制数据追加到 accessor
    _accessor->AppendValue(meta.data, meta.hashKey);
}
```

`AddField` 的逻辑清晰地展示了 `VarLenDataAccessor` 的用法：

1.  **编码**: 首先，`MultiValueAttributeConvertor` 将用户输入的、可读的字符串（如 `"tagA,tagB"`）转换成紧凑的二进制格式。这个二进制格式通常是 `[count][value1][value2]...` 的形式，其中 `count` 记录了值的数量。
2.  **追加**: 调用 `_accessor->AppendValue`，将编码后的二进制数据块追加到 `VarLenDataAccessor` 的数据区，并记录下新的 Offset。

#### **更新字段 (`UpdateField`)**

```cpp
// indexlib/index/attribute/MultiValueAttributeMemIndexer.h

template <typename T>
inline bool MultiValueAttributeMemIndexer<T>::UpdateField(
    docid_t docId, const autil::StringView& value, bool isNull, const uint64_t* hashKey)
{
    // ...
    // 直接调用 accessor 的更新接口
    return _accessor->UpdateValue(docId, value);
}
```

`UpdateField` 直接调用 `_accessor->UpdateValue`。在 `VarLenDataAccessor` 内部，更新操作通常也采用**追加写（Append-Only）**的策略：

1.  将新的值追加到数据区的末尾。
2.  更新 `_offsets[docId]`，使其指向新的数据位置。
3.  旧的数据空间不会被立即回收，而是成为“碎片”。

这种 Append-Only 的方式避免了原地更新可能导致的复杂空间管理问题（例如，新值比旧值长，需要移动后续数据），简化了设计，但代价是会产生内存碎片。这些碎片会在 `Dump` 之后随着内存池的释放而被统一回收。

#### **Dump 操作**

`Dump` 方法的核心是调用 `VarLenDataDumper`。这个 Dumper 会：

1.  **Dump 数据文件**: 将 `VarLenDataAccessor` 的整个数据区一次性写入磁盘，形成 `data` 文件。
2.  **Dump Offset 文件**: 遍历 `_offsets` 数组，将其内容写入磁盘，形成 `offset` 文件。如果配置了等效压缩，还会在此阶段对 Offset 进行压缩。
3.  **写入元信息**: 将数据的统计信息（如总数、最大长度等）写入 `ATTRIBUTE_DATA_INFO_FILE_NAME` 文件。

## 3. 磁盘服务：`MultiValueAttributeDiskIndexer<T>`

`MultiValueAttributeDiskIndexer<T>` 负责加载和查询磁盘上的多值属性数据。其设计是内存 `VarLenDataAccessor` 结构的磁盘化映射，同样遵循“数据 + Offset”的核心思想。

### 3.1. 核心数据结构与设计

`MultiValueAttributeDiskIndexer` 的核心是管理两个文件：

1.  **数据文件 (`data`)**: 存储所有文档的、紧凑排列的二进制属性数据。通常通过 `mmap` 或文件流（`FileStream`）的方式访问。
2.  **Offset 文件 (`offset`)**: 存储每个 `docId` 对应的数据在 `data` 文件中的起始偏移量。这是一个定长的 `uint64_t` 数组（或经过压缩），可以通过 `docId` 直接计算出其在文件中的位置，实现 O(1) 复杂度的 Offset 查找。

```cpp
// indexlib/index/attribute/MultiValueAttributeDiskIndexer.h

protected:
    uint8_t* _data; // mmap'ed data file base address
    std::shared_ptr<indexlib::file_system::FileStream> _fileStream; // data file stream
    MultiValueAttributeOffsetReader _offsetReader; // offset file reader
    // ...
```

### 3.2. 关键实现分析

#### **数据读取 (`Read`)**

`Read` 方法是整个类的核心，其执行流程如下：

```cpp
// indexlib/index/attribute/MultiValueAttributeDiskIndexer.h (Conceptual)

template <typename T>
inline const uint8_t* MultiValueAttributeDiskIndexer<T>::ReadData(docid_t docId, ReadContext& ctx) const
{
    // 1. 优先从 Patch 读取
    if (unlikely(_patch != nullptr)) {
        // ... patch read logic ...
    }

    // 2. 从 OffsetReader 获取数据偏移量
    auto [status, offset] = ctx.offsetReader.GetOffset(docId);
    if (!status.IsOK()) {
        return nullptr;
    }

    // 3. 根据 Offset 类型从不同地方读取数据
    const uint8_t* data = NULL;
    if (!_offsetFormatter.IsSliceArrayOffset(offset)) {
        // 3a. Offset 指向主数据文件
        if (_data == nullptr) { // FileStream 模式
            data = FetchValueFromStream(offset, ctx.fileStream, ctx.pool);
        } else { // mmap 模式
            data = _data + offset;
        }
    } else {
        // 3b. Offset 指向扩展的 Slice 文件（用于在线更新）
        assert(_defragSliceArray);
        data = (const uint8_t*)_defragSliceArray->Get(
            _offsetFormatter.DecodeToSliceArrayOffset(offset));
    }
    return data;
}
```

这个过程揭示了 `MultiValueAttributeDiskIndexer` 的几个关键设计点：

1.  **Patch 优先**: 与单值索引器一样，严格遵循 Patch 优先原则。
2.  **Offset 读取**: `_offsetReader` 封装了对 `offset` 文件的读取逻辑，包括可能的解压操作。
3.  **数据读取**: 获取到 `offset` 后，会判断 `offset` 的类型：
    *   **普通 Offset**: 指向主 `data` 文件。根据 `_data` 是否为 `nullptr`，决定是从 `mmap` 的内存地址直接计算指针，还是通过 `FileStream` 从磁盘读取数据到 `pool` 中。`mmap` 模式性能更高，但会占用虚拟内存地址空间；`FileStream` 模式更节省地址空间，但读取时有 IO 开销。
    *   **Slice Array Offset**: 这是一个特殊的 Offset，表示数据存储在用于在线更新的“扩展文件”中。

#### **在线更新与碎片整理 (`UpdateField`, `Defrag`)**

多值属性的在线更新是一个挑战，因为新值的长度通常与旧值不同，无法原地更新。Indexlib 采用了一个**扩展文件（Extend File）**的策略来解决这个问题。

*   **`UpdateField` 逻辑**: 当 `UpdateField` 被调用时：
    a.  新的属性值被写入一个名为 `ATTRIBUTE_DATA_EXTEND_SLICE_FILE_NAME` 的独立文件中。这个文件由 `MultiValueAttributeDefragSliceArray` 管理。
    b.  `UpdateField` 会从这个扩展文件中获取一个指向新数据的、特殊的 `SliceArrayOffset`。
    c.  然后，它会更新主 `offset` 文件中对应 `docId` 的条目，用这个新的 `SliceArrayOffset` 替换掉旧的 `offset`。
    d.  旧数据在主 `data` 文件中就成了“碎片”。
*   **`Defrag` 逻辑**: `MultiValueAttributeDefragSliceArray` 不仅负责写入新数据，还负责**碎片整理（Defragmentation）**。当一个 `offset` 被更新后，它持有的旧 `offset` 就指向了不再被使用的数据。`_defragSliceArray` 会记录这些可以被回收的空间。当扩展文件中的碎片达到一定比例（由 `defragSlicePercent` 配置）时，它可能会触发整理操作，重用这些碎片空间。

这种“主文件 + 扩展文件”的设计，巧妙地实现了变长数据的在线更新，同时通过碎片整理机制控制了存储空间的膨胀。

## 4. 结论

Indexlib 的多值属性索引器是其处理复杂数据场景能力的集中体现。面对变长数据带来的挑战，它通过一系列精巧的设计给出了高效的解决方案。

1.  **统一的变长数据管理 (`VarLenDataAccessor`)**: 在内存构建阶段，通过将数据和 Offset 分离存储，有效解决了内存碎片问题，并简化了管理。
2.  **经典的“数据 + Offset”磁盘结构**: 这是处理变长数据的行业标准实践。Indexlib 在此基础上，通过 `mmap` 和 `FileStream` 两种模式提供了灵活性，并通过 `OffsetReader` 封装了压缩等复杂逻辑。
3.  **创新的在线更新与碎片整理机制**: 通过引入“扩展文件” (`SliceArray`) 和特殊的 `SliceArrayOffset`，实现了变长数据的在线更新，并通过 `Defrag` 机制保证了长期的存储效率。
4.  **性能与资源的权衡**: 无论是 `mmap` vs `FileStream` 的选择，还是 Append-Only 更新策略，都体现了在实现高性能的同时，对系统资源（内存、磁盘IO）的精细化管理和权衡。

深入理解多值属性索引器的实现，不仅能帮助我们更好地使用 Indexlib，也能为我们设计其他需要处理变长数据的系统提供宝贵的经验和启示。
