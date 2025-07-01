
# 变长属性格式化 (Variable-Length Attribute Formatting)

**涉及文件:**
*   `indexlib/index/normal/attribute/format/var_num_attribute_data_formatter.h`
*   `indexlib/index/normal/attribute/format/var_num_attribute_data_formatter.cpp`
*   `indexlib/index/normal/attribute/format/updatable_var_num_attribute_offset_formatter.h`
*   `indexlib/index/normal/attribute/format/updatable_var_num_attribute_offset_formatter.cpp`

## 1. 功能目标

变长属性（如 `string`、`multi-value integer` 等）是索引系统中不可或缺的一部分，用于存储长度不一的数据。与定长属性相比，其管理和存取更为复杂。本模块的核心目标是为这类属性设计一套高效、紧凑且支持更新的存储和访问方案。

具体功能目标包括：
1.  **紧凑的数据表示**：变长数据的核心挑战之一是如何在不浪费空间的前提下，记录每个值的长度。本模块需要实现一种紧凑的长度编码方案（如 `Varint` 编码），并将其与数据本身高效地组织在一起。
2.  **高效的数据访问**：即使数据是变长的，查询时也需要能够快速定位到任意 `docId` 对应的数据。这意味着需要一个高效的 `offset`（偏移量）管理机制。
3.  **支持在线更新**：在线服务中，对变长属性的更新（如修改一个用户的标签）是常见需求。由于新值的长度可能与旧值不同，原地更新（in-place update）通常不可行。因此，需要设计一种能够支持变长数据更新的机制，同时最小化更新带来的空间和性能开销。
4.  **支持多种变长类型**：方案需要足够通用，既能支持单个字符串（`string`），也能支持多值数值类型（`multi-int`），还能支持最复杂的多值字符串（`multi-string`）。

## 2. 系统架构与设计动机

本模块的架构围绕着“数据”和“偏移”两个核心概念构建，并通过精巧的编码方案将它们联系起来，同时解决了更新难题。

### 2.1. 核心组件

1.  **`VarNumAttributeDataFormatter` (数据格式化器)**
    *   **角色**：定义了变长属性数据文件（`data` 文件）的二进制布局。它负责编码和解码值的个数、每个值的具体内容，是变长数据存储的核心。
    *   **设计动机**：将复杂的数据布局逻辑封装起来。不同的变长类型（`string`, `multi-value`, `multi-string`）其布局有细微差别，`VarNumAttributeDataFormatter` 通过内部状态（如 `mIsMultiString`）来处理这些差异，对上层提供统一的 `GetDataLength` 接口。这使得上层逻辑无需关心底层布局的复杂性。

2.  **`common::VarNumAttributeFormatter` (底层编码工具)**
    *   **角色**：这是一个位于 `common` 模块的工具类，提供了变长整数（`Varint`）编码（`EncodeCount`）和解码（`DecodeCount`）的核心静态方法。
    *   **设计动机**：`Varint` 编码是一种用变动字节数表示整数的方法，数值越小，占用的字节数越少。这对于存储大量小整数（如数组长度）非常有效，是许多序列化系统（如 Protobuf）的基石。将其作为通用工具，可以在系统的多个地方复用。

3.  **`UpdatableVarNumAttributeOffsetFormatter` (可更新偏移格式化器)**
    *   **角色**：这是解决变长属性更新问题的关键组件。它负责管理 `offset` 文件，并定义了一种特殊的 `offset` 编码方案。
    *   **设计动机**：变长数据更新时，新值无法放在原位。一个常见的解决方案是将其追加到数据文件的末尾。但此时，`offset` 文件中就需要记录一个指向新位置的偏移量。为了区分这个偏移量是指向原始数据区还是追加数据区，`UpdatableVarNumAttributeOffsetFormatter` 引入了一种巧妙的编码方案，从而使得系统可以无缝地处理这两种情况。

### 2.2. 整体工作流程与数据布局

一个完整的变长属性通常由三个文件组成：`offset`、`data` 和 `slice_data`（用于更新）。

*   **`offset` 文件**：定长文件，存储每个 `docId` 对应的数据在 `data` 文件中的起始偏移量。`offset[docId]` 的值给出了 `docId` 数据的起点。
*   **`data` 文件**：变长文件，紧凑地存储所有文档的实际数据。
*   **`slice_data` 文件**（可选，用于更新）：一个分片的、仅追加的存储区域，用于存放更新后的新值。

**读取流程 (Read Path):**
1.  `AttributeReader` 根据 `docId` 读取 `offset` 文件，得到 `offset_value`。
2.  `UpdatableVarNumAttributeOffsetFormatter` 检查 `offset_value`：
    *   如果 `offset_value < mDataSize` (原始数据文件大小)，说明数据在 `data` 文件中，直接使用该 `offset_value`。
    *   如果 `offset_value >= mDataSize`，说明数据在 `slice_data` 中。通过 `DecodeToSliceArrayOffset` 将其解码为在 `slice_data` 中的真实偏移量。
3.  根据计算出的偏移量，从 `data` 文件或 `slice_data` 中读取数据。
4.  `VarNumAttributeDataFormatter` 解析读取到的二进制流，解码出值的个数和内容。

**更新流程 (Update Path):**
1.  `AttributeUpdater` 接收到对 `docId` 的更新请求，新值为 `new_value`。
2.  将 `new_value` 序列化后，追加写入到 `slice_data` 文件中，并获得其在 `slice_data` 中的偏移量 `slice_offset`。
3.  `UpdatableVarNumAttributeOffsetFormatter` 将 `slice_offset` 编码成一个新的 `offset` 值：`encoded_offset = EncodeSliceArrayOffset(slice_offset)`。
4.  用这个 `encoded_offset` 更新 `offset` 文件中 `docId` 对应的条目。

## 3. 核心逻辑与算法

### 3.1. `VarNumAttributeDataFormatter` 的数据布局

这是变长数据存储的核心，我们以最复杂的多值字符串（`multi-string`）为例：

**一个 `multi-string` 字段的二进制布局:**
```
+------------+----------+-------------------+-------------------+
| Count (Varint) | OffsetSize (1B) | Offsets Array     |  Data Area        |
+------------+----------+-------------------+-------------------+
| (个数)     | (偏移类型) | [off1, off2, ...] | [str1, str2, ...] |
+------------+----------+-------------------+-------------------+
```

1.  **Count**: 变长编码的整数，表示有多少个字符串。
2.  **OffsetSize**: 1个字节，表示 `Offsets Array` 中每个偏移量占几个字节（1, 2, 或 4）。这允许根据字符串的总长度自适应地选择偏移量类型，节省空间。
3.  **Offsets Array**: 存储每个字符串在 `Data Area` 中的相对结束位置。
4.  **Data Area**: 依次存储每个字符串的内容，每个字符串自身也是 `[length(varint), content]` 的格式。

`GetDataLength` 方法就是根据这个布局来计算一个完整的 `multi-string` 字段占用的总字节数。它需要解码 `Count`，读取 `OffsetSize`，然后跳转到最后一个字符串的偏移量，再解码最后一个字符串的长度，最终将所有部分加起来。

### 3.2. `UpdatableVarNumAttributeOffsetFormatter` 的偏移编码

这是实现可更新的关键。其逻辑非常简单但极为有效。

*   **初始化**：在加载时，用 `data` 文件的总大小 `mDataSize` 来初始化 `UpdatableVarNumAttributeOffsetFormatter`。

*   **`EncodeSliceArrayOffset`**:
    ```cpp
    inline uint64_t UpdatableVarNumAttributeOffsetFormatter::EncodeSliceArrayOffset(uint64_t originalOffset) const
    {
        // 将在 slice_data 中的偏移量加上一个巨大的基址（mDataSize）
        return originalOffset + mDataSize;
    }
    ```

*   **`IsSliceArrayOffset`**:
    ```cpp
    inline bool UpdatableVarNumAttributeOffsetFormatter::IsSliceArrayOffset(uint64_t offset) const
    {
        // 通过比较大小，就能判断偏移量指向哪个文件
        return offset >= mDataSize;
    }
    ```

*   **`DecodeToSliceArrayOffset`**:
    ```cpp
    inline uint64_t UpdatableVarNumAttributeOffsetFormatter::DecodeToSliceArrayOffset(uint64_t offset) const
    {
        // 减去基址，还原为在 slice_data 中的真实偏移量
        return offset - mDataSize;
    }
    ```

这个简单的加法/减法技巧，优雅地解决了区分两种不同来源的偏移量的问题，避免了使用额外的标记位或复杂的逻辑。

## 4. 关键实现细节

### `var_num_attribute_data_formatter.h`

```cpp
class VarNumAttributeDataFormatter
{
public:
    // ...
    void Init(const config::AttributeConfigPtr& attrConfig);

    // 对外暴露的核心接口，计算一个变长字段的总长度
    uint32_t GetDataLength(const uint8_t* data, bool& isNull) const __ALWAYS_INLINE;

private:
    // 内部根据不同类型，分发到不同的实现
    uint32_t GetNormalAttrDataLength(const uint8_t* data, bool& isNull) const __ALWAYS_INLINE;
    uint32_t GetMultiStringAttrDataLength(const uint8_t* data, bool& isNull) const __ALWAYS_INLINE;

private:
    bool mIsMultiString;      // 是否为 multi-string 类型
    uint32_t mFieldSizeShift; // 用于计算 multi-value 数值类型的长度 (count << shift)
    int32_t mFixedValueCount; // 是否是定长多值类型
    // ...
};
```
`Init` 方法根据 `AttributeConfig` 初始化内部状态，`GetDataLength` 则像一个路由器，根据这些状态调用不同的私有实现。`mFieldSizeShift` 是一个优化，对于 `multi-int32` 这样的类型，其数据区总长度就是 `count * 4`，即 `count << 2`，通过位移运算代替乘法可以提升性能。

### `updatable_var_num_attribute_offset_formatter.h`

```cpp
class UpdatableVarNumAttributeOffsetFormatter
{
public:
    UpdatableVarNumAttributeOffsetFormatter();
    ~UpdatableVarNumAttributeOffsetFormatter();

public:
    void Init(uint64_t dataSize);

    // 判断偏移量是否指向 slice array
    bool IsSliceArrayOffset(uint64_t offset) const __ALWAYS_INLINE;
    // 将 slice array 的偏移量编码
    uint64_t EncodeSliceArrayOffset(uint64_t offset) const __ALWAYS_INLINE;
    // 将编码后的偏移量解码回 slice array 的偏移量
    uint64_t DecodeToSliceArrayOffset(uint64_t offset) const __ALWAYS_INLINE;

private:
    uint64_t mDataSize; // 核心状态：原始 data 文件的大小

private:
    IE_LOG_DECLARE();
};
```
这个类的实现非常简洁，所有核心方法都被声明为 `__ALWAYS_INLINE`，因为它们的逻辑非常简单，内联可以消除函数调用开销，使得这部分逻辑在最终的二进制代码中几乎是“零成本”的。

## 5. 技术风险与未来展望

*   **技术风险**：
    *   **碎片化问题**：频繁的更新会导致 `slice_data` 中存在大量不再被任何 `offset` 引用的“垃圾”数据。同时，`data` 文件中被更新掉的旧数据也成为碎片。这些碎片会持续占用磁盘空间，直到下一次全量合并（merge）时才被回收。如果合并间隔太长，空间浪费会很严重。
    *   **Offset 文件写放大**：每次更新一个变长属性，都需要对 `offset` 文件进行一次随机写。如果更新量很大，`offset` 文件的随机 I/O 会成为瓶颈。
    *   **多值字符串的复杂性**：`multi-string` 的嵌套变长结构（外层是变长数组，内层每个 string 又是变长的）使其解析逻辑相对复杂，计算 `GetDataLength` 需要多次内存跳转，可能影响缓存友好性。

*   **未来展望**：
    *   **日志结构存储 (LSM-Tree)**：当前的更新模型可以看作是简化版的日志结构存储。未来可以借鉴 LSM-Tree 的思想，引入更完善的 MemTable 和多层 SSTable（对应这里的 `slice_data`）结构。更新首先写入内存中的 MemTable，达到阈值后刷到磁盘成为一个新的、有序的 `slice_data` 层。查询时从新到旧逐层查找，合并时也逐层进行。这可以使更新和空间回收更加高效和有组织。
    *   **数据与偏移分离存储**：对于超长字符串，可以考虑将其存储在一个单独的大对象存储（Blob Store）中，`data` 文件只存储其引用（如一个 hash 值或 ID）。这可以避免超长字段撑爆 `data` 文件，影响其他短字段的访问效率。
    *   **更激进的压缩**：可以对 `data` 文件中的数据块应用通用的压缩算法（如 ZSTD、LZ4），尤其是在数据局部性较好时，能获得很高的压缩率。这需要在读取时付出解压的 CPU 开销，是一种空间与时间的权衡。

总的来说，变长属性格式化模块通过“数据/偏移分离”和“基地址编码”等经典技术，成功地在紧凑存储、快速访问和在线更新这几个相互冲突的目标之间取得了巧妙的平衡。它是 Indexlib 能够灵活支持各种复杂数据类型，并保持高性能和准实时能力的关键所在。
