
# 通用属性格式化 (General Attribute Formatting)

**涉及文件:**
*   `indexlib/index/normal/attribute/format/attribute_formatter.h`
*   `indexlib/index/normal/attribute/format/attribute_formatter.cpp`

## 1. 功能目标

通用属性格式化模块是 Indexlib 属性（Attribute）系统的基石。它的核心目标是定义一个统一的、可扩展的接口，用于管理不同类型属性数据的序列化和反序列化。无论属性是定长的、变长的、单值的还是多值的，都需要遵循这个接口进行数据的读取和写入。这套接口的设计旨在将上层属性读写逻辑与底层数据的物理存储格式解耦，从而实现高度的灵活性和可维护性。

具体来说，该模块要实现以下功能：
*   **定义通用接口**：提供一组纯虚函数，作为所有具体属性格式化器的契约。这些接口包括初始化、设置（写入）、重置（更新）和获取（读取）属性值等。
*   **数据转换**：集成 `AttributeConvertor`，支持将字符串形式的原始属性值转换为内部二进制格式，以优化存储和计算效率。
*   **支持可空属性**：在接口层面考虑对可空属性的支持，允许具体的格式化器实现对空值的特殊处理。

## 2. 系统架构与设计动机

`AttributeFormatter` 被设计为一个抽象基类（Abstract Base Class），这是一种典型的面向对象设计模式。其设计动机在于：

*   **统一接口，简化调用**：通过定义统一的接口，上层模块（如 `AttributeWriter` 和 `AttributeReader`）无需关心属性的具体类型和存储细节，只需通过 `AttributeFormatter` 的通用接口进行操作。这大大降低了系统的复杂性，使得上层逻辑更加清晰和稳定。
*   **多态扩展，易于维护**：当需要支持一种新的属性类型或存储格式时，只需派生一个新的格式化器类并实现其纯虚函数即可。这种设计符合“开闭原则”（Open-Closed Principle），即对扩展开放，对修改关闭。系统在不修改现有代码的情况下即可支持新功能，增强了代码的可扩展性和可维护性。
*   **责任分离，聚焦核心**：`AttributeFormatter` 将数据格式化的责任从读写逻辑中分离出来。`AttributeWriter` 专注于“何时”和“为何”写入数据，而 `AttributeFormatter` 则专注于“如何”将数据编码为二进制流。这种分离使得各自的实现可以独立演进。

其核心架构如下：

```
+---------------------------+
|      AttributeWriter      |
|      AttributeReader      |
+-------------+-------------+
              |
              | (Uses)
              v
+---------------------------+
|  AttributeFormatter (ABC) |
+-------------+-------------+
              ^
              | (Inherits)
              |
+---------------------------+
|  SingleValueAttribute...  |
|  VarNumAttribute...       |
|  ... (Concrete Formatters)|
+---------------------------+
```

## 3. 核心逻辑与算法

`AttributeFormatter` 本身不包含具体的实现逻辑，其核心在于定义了一套接口契约。

### 3.1. `Init` 方法

```cpp
virtual void Init(config::CompressTypeOption compressType = config::CompressTypeOption(),
                  bool supportNull = false) = 0;
```

*   **功能**：初始化格式化器。
*   **参数**：
    *   `compressType`：指定压缩类型，如 `fp16`、`int8` 等。
    *   `supportNull`：指示该属性是否支持空值。
*   **逻辑**：这是一个纯虚函数，具体的初始化逻辑由子类实现。子类通常会根据压缩类型和是否支持空值来确定记录的大小、内部状态等。

### 3.2. `Set` 和 `Reset` 方法

```cpp
virtual void Set(docid_t docId, const autil::StringView& attributeValue,
                 const util::ByteAlignedSliceArrayPtr& fixedData, bool isNull = false) = 0;

virtual bool Reset(docid_t docId, const autil::StringView& attributeValue,
                   const util::ByteAlignedSliceArrayPtr& fixedData, bool isNull = false) = 0;
```

*   **功能**：
    *   `Set`：用于向新的文档（`docId`）写入属性值。
    *   `Reset`：用于更新已存在的文档的属性值。
*   **参数**：
    *   `docId`：文档 ID。
    *   `attributeValue`：字符串视图表示的属性值。
    *   `fixedData`：指向存储属性数据的内存区域（通常是 `ByteAlignedSliceArray`）。
    *   `isNull`：标记该值是否为空。
*   **逻辑**：
    1.  **数据转换**：通过 `mAttrConvertor` 将 `attributeValue` 从字符串转换为内部二进制表示。
    2.  **写入数据**：将转换后的二进制数据根据 `docId` 写入到 `fixedData` 的相应位置。具体的写入方式（如定长、变长）由子类决定。
    3.  `Reset` 会额外检查 `docId` 是否有效，并可能执行原地更新（in-place update）的逻辑。

### 3.3. `Get` 方法

```cpp
virtual void Get(docid_t docId, const util::ByteAlignedSliceArrayPtr& fixedData, std::string& attributeValue,
                 bool& isNull) const;
```

*   **功能**：根据 `docId` 从 `fixedData` 中读取属性值。
*   **逻辑**：
    1.  **定位数据**：子类根据 `docId` 和自身的格式化逻辑（如记录大小）计算出属性值在 `fixedData` 中的位置。
    2.  **读取数据**：从计算出的位置读取二进制数据。
    3.  **数据转换**：将读取的二进制数据转换回字符串形式，并存入 `attributeValue`。
    4.  **空值处理**：同时判断并返回该值是否为空。

## 4. 关键实现细节

`AttributeFormatter` 的实现非常简洁，因为它是一个接口类。

### `attribute_formatter.h`

```cpp
#pragma once

#include <memory>

#include "indexlib/common/field_format/attribute/attribute_convertor.h"
#include "indexlib/common_define.h"
#include "indexlib/config/CompressTypeOption.h"
#include "indexlib/indexlib.h"
#include "indexlib/util/slice_array/ByteAlignedSliceArray.h"

DECLARE_REFERENCE_CLASS(file_system, FileStream);

namespace indexlib { namespace index {

class AttributeFormatter
{
public:
    AttributeFormatter();
    virtual ~AttributeFormatter();

public:
    // 初始化接口，纯虚函数，由子类实现
    virtual void Init(config::CompressTypeOption compressType = config::CompressTypeOption(),
                      bool supportNull = false) = 0;

    // 写入新数据的接口
    virtual void Set(docid_t docId, const autil::StringView& attributeValue,
                     const util::ByteAlignedSliceArrayPtr& fixedData, bool isNull = false) = 0;

    // 写入单个文档数据的接口（通常用于内存操作）
    virtual void Set(const autil::StringView& attributeValue, uint8_t* oneDocBaseAddr, bool isNull = false)
    {
        assert(false); // 基类中断言失败，强制子类实现
    }

    // 更新已存在数据的接口
    virtual bool Reset(docid_t docId, const autil::StringView& attributeValue,
                       const util::ByteAlignedSliceArrayPtr& fixedData, bool isNull = false) = 0;

    // 设置属性转换器
    void SetAttrConvertor(const common::AttributeConvertorPtr& attrConvertor);
    const common::AttributeConvertorPtr& GetAttrConvertor() const { return mAttrConvertor; }

    // 获取数据总长度的接口
    virtual uint32_t GetDataLen(int64_t docCount) const = 0;

public:
    // for test: 从 ByteAlignedSliceArray 获取数据
    virtual void Get(docid_t docId, const util::ByteAlignedSliceArrayPtr& fixedData, std::string& attributeValue,
                     bool& isNull) const
    {
        assert(false);
    }

    // for test: 从原始 buffer 获取数据
    virtual void Get(docid_t docId, const uint8_t*& buffer, std::string& attributeValue, bool& isNull) const
    {
        assert(false);
    }

protected:
    // 持有 AttributeConvertor，用于数据类型转换
    common::AttributeConvertorPtr mAttrConvertor;

private:
    IE_LOG_DECLARE();
};

DEFINE_SHARED_PTR(AttributeFormatter);
}} // namespace indexlib::index
```

### `attribute_formatter.cpp`

```cpp
#include "indexlib/index/normal/attribute/format/attribute_formatter.h"

using namespace std;
using namespace indexlib::common;

namespace indexlib { namespace index {
IE_LOG_SETUP(index, AttributeFormatter);

AttributeFormatter::AttributeFormatter() {}

AttributeFormatter::~AttributeFormatter() {}

// 唯一的实现就是设置转换器
void AttributeFormatter::SetAttrConvertor(const AttributeConvertorPtr& attrConvertor)
{
    assert(attrConvertor);
    mAttrConvertor = attrConvertor;
}
}} // namespace indexlib::index
```

*   **`assert(false)` 的妙用**：在基类的虚函数中直接 `assert(false)` 是一种强制子类重写该函数的设计技巧。如果子类忘记实现某个必要的虚函数，程序在调试阶段就会立即崩溃，从而暴露问题。
*   **`AttributeConvertorPtr`**：成员变量 `mAttrConvertor` 是连接原始数据和内部存储的桥梁。它负责将用户传入的 `StringView` 转换为特定类型的二进制数据，是实现数据解耦的关键。

## 5. 技术风险与未来展望

*   **技术风险**：
    *   **接口膨胀**：随着支持的属性类型和场景越来越复杂，`AttributeFormatter` 的接口可能会变得越来越臃肿。例如，未来如果需要支持更复杂的更新操作（如原子加、位运算等），可能需要向基类添加更多接口，这会增加所有子类的实现负担。
    *   **性能开销**：虚函数调用会带来微小的性能开销。在需要极致性能的场景下，这种开销可能会被放大。不过，在大多数情况下，数据转换和 I/O 的耗时远大于虚函数调用，因此这不是主要矛盾。

*   **未来展望**：
    *   **模板化与泛型编程**：可以考虑使用模板元编程（Template Metaprogramming）来进一步优化。通过模板，可以在编译期确定属性的类型和格式，从而消除虚函数调用，生成更高效的特化代码。但这会增加编译时间和代码的复杂性，需要权衡。
    *   **更细粒度的接口**：可以将 `AttributeFormatter` 的接口拆分得更细。例如，可以分离出只读接口（`ReadOnlyAttributeFormatter`）和读写接口，使得权限控制更加明确。
    *   **与新硬件结合**：未来可以设计新的 `AttributeFormatter` 子类，以充分利用新硬件的特性，例如针对持久内存（PMem）或 GPU 进行优化的数据格式。

总之，`AttributeFormatter` 作为 Indexlib 属性系统的基础组件，其抽象接口设计为整个系统带来了良好的扩展性和灵活性。虽然存在一些潜在的技术风险，但其清晰的架构和责任分离原则，为未来应对更复杂的需求奠定了坚实的基础。
