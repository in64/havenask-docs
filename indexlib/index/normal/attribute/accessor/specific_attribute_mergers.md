
# Indexlib 特定类型 Attribute Merger 深度解析

**涉及文件:**

*   `indexlib/index/normal/attribute/accessor/single_value_attribute_merger.h`
*   `indexlib/index/normal/attribute/accessor/fixed_value_attribute_merger.h`
*   `indexlib/index/normal/attribute/accessor/fixed_value_attribute_merger.cpp`
*   `indexlib/index/normal/attribute/accessor/var_num_attribute_merger.h`
*   `indexlib/index/normal/attribute/accessor/uniq_encoded_var_num_attribute_merger.h`
*   `indexlib/index/normal/attribute/accessor/pack_attribute_merger.h`
*   `indexlib/index/normal/attribute/accessor/pack_attribute_merger.cpp`
*   `indexlib/index/normal/attribute/accessor/uniq_encoded_pack_attribute_merger.h`
*   `indexlib/index/normal/attribute/accessor/uniq_encoded_pack_attribute_merger.cpp`
*   `indexlib/index/normal/attribute/accessor/docid_join_value_attribute_merger.h`
*   `indexlib/index/normal/attribute/accessor/docid_join_value_attribute_merger.cpp`
*   `indexlib/index/normal/attribute/accessor/primary_key_attribute_merger.h`
*   以及 `date`、`time`、`timestamp`、`string`、`location`、`line`、`polygon` 等类型的 `AttributeMerger` 头文件。

## 摘要

本文深入探讨 Indexlib 中针对特定数据类型的 Attribute Merger 的实现，详细阐述其如何基于通用的合并框架，为不同类型的数据（如定长数值、变长字符串、地理位置信息、Pack Attribute 等）提供高效、可靠的合并策略。Indexlib 通过模板元编程和继承机制，构建了一个层次分明、易于扩展的 Attribute Merger 体系。本文将重点分析几类具有代表性的 Merger 实现，包括 `SingleValueAttributeMerger`、`VarNumAttributeMerger`、`PackAttributeMerger` 等，揭示其内部工作原理和关键技术细节。

## 1. Attribute Merger 的层次化设计

为了应对多样化的数据类型，Indexlib 的 Attribute Merger 采用了层次化的设计。顶层是抽象基类 `AttributeMerger`，它定义了合并的通用接口。在其之下，根据数据的存储特性，衍生出两个核心的中间基类：

*   **`FixedValueAttributeMerger`**: 针对定长数据类型（如 `int`、`float`、`double` 等）的合并。这类数据的每个值都占用固定的存储空间。
*   **`VarNumAttributeMerger`**: 针对变长数据类型（如 `string`、多值数值等）的合并。这类数据的每个值占用的存储空间是可变的。

具体的 Attribute Merger，如 `SingleValueAttributeMerger<T>` 或 `StringAttributeMerger`，都继承自这两个中间基类之一，并根据自身特性进行特化实现。

## 2. `SingleValueAttributeMerger<T>`：定长单值数据的合并利器

`SingleValueAttributeMerger<T>` 是 `FixedValueAttributeMerger` 的一个模板子类，专门用于处理定长的单值 Attribute，例如 `int32`、`uint64`、`float` 等。它是 Indexlib 中使用最广泛的 Merger 之一。

### 2.1. 核心思想

其核心思想非常直观：逐一读取旧 Segment 中的文档，根据 `ReclaimMap` 找到其在新 Segment 中的 `newDocId`，然后将对应的 Attribute 值写入新 Segment 的数据文件中。由于数据是定长的，所以写入操作非常高效，可以直接通过 `docId` 计算出数据在文件中的偏移量。

### 2.2. 关键实现

*   **`MergeSegment`**: 这是 `SingleValueAttributeMerger` 的核心方法。它会遍历指定 Segment 中的所有文档，对于每个未被删除的文档，它会：
    1.  通过 `ReclaimMap` 获取 `newDocId`。
    2.  从 `SegmentReader` 中读取该文档的 Attribute 值。
    3.  将 Attribute 值写入到 `newDocId` 对应的目标 Segment 的 `OutputData` 中。
*   **`OutputData`**: `FixedValueAttributeMerger` 使用 `OutputData` 结构来管理输出。每个目标 Segment 都对应一个 `OutputData` 实例，其中包含了数据写入器（`dataAppender`）等信息。`dataAppender` 内部维护了一个缓冲区，当缓冲区写满后，会自动将数据刷入磁盘。
*   **支持压缩**: `SingleValueAttributeMerger` 支持对数据进行等值压缩（Equal Value Compress）。它通过 `EqualValueCompressDumper` 来实现，在数据写入时，如果连续出现多个相同的值，则会用一种紧凑的格式来表示，从而有效减少存储空间。

### 2.3. 关键代码示例：`SingleValueAttributeMerger.h`

```cpp
template <typename T>
inline void SingleValueAttributeMerger<T>::MergeSegment(const MergerResource& resource,
                                                        const index_base::SegmentMergeInfo& segMergeInfo,
                                                        const index_base::OutputSegmentMergeInfos& outputSegMergeInfos,
                                                        const config::AttributeConfigPtr& attrConfig)
{
    auto segReader = OpenSingleValueAttributeReader(attrConfig, segMergeInfo);
    auto reclaimMap = resource.reclaimMap;
    uint32_t docCount = segMergeInfo.segmentInfo.docCount;
    for (docid_t localId = 0; localId < (docid_t)docCount; ++localId)
    {
        docid_t newId = reclaimMap->GetNewId(localId + segMergeInfo.baseDocId);
        if (newId < 0) // is deleted
        {
            continue;
        }
        auto output = mSegOutputMapper.GetOutput(newId);
        if (!output) {
            continue;
        }

        T value {};
        bool isNull = false;
        segReader.reader->Read(localId, segReader.ctx, (uint8_t*)&value, sizeof(value), isNull);
        output->Set(newId, value, isNull);
        if (output->BufferFull()) {
            FlushDataBuffer(*output);
        }
    }
}
```

## 3. `VarNumAttributeMerger<T>`：应对变长数据的挑战

`VarNumAttributeMerger<T>` 用于处理变长数据，如字符串和多值数值。与定长数据不同，变长数据的合并要复杂得多，因为它需要同时处理数据文件和偏移文件。

### 3.1. 数据存储结构

变长 Attribute 通常由两部分组成：

*   **Data File**: 紧凑地存储所有文档的 Attribute 数据。
*   **Offset File**: 存储每个文档对应的数据在 Data File 中的偏移量。

### 3.2. 核心思想

`VarNumAttributeMerger` 的合并流程如下：

1.  **创建 `DocumentMergeInfoHeap`**: 这是一个最小堆，用于按 `newDocId` 的顺序依次返回需要合并的文档信息（`DocumentMergeInfo`）。
2.  **遍历堆**: 从堆中依次取出 `DocumentMergeInfo`。
3.  **读取旧数据**: 根据 `DocumentMergeInfo` 中的 `segmentIndex` 和 `oldDocId`，从对应的 `SegmentReader` 中读取原始的 Attribute 数据。
4.  **处理 Patch**: 如果存在 Patch 文件，则需要将 Patch 应用到原始数据上，得到最新的数据。
5.  **写入新数据**: 将最终的数据写入到目标 Segment 的 `VarLenDataWriter` 中。`VarLenDataWriter` 会自动处理数据和偏移的写入。

### 3.3. `UniqEncodedVarNumAttributeMerger<T>`：为去重而生

对于开启了“唯一值编码”（Uniq Encode）的变长 Attribute，Indexlib 提供了 `UniqEncodedVarNumAttributeMerger<T>`。它的核心思想是，在合并过程中，为每个唯一的 Attribute 值分配一个 ID，从而将对变长数据的操作转换为对定长 ID 的操作，以提高效率和减少存储。

*   **核心数据结构**: 它使用一个 `HashMap`（`EncodeMap`）来维护 Attribute 值（或其 Hash 值）到新 Offset 的映射。
*   **合并流程**: 在合并数据时，它会先在 `EncodeMap` 中查找当前 Attribute 值是否已经存在。如果存在，则直接复用其 Offset；如果不存在，则将该值写入 Data File，生成一个新的 Offset，并将其存入 `EncodeMap`。

## 4. `PackAttributeMerger`：处理打包的 Attribute

`PackAttributeMerger` 专门用于合并 Pack Attribute。Pack Attribute 是将多个 Attribute 打包存储在一起，以减少 IO 次数和存储开销。`PackAttributeMerger` 继承自 `VarNumAttributeMerger<char>`，因为它将整个 Pack Attribute 值视为一个变长的字符数组。

### 4.1. 核心挑战

Pack Attribute 的合并面临一个核心挑战：如何处理对其中部分子 Attribute 的更新（Patch）？

### 4.2. 解决方案

`PackAttributeMerger` 的解决方案是：

1.  **读取原始 Pack 数据**: 从旧 Segment 中读取完整的 Pack Attribute 数据。
2.  **读取 Patch 数据**: 使用 `PackAttributePatchReader` 读取针对该文档的所有 Patch。
3.  **合并 Patch**: 使用 `PackAttributeFormatter` 的 `MergeAndFormatUpdateFields` 方法，将 Patch 应用到原始数据上，生成一个新的、完整的 Pack Attribute 值。
4.  **写入新数据**: 将合并后的新值写入到目标 Segment。

`UniqEncodedPackAttributeMerger` 则在 `PackAttributeMerger` 的基础上，增加了对唯一值编码的支持，其原理与 `UniqEncodedVarNumAttributeMerger` 类似。

## 5. 特殊的 Merger：`DocidJoinValue` 与 `PrimaryKey`

*   **`DocidJoinValueAttributeMerger`**: 用于合并 Join 索引中的 `main_docid_to_sub_docid` 和 `sub_docid_to_main_docid` 这两个特殊的 Attribute。它的合并逻辑与 `ReclaimMap` 紧密耦合，直接利用 `ReclaimMap` 中已经计算好的 Join Value 信息来生成新的 Attribute 数据。
*   **`PrimaryKeyAttributeMerger`**: 用于合并主键（Primary Key）的 Attribute。它继承自 `SingleValueAttributeMerger`，但重写了 `OpenSingleValueAttributeReader` 和 `CreateDataFileWriter` 等方法，以从正确的主键索引目录中读取和写入数据。

## 6. 结论

Indexlib 通过一个设计精良的、层次化的继承体系，为各种特定类型的 Attribute 提供了专门的 Merger 实现。从处理简单定长值的 `SingleValueAttributeMerger`，到应对复杂变长数据的 `VarNumAttributeMerger`，再到为 Pack Attribute 和唯一值编码等高级特性量身定制的 `PackAttributeMerger` 和 `UniqEncoded...Merger`，每一层都清晰地封装了特定的逻辑。这种设计不仅保证了功能上的完备性，也为未来的扩展提供了极大的便利。深入理解这些特定类型 Merger 的实现细节，是掌握 Indexlib 高级特性、进行性能优化和二次开发的关键。
