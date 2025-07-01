
# 定长单值属性补丁与空值处理

**涉及文件:**
*   `indexlib/index/normal/attribute/format/single_value_attribute_patch_formatter.h`
*   `indexlib/index/normal/attribute/format/single_value_attribute_patch_formatter.cpp`
*   `indexlib/index/normal/attribute/format/single_value_null_attr_formatter.h`
*   `indexlib/index/normal/attribute/format/single_value_null_attr_formatter.cpp`

## 1. 功能目标

在复杂的搜索引擎和数据分析场景中，数据不是一成不变的。本模块专注于解决两个在数据生命周期中至关重要的问题：**数据更新**和**数据缺失**。

1.  **数据更新 (Patching)**：当需要修改少量文档的属性值时，重写整个庞大的属性数据文件是极其低效的。因此，需要一种“打补丁”的机制，将变更信息记录在一个独立的补丁文件（patch file）中。在查询时，系统合并基础数据和补丁数据，提供最终一致的视图。本模块的核心目标就是定义和实现这个补丁文件的格式，使其既紧凑又高效。

2.  **数据缺失 (Null Handling)**：在现实世界的数据中，字段缺失是常态。数据库系统需要一种明确的方式来表示“空值”（NULL），以区别于“0”、“空字符串”等有效值。本模块的另一个核心目标是为定长单值属性设计一种空间高效的空值表示和存取方案，并确保其与普通值的读写逻辑无缝集成。

## 2. 系统架构与设计动机

本模块由两个相对独立但目标一致（处理特殊数据状态）的子系统构成：补丁格式化器和空值格式化器。

### 2.1. `SingleValueAttributePatchFormatter` (补丁格式化器)

*   **角色**：定义了单值属性补丁文件的二进制布局，并提供读写该布局的工具方法。
*   **设计动机**：
    *   **性能**：为了避免昂贵的“读-修改-写”操作，采用仅追加（Append-Only）的方式生成补丁文件。这种方式写入开销极小。
    *   **紧凑性**：补丁文件通常包含大量的小记录，因此其格式必须非常紧凑以节省存储空间。每个补丁项仅包含 `docid` 和新的 `value`。
    *   **可合并性**：补丁文件最终会在合并（merge）阶段被应用到全量数据中。其格式需要易于解析，以便高效地进行合并操作。

### 2.2. `SingleValueNullAttrFormatter<T>` (空值格式化器)

*   **角色**：专门负责处理支持 `NULL` 的定长单值属性的读写逻辑。它并非一个独立的 `AttributeFormatter`，而是作为 `SingleValueAttributeFormatter` 的一个内部组件，在其 `mSupportNull` 为 `true` 时被激活和使用。
*   **设计动机**：
    *   **空间效率**：如果为每个文档都增加一个 `bool` 标记来表示是否为 `NULL`，会浪费大量空间。采用位图（Bitmap）是解决这个问题的经典方案。每64个文档共享一个 `uint64_t` 的位图，将表示空值的空间开销降低到原来的 1/64。
    *   **逻辑复用与隔离**：将空值处理逻辑封装在 `SingleValueNullAttrFormatter` 中，而不是直接散布在 `SingleValueAttributeFormatter` 的代码里，使得主逻辑更清晰。当 `mSupportNull` 为 `false` 时，这部分逻辑完全不参与，避免了不必要的性能开销。这体现了良好的内聚和关注点分离。
    *   **特殊值编码**：为了快速判断，除了位图，数据本身也会被设置为一个预定义的“哨兵值”（sentinel value），即该类型的最小值（如 `numeric_limits<int32_t>::min()`）。查询时，只有当值等于哨兵值时，才需要去查位图确认是否真的为 `NULL`，这是一种有效的快速路径优化。

### 2.3. 架构整合

*   **补丁流程**：`AttributeUpdater` 在接收到更新请求时，会使用 `SingleValueAttributePatchFormatter` 将 `(docid, new_value)` 或 `(docid, is_null)` 写入到段（segment）内的 `attribute_patch` 文件中。查询时，`AttributeReader` 会加载这些补丁文件，并优先从补丁中查找数据。
*   **空值流程**：`SingleValueAttributeFormatter` 在初始化时，如果 `supportNull` 为 `true`，则会创建并初始化一个 `SingleValueNullAttrFormatter` 成员。之后所有 `Get` 和 `Set` 操作都会委托给这个成员来处理。数据文件中，数据和位图会交错存储。

## 3. 核心逻辑与算法

### 3.1. `SingleValueAttributePatchFormatter` 的核心逻辑

*   **补丁项格式**：一个补丁项由 `docid_t` 和 `T value` 组成。即 `[docid, value]`。

*   **空值的特殊处理**：如何在一个补丁文件中同时表示“将值更新为X”和“将值更新为NULL”？`SingleValueAttributePatchFormatter` 采用了一种巧妙的编码技巧：
    *   **正常值**：`docid` 保持原样（正数）。
    *   **NULL 值**：将 `docid` 按位取反（`~docid`），使其变为一个负数。此时，补丁项中不再需要存储 `value` 部分。

    ```cpp
    // 写 NULL 值的逻辑
    inline void Write(docid_t docId, uint8_t* value, uint8_t length)
    {
        mOutput->Write(&docId, sizeof(docid_t)).GetOrThrow();
        if (docId >= 0) // 如果 docId 是正数，说明不是 NULL，才写入 value
        {
            mOutput->Write(value, length).GetOrThrow();
        }
        mPatchItemCount++;
    }
    ```

*   **文件末尾的元数据**：为了在读取时能快速知道补丁文件中有多少项，`SingleValueAttributePatchFormatter` 会在文件末尾写入一个 `uint32_t` 来记录总的补丁项数量 (`mPatchItemCount`)。这避免了在加载时需要完整扫描一遍文件来计数。

    ```cpp
    void SingleValueAttributePatchFormatter::Close()
    {
        if (mOutput) {
            if (mIsSupportNull) {
                // 在关闭文件前，写入总数
                mOutput->Write(&mPatchItemCount, sizeof(uint32_t)).GetOrThrow();
            }
            mOutput->Close().GetOrThrow();
        }
        // ...
    }
    ```

### 3.2. `SingleValueNullAttrFormatter<T>` 的核心逻辑

*   **数据布局**：这是该模块最核心的设计。数据文件不再是简单的 `[value0, value1, ...]` 数组。而是被分成了多个组（Group），每个组包含64个文档的数据。

    **一个 Group 的布局:**
    ```
    +----------------+----------------------------------------------------+
    |  Bitmap (8 B)  |  Value Data (64 * recordSize B)                    |
    +----------------+----------------------------------------------------+
    |  (uint64_t)    |  [value_0, value_1, ..., value_63]                 |
    +----------------+----------------------------------------------------+
    ```

    *   `Bitmap`: 一个64位的整数。如果第 `i` 位是 `1`，表示这个组内第 `i` 个文档的属性值为 `NULL`。
    *   `Value Data`: 连续存储64个文档的属性值。

*   **`Get` 方法的算法**:
    1.  **计算组号和组内偏移**：
        ```cpp
        int64_t groupId = docId / SingleEncodedNullValue::NULL_FIELD_BITMAP_SIZE; // docId / 64
        int inGroupOffset = docId % SingleEncodedNullValue::NULL_FIELD_BITMAP_SIZE; // docId % 64
        ```
    2.  **计算数据地址**：根据 `groupId` 和 `docId` 计算出目标值在整个数据块中的绝对地址。
        ```cpp
        // 伪代码
        uint8_t* groupBaseAddr = data + groupId * (8 + 64 * recordSize);
        uint8_t* valueAddr = groupBaseAddr + 8 + docId * recordSize;
        ```
        `CALC_DOC_OFFSET` 宏封装了这个计算。
    3.  **读取值并进行快速判断**：
        ```cpp
        value = *(T*)valueAddr;
        if (value != mEncodedNullValue) { // mEncodedNullValue 是类型的最小值
            isNull = false; // 快速路径：值不是哨兵值，肯定不是 NULL
            return;
        }
        ```
    4.  **慢速路径：查位图**：如果值等于哨兵值，则必须查询位图来最终确认。
        ```cpp
        uint64_t* bitmapAddr = (uint64_t*)groupBaseAddr;
        isNull = (*bitmapAddr) & (1UL << inGroupOffset);
        ```
        `CHECK_FIELD_IS_NULL` 宏封装了这个逻辑。

*   **`Set` 方法的算法**:
    1.  计算 `groupId` 和 `valueAddr` (同 `Get` 方法)。
    2.  **如果要设置为 `NULL`**:
        a.  将 `valueAddr` 的内容设置为哨兵值 `mEncodedNullValue`。
        b.  更新位图，将对应的位置 `1` (`*bitmapAddr |= (1UL << inGroupOffset)`)。
    3.  **如果要设置为非 `NULL`**:
        a.  将 `valueAddr` 的内容设置为给定的 `value`。
        b.  更新位图，将对应的位置 `0` (`*bitmapAddr &= ~(1UL << inGroupOffset)`)。

## 4. 关键实现细节

### `single_value_attribute_patch_formatter.h`

```cpp
class SingleValueAttributePatchFormatter
{
public:
    // ...
    // 核心写接口
    inline void Write(docid_t docId, uint8_t* value, uint8_t length);

    // 核心读接口
    template <typename T>
    inline bool Read(docid_t& docId, T& value, bool& isNull);

    // 用于在合并时比较 docid（因为有正有负）
    inline static docid_t CompareDocId(const docid_t& left, const docid_t& right);

    // 将 docid 编码为负数表示 null
    inline static void EncodedDocId(docid_t& docId) { docId = ~docId; }
    // ...
private:
    bool mIsSupportNull;
    file_system::FileWriterPtr mOutput;
    file_system::FileReaderPtr mInput;
    uint64_t mCursor; // 读取时的游标
    uint64_t mPatchItemCount; // 写入时计数
    // ...
};
```

### `single_value_null_attr_formatter.h`

```cpp
// 定义了各种类型的哨兵值
class SingleEncodedNullValue
{
public:
    template <typename T>
    inline static void GetEncodedValue(void* ret);
    // ...
    static uint32_t NULL_FIELD_BITMAP_SIZE; // = 64
};

template <typename T>
class SingleValueNullAttrFormatter
{
    // ...
private:
    // 核心 Get/Set 逻辑
    void Get(docid_t docId, uint8_t* data, T& value, bool& isNull) const;
    void Set(docid_t docId, uint8_t* data, const T& value, bool isNull);

    // ...
private:
    uint32_t mRecordSize; // 单个记录大小
    uint32_t mGroupSize;  // 一个组的大小 (bitmap + 64 * recordSize)
    T mEncodedNullValue;  // 哨兵值
    // ...
};

// 大量使用宏来简化和统一地址计算与位操作
#define CALC_DOC_OFFSET(baseAddr, groupId, docId) ...
#define CHECK_FIELD_IS_NULL(baseAddr, groupId, docId, isNull) ...
#define SET_NULL_VALUE(baseAddr, groupId, docId) ...
```

*   **宏的使用**：代码中大量使用了宏定义。优点是简化了重复的、复杂的地址计算和位操作代码，使得 `Get`/`Set` 的核心逻辑更突出。缺点是降低了代码的可读性和可调试性，且容易出错。在现代 C++ 中，更推荐使用 `inline` 函数来替代这些宏。

## 5. 技术风险与未来展望

*   **技术风险**：
    *   **补丁文件膨胀**：如果一个字段被频繁更新，会导致补丁文件链条过长，包含大量冗余更新。这会增加段的加载时间和查询时的合并开销。需要依赖及时的索引合并（merge）来清理和压缩补丁。
    *   **哨兵值冲突**：使用类型的最小/最大值作为哨兵值是一种常见做法，但它假设了用户数据不会真正使用到这个边界值。如果用户的有效数据恰好就是这个哨兵值，系统将无法区分它是有效值还是 `NULL` 的标记。这是一个潜在的数据正确性风险。
    *   **数据对齐**：空值格式化器的 `mGroupSize` 几乎总是奇数个字节（例如 `8 + 64 * 4 = 264`），这可能导致跨缓存行（cache line）读取，对性能有轻微影响。

*   **未来展望**：
    *   **更通用的空值表示**：当前的空值方案与定长单值格式紧密耦合。可以设计一个更通用的、可插拔的空值存储策略。例如，将所有属性的空值位图集中存储在一个单独的文件中，而不是与各自的数据交错。这可能提高缓存效率，并简化不同类型属性的空值处理。
    *   **Delta 补丁**：对于数值类型，除了记录新值，还可以记录增量（delta）。例如，`+10` 或 `-5`。这对于累加、累减类的更新场景，可以节省空间并可能支持更丰富的原子操作。
    *   **补丁索引**：当一个段的补丁文件非常大时，在查询时线性扫描补丁文件会变慢。可以为补丁文件本身建立一个小的内存索引（如哈希表），实现 `docid -> patch_offset` 的快速定位，加速补丁数据的查找。

总的来说，该模块通过巧妙的编码和数据布局，以较低的成本解决了数据更新和缺失这两个普遍存在的数据管理难题。其设计在空间效率、时间和实现复杂度之间取得了很好的平衡，是 Indexlib 能够支持准实时更新和处理不完美数据的重要基础。
