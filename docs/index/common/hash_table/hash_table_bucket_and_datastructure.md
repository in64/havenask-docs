
# Indexlib 哈希表存储桶与数据结构深度解析

**涉及文件:**
*   `index/common/hash_table/SpecialKeyBucket.h`
*   `index/common/hash_table/CompactSpecialKeyBucket.h`
*   `index/common/hash_table/SpecialValueBucket.h`
*   `index/common/hash_table/SpecialValue.h`
*   `index/common/hash_table/HashTableNode.h`
*   `index/common/hash_table/BucketCompressor.h`
*   `index/common/hash_table/BucketOffsetCompressor.h`

## 1. 系统概述

在哈希表的宏伟架构之下，是其最基础、最关键的构成单元——**存储桶（Bucket）**。Bucket 的设计直接决定了哈希表的内存布局、功能上限和性能表现。为了在不同场景下实现最优的平衡，Indexlib 设计了多种 `Bucket` 实现和特殊的数据结构，以支持逻辑删除、时间戳、紧凑存储和数据压缩等高级功能。

本文档将深入到哈希表的“微观世界”，剖析这些基础数据结构的设计思想：

1.  **开放寻址法中的 Bucket**: 重点分析 `SpecialKeyBucket` 和 `SpecialValueBucket`，揭示它们如何通过不同的策略在 Bucket 内部编码“空”（Empty）和“已删除”（Deleted）状态。
2.  **特殊值类型 (`SpecialValue.h`)**: 探讨 `TimestampValue` 和 `OffsetValue` 等封装类，理解它们如何将业务逻辑（如时间戳、删除标记）与值本身融合，并与 `SpecialValueBucket` 协同工作。
3.  **拉链法中的节点 (`HashTableNode.h`)**: 分析拉链法中链表节点的基础结构。
4.  **空间优化**: 介绍 `CompactSpecialKeyBucket` 如何通过牺牲部分功能来换取更紧凑的内存布局，以及 `BucketCompressor` 如何在持久化时对数据进行压缩。

理解这些底层数据结构，是领会 Indexlib 哈希表如何在有限的内存空间内实现丰富功能的关键。

---

## 2. Bucket：开放寻址法的基石

在开放寻址法（如 Dense 和 Cuckoo 哈希）中，所有数据都存储在一个连续的 Bucket 数组中。因此，每个 Bucket 必须能够独立地表达自己的状态：是**空的（Empty）**，还是**被占用的（Occupied）**，又或者是**被逻辑删除的（Deleted）**。

Indexlib 提供了两种主要的策略来实现这一点。

### 2.1. 策略一：特殊键 `SpecialKeyBucket`

`SpecialKeyBucket` 是最经典的设计。它通过预留两个特殊的键值来标记“空”和“已删除”状态。

**核心思想**：
*   选取两个在正常业务中永远不会出现的 `_KT`（KeyType）值作为“哨兵”。
*   `_EmptyKey` (默认为 `0xFFFFFFFFFFFFFEFFUL`): 代表这个 Bucket 是空的。
*   `_DeleteKey` (默认为 `0xFFFFFFFFFFFFFDFFUL`): 代表这个 Bucket 中的数据已被逻辑删除。

**数据布局与逻辑**:

```cpp
#pragma pack(push)
#pragma pack(4)
template <typename _KT, typename _VT, _KT _EmptyKey, _KT _DeleteKey>
class SpecialKeyBucket
{
    // ...
private:
    _KT _key;
    union {
        _VT _value;
        _KT _keyInValue; // When deleted, value field stores the original key
    };
};
#pragma pack(pop)
```

*   **正常状态**: `_key` 存储用户的键，`_value` 存储用户的值。
*   **空状态**: `_key` 被设置为 `_EmptyKey`。
*   **删除状态**: `_key` 被设置为 `_DeleteKey`。这是一个巧妙的设计：当一个键被删除时，探测链不能被破坏。因此，我们仍然需要知道这个位置“曾经”是哪个键，以便在查找其他键时能继续探测。但如果只将 `_key` 设为 `_DeleteKey`，原始键信息就丢失了。所以，它利用了一个 `union`，在删除时，将原本存储 `_value` 的空间用来保存原始的键 `_keyInValue`。这样，在 `IsEqual()` 判断时，如果发现 `_key == _DeleteKey`，就会去比较 `_keyInValue`，保证了探测链的正确性。

**优点**：逻辑清晰，对于任何 `ValueType` 都通用。
**缺点**：
1.  需要一个 `union` 和额外的逻辑来保存被删除的键，略显复杂。
2.  `KeyType` 必须是整数类型，且需要预留出两个特殊值。

### 2.2. 策略二：特殊值 `SpecialValueBucket`

`SpecialValueBucket` 采用了另一种思路：它假设 `KeyType` 是普通的，没有任何特殊值，而是要求 `ValueType` 自身能够表达“空”和“已删除”的状态。

**核心思想**：将状态信息编码到 `ValueType` 内部。

**数据布局与逻辑**:

```cpp
#pragma pack(push)
#pragma pack(4)
template <typename _KT, typename _VT> // _VT must be a SpecialValue type
class SpecialValueBucket
{
public:
    // ...
    bool IsEmpty() const { return _value.IsEmpty(); }
    bool IsDeleted() const { return _value.IsDeleted(); }
    // ...
private:
    _KT _key;
    _VT _value;
};
#pragma pack(pop)
```

`SpecialValueBucket` 的结构非常简单，就是一个 `_key` 加一个 `_value`。它将所有状态判断的责任都委托给了 `_value` 对象。`_value` 必须是一个实现了 `IsEmpty()` 和 `IsDeleted()` 方法的特殊值类型。这种设计将键和值/状态的关注点完全分离。

---

## 3. 特殊值类型 (`SpecialValue.h`)

`SpecialValue.h` 文件是 `SpecialValueBucket` 策略的核心支撑。它定义了一系列封装类，这些类将业务逻辑（如时间戳）和状态信息（空/删除）包装在一起，形成一个自包含的 `ValueType`。

### 3.1. `TimestampValue`

这是为 KV 索引设计的最重要的特殊值类型，它同时编码了**时间戳（Timestamp）**、**删除标记（Deletion Flag）**和**真实值（Real Value）**。

**核心思想**：利用一个 32 位整数 `_timestamp` 的所有比特位来编码信息。

**数据布局与逻辑**:

```cpp
#pragma pack(push)
#pragma pack(4)
template <typename _RealVT>
class TimestampValue
{
private:
    // Empty: 0x7FFFFFFF, Valid: 0-------, Deleted: 1-------
    static const uint32_t Empty = 0x7FFFFFFF;
    static const uint32_t DeleteMask = 0x80000000;
    static const uint32_t TimestampMask = 0x7FFFFFFF;

public:
    // ...
    bool IsEmpty() const { return _timestamp == Empty; }
    bool IsDeleted() const { return _timestamp & DeleteMask; }
    uint32_t Timestamp() const { return _timestamp & TimestampMask; }
    const _RealVT& Value() const { return _value; }
    // ...
private:
    uint32_t _timestamp;
    _RealVT _value;
};
#pragma pack(pop)
```

*   **空状态 (`IsEmpty`)**: `_timestamp` 被设置为一个特殊值 `0x7FFFFFFF`。
*   **删除状态 (`IsDeleted`)**: `_timestamp` 的最高位（符号位）被置为 1 (`DeleteMask`)。这是一个非常高效的判断。
*   **时间戳 (`Timestamp`)**: `_timestamp` 的低 31 位用于存储实际的时间戳值。
*   **真实值 (`Value`)**: `_value` 字段存储用户数据的真实值。

`TimestampValue` 与 `SpecialValueBucket` 完美结合，使得 Bucket 本身无需关心任何状态，只需存储键和一个 `TimestampValue` 对象即可。这种设计优雅且高效。

### 3.2. `OffsetValue` 和 `SpecialValue`

*   `OffsetValue`: 类似于 `TimestampValue`，但它只编码了“空”、“删除”和“值”三种状态，没有时间戳。它通常用于存储偏移量（offset）等场景，通过预留两个特殊的偏移量值来代表“空”和“删除”。
*   `SpecialValue`: 是 `OffsetValue` 的一个更通用的版本，允许用户通过模板参数自定义“空”和“删除”对应的哨兵值。

### 3.3. `ValueUnpacker`

由于 `TimestampValue` 等结构将多种信息打包在一起，上层用户在使用时可能只关心其中的一部分（比如真实值）。`ValueUnpacker` 提供了一个解包工具，可以将一个打包后的 `StringView` 解成 `timestamp` 和 `realValue` 两部分，方便上层调用。

---

## 4. 空间优化与压缩

### 4.1. `CompactSpecialKeyBucket`

这是 `SpecialKeyBucket` 的一个变种，旨在追求极致的内存紧凑。

**核心思想**：放弃对逻辑删除的支持，从而简化 Bucket 结构。

```cpp
template <typename _KT, typename _VT, _KT _EmptyKey, _KT _DeleteKey>
class CompactSpecialKeyBucket
{
    // ...
    // No union, no _keyInValue
private:
    _KT _key;
    _VT _value;
};
```

它只有一个 `_key` 和一个 `_value`，通过 `_key == _EmptyKey` 来判断是否为空。它不支持 `IsDeleted()`，或者说其 `IsDeleted()` 永远返回 `false`。这使得它的结构和 `SpecialValueBucket` 一样简单，但它依赖的是特殊键而非特殊值。

**适用场景**：在确定不需要逻辑删除，且 `ValueType` 很简单（不适合做成特殊值）的场景下，使用它可以节省 `union` 带来的潜在对齐开销，获得最紧凑的内存布局。

### 4.2. `BucketCompressor`

这是一个用于在持久化时压缩 Bucket 的抽象基类。它定义了一个 `Compress` 接口，接收输入 Bucket 的内存指针和大小，并输出压缩后的数据。

### 4.3. `BucketOffsetCompressor`

这是 `BucketCompressor` 的一个具体实现，专门用于压缩含有长偏移量（如 64 位 `offset_t`）的 Bucket。

**核心思想**：如果系统确信所有的偏移量都不会超过某个较小的值（如 `short_offset_t`，16位），那么在持久化时，可以将 64 位的偏移量强制转换为 16 位，从而大幅减少存储空间。

**关键代码 (`BucketOffsetCompressor.h`)**:
```cpp
template <typename _KT, typename _OFFSET>
class BucketOffsetCompressor<SpecialValueBucket<_KT, OffsetValue<_OFFSET>>> : public BucketCompressor
{
public:
    // ...
    size_t Compress(char* in, size_t bucketSize, char* out) override
    {
        typedef SpecialValueBucket<_KT, OffsetValue<_OFFSET>> LongBucket;
        typedef OffsetValue<short_offset_t> ShortValue;
        typedef SpecialValueBucket<_KT, ShortValue> ShortBucket;
        
        LongBucket* inBucket = static_cast<LongBucket*>((void*)in);
        ShortBucket outBucket;
        // ... logic to copy key and cast value ...
        outBucket.Set(inBucket->Key(), ShortValue(static_cast<short_offset_t>(inBucket->Value().Value())));
        
        *static_cast<ShortBucket*>((void*)out) = outBucket;
        return sizeof(ShortBucket);
    }
};
```
这个特化模板展示了其工作原理：读取一个 `LongBucket`，创建一个 `ShortBucket`，将 `LongBucket` 的 `_OFFSET` 类型的 `Value` 强制转换为 `short_offset_t`，然后将整个 `ShortBucket` 写入输出缓冲区。文件的尺寸也因此得以减小。

**风险**：这是一种有损压缩。如果数据构建过程中出现了大于 `short_offset_t` 能表示的最大值的偏移量，转换时会发生数据截断，导致索引数据错误。因此，这必须在确认数据范围的前提下才能安全使用。

## 5. 总结

Indexlib 的 Bucket 和基础数据结构设计，展现了在系统底层进行精细化优化的工程实践。通过组合不同的 Bucket 策略和特殊值类型，Indexlib 得以在统一的哈希表框架下，支持多样化的功能需求。

*   **状态编码**：通过“特殊键”和“特殊值”两种策略，优雅地解决了开放寻址法中“空/删除”状态的表示问题。
*   **逻辑封装**：`TimestampValue` 等类将业务逻辑（时间戳）与数据状态（删除）封装在一起，实现了高度内聚和信息隐藏。
*   **按需优化**：提供了 `Compact` 版本和 `Compressor` 等工具，允许开发者在特定场景下，通过牺牲通用性来换取极致的空间效率。

这些看似微小的底层设计，共同构成了 Indexlib 高性能、高功能、高效率哈希表模块的坚实地基。
