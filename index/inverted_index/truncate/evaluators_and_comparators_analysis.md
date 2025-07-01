
# Indexlib 截断模块评估器与比较器（Evaluators & Comparators）源码分析

**涉及文件:**
* `index/inverted_index/truncate/IEvaluator.h`
* `index/inverted_index/truncate/AttributeEvaluator.h`
* `index/inverted_index/truncate/DocPayloadEvaluator.h`
* `index/inverted_index/truncate/DocPayloadEvaluator.cpp`
* `index/inverted_index/truncate/MultiAttributeEvaluator.h`
* `index/inverted_index/truncate/MultiAttributeEvaluator.cpp`
* `index/inverted_index/truncate/Comparator.h`
* `index/inverted_index/truncate/MultiComparator.h`
* `index/inverted_index/truncate/MultiComparator.cpp`
* `index/inverted_index/truncate/Reference.h`
* `index/inverted_index/truncate/ReferenceTyped.h`

---

## 1. 概述

在 Indexlib 的倒排索引截断（Truncate）流程中，**评估器（Evaluator）** 和 **比较器（Comparator）** 扮演着至关重要的角色。它们共同构成了截断排序的核心机制。当一个词（Term）的倒排链（Posting List）过长，需要根据预设规则进行截断时，系统必须能够评估每个文档（Document）的“价值”或“权重”，并根据这些价值对文档进行排序，最终保留价值最高的文档。

- **评估器 (Evaluator)** 的核心职责是为每个文档计算一个或多个排序依据的数值。这些数值可以来源于文档的某个或某些属性（Attribute），也可以是文档在倒排链中的 payload 信息，甚至是两者的结合。评估器将计算出的值写入一个临时的 `DocInfo` 对象中，供后续的比较和排序使用。

- **比较器 (Comparator)** 则负责定义文档间的排序规则。它利用评估器填充到 `DocInfo` 中的值，对两个 `DocInfo` 对象进行比较，从而确定它们的先后顺序。系统支持单一维度或多维度的排序，也支持升序或降序排列。

- **引用 (Reference)** 是评估器和比较器之间的桥梁。它提供了一种类型安全的方式来访问 `DocInfo` 对象中存储的、由评估器计算出的具体值。`Reference` 知道特定值在 `DocInfo` 内存布局中的偏移量和数据类型，使得比较器可以精确地读取这些值进行比较。

这套机制的设计目标是实现一个高度灵活、可扩展的排序框架。通过组合不同的评估器和比较器，用户可以轻松定义复杂的截断策略，以满足多样化的业务需求，例如按商品销量、用户评分或相关性分数进行排序。

---

## 2. 核心设计与实现机制

### 2.1. 评估器 (Evaluator) 体系

评估器体系围绕 `IEvaluator` 接口构建，定义了所有评估器必须遵循的核心规范。

#### 2.1.1. `IEvaluator` 接口

`IEvaluator` 是所有评估器的基类，它定义了两个核心的纯虚函数：

```cpp
// index/inverted_index/truncate/IEvaluator.h

class IEvaluator
{
public:
    IEvaluator() = default;
    virtual ~IEvaluator() = default;

public:
    // 计算文档的评估值，并将其存入 DocInfo 对象
    virtual void Evaluate(docid_t docId, const std::shared_ptr<PostingIterator>& postingIter, DocInfo* docInfo) = 0;
    
    // 直接获取文档的评估值，通常用于某些需要直接使用数值的场景（如多样性过滤）
    virtual double GetValue(docid_t docId, const std::shared_ptr<PostingIterator>& postingIter, DocInfo* docInfo) = 0;
};
```

- `Evaluate`: 这是评估器的主要功能。它接收一个 `docId` 和一个指向当前倒排链的 `postingIter`，然后计算出评估值，并通过 `Reference` 将其写入 `docInfo` 对象。
- `GetValue`: 该函数直接返回一个 `double` 类型的评估值。这个设计主要服务于一些特殊场景，例如在进行多样性过滤（Diversity Filter）时，需要直接获取属性值进行范围判断，而不是为了排序。

这种接口设计将“评估”和“存储”两个动作解耦，使得上层逻辑可以根据需要选择不同的操作。

#### 2.1.2. `AttributeEvaluator`：基于属性的评估

`AttributeEvaluator` 是最常用的评估器之一，它根据文档的某个特定属性值进行评估。

**核心逻辑**:
它在构造时接收一个 `AttributeDiskIndexer`（属性的磁盘读取器）和一个 `ReferenceTyped<T>`（类型化的 `DocInfo` 引用）。当 `Evaluate` 方法被调用时，它通过 `AttributeDiskIndexer` 读取指定 `docId` 的属性值，然后通过 `ReferenceTyped<T>` 将这个值写入 `DocInfo` 对象的正确位置。

**关键实现**:
```cpp
// index/inverted_index/truncate/AttributeEvaluator.h

template <typename T>
class AttributeEvaluator : public IEvaluator
{
public:
    AttributeEvaluator(const std::shared_ptr<indexlibv2::index::AttributeDiskIndexer>& attrDiskIndexer,
                       Reference* refer)
    {
        _refer = dynamic_cast<ReferenceTyped<T>*>(refer);
        assert(_refer);
        _attrDiskIndexer = attrDiskIndexer;
        _diskReadCtx = _attrDiskIndexer->CreateReadContextPtr(&_pool);
    }

    // ...

    void Evaluate(docid_t docId, const std::shared_ptr<PostingIterator>& postingIter,
                  DocInfo* docInfo) override
    {
        T attrValue {};
        bool isNull = false;
        assert(_attrDiskIndexer);
        uint32_t dataLen = 0;
        // 从磁盘读取属性值
        _attrDiskIndexer->Read(docId, _diskReadCtx, (uint8_t*)&attrValue, sizeof(T), dataLen, isNull);
        // 通过 Reference 写入 DocInfo
        _refer->Set(attrValue, isNull, docInfo);
    }

    // ...
private:
    ReferenceTyped<T>* _refer;
    std::shared_ptr<indexlibv2::index::AttributeDiskIndexer> _attrDiskIndexer;
    std::shared_ptr<indexlibv2::index::AttributeDiskIndexer::ReadContextBase> _diskReadCtx;
    autil::mem_pool::UnsafePool _pool;
};
```
该实现利用模板 `T` 实现了对不同属性类型（如 `int32_t`, `double` 等）的通用支持。通过 `ReferenceTyped<T>`，它能精确地知道应将读取到的 `T` 类型数据写入 `DocInfo` 的哪个内存位置。

#### 2.1.3. `DocPayloadEvaluator`：基于 Payload 的评估

`DocPayloadEvaluator` 用于处理那些排序依据直接存储在倒排链的 `payload` 中的场景。`payload` 通常用于存储与特定词相关的文档级别信息，如 TF-IDF 分数、BM25 分数或其他自定义的相关性分数。

**核心逻辑**:
它从 `PostingIterator` 中获取当前文档的 `docpayload_t` 值，并将其写入 `DocInfo`。特别地，它还支持一个“因子评估器”（`factorEvaluator`），可以将 `payload` 值与另一个属性值相乘，从而实现更复杂的混合排序策略。

**关键实现**:
```cpp
// index/inverted_index/truncate/DocPayloadEvaluator.cpp

void DocPayloadEvaluator::Evaluate(docid_t docId, const std::shared_ptr<PostingIterator>& postingIter, DocInfo* docInfo)
{
    double docPayload = GetValue(docId, postingIter, docInfo);
    _refer->Set(docPayload, false, docInfo);
}

double DocPayloadEvaluator::GetValue(docid_t docId, const std::shared_ptr<PostingIterator>& postingIter,
                                     DocInfo* docInfo)
{
    // 从 posting iterator 获取 payload
    double docPayload = EvaluateDocPayload(postingIter);
    // 如果设置了因子评估器，则将 payload 与另一个属性值相乘
    if (_factorEvaluator) {
        docPayload *= _factorEvaluator->GetValue(docId, postingIter, docInfo);
    }
    return docPayload;
}
```
此外，`DocPayloadEvaluator` 还考虑了 `FP16`（半精度浮点数）的场景，以节省存储空间。

#### 2.1.4. `MultiAttributeEvaluator`：组合多个评估器

为了支持多维排序，系统需要一次性为文档计算多个评估值。`MultiAttributeEvaluator` 就是为此设计的。

**核心逻辑**:
它内部维护一个 `IEvaluator` 的列表。当其 `Evaluate` 方法被调用时，它会依次调用列表中每个子评估器的 `Evaluate` 方法。

**关键实现**:
```cpp
// index/inverted_index/truncate/MultiAttributeEvaluator.cpp

void MultiAttributeEvaluator::AddEvaluator(const std::shared_ptr<IEvaluator>& evaluator)
{
    _evaluators.push_back(evaluator);
}

void MultiAttributeEvaluator::Evaluate(docid_t docId, const std::shared_ptr<PostingIterator>& postingIter,
                                       DocInfo* docInfo)
{
    // 依次调用所有子评估器
    for (auto i = 0; i < _evaluators.size(); ++i) {
        _evaluators[i]->Evaluate(docId, postingIter, docInfo);
    }
}
```
这种组合模式使得评估器体系具有极佳的扩展性。上层逻辑（如 `EvaluatorCreator`）可以根据配置文件动态地创建和组合多个评估器，以满足复杂的排序需求，而无需修改核心排序逻辑。

### 2.2. 比较器 (Comparator) 体系

比较器负责定义具体的排序逻辑，它利用评估器填充好的 `DocInfo` 对象来进行比较。

#### 2.2.1. `Comparator` 接口与 `ComparatorTyped`

`Comparator` 是所有比较器的基类，定义了一个核心的 `LessThan` 方法。

```cpp
// index/inverted_index/truncate/Comparator.h

class Comparator
{
public:
    // ...
    virtual bool LessThan(const DocInfo* left, const DocInfo* right) const = 0;
};
```

`ComparatorTyped<T>` 是 `Comparator` 的一个模板实现，它通过 `ReferenceTyped<T>` 来获取 `DocInfo` 中的具体值，并根据升序（asc）或降序（desc）标志进行比较。

**关键实现**:
```cpp
// index/inverted_index/truncate/Comparator.h

template <typename T>
class ComparatorTyped : public Comparator
{
public:
    ComparatorTyped(Reference* refer, bool desc)
    {
        _ref = dynamic_cast<ReferenceTyped<T>*>(refer);
        assert(_ref);
        _desc = desc;
    }

    bool LessThan(const DocInfo* left, const DocInfo* right) const override
    {
        T t1 {}, t2 {};
        bool null1 = false, null2 = false;
        // 通过 Reference 获取值
        _ref->Get(left, t1, null1);
        _ref->Get(right, t2, null2);
        
        // 处理 NULL 值，并根据 desc 标志进行比较
        if (!_desc) { // asc
            if (null1 && null2) return false;
            if (null1) return true; // NULL 值排在最前面
            if (null2) return false;
            return t1 < t2;
        } else { // desc
            if (null1) return false;
            if (null2) return true; // NULL 值排在最后面
            return t1 > t2;
        }
    }
private:
    ReferenceTyped<T>* _ref;
    bool _desc;
};
```
该实现清晰地展示了比较器如何与 `Reference` 协作，并优雅地处理了 `NULL` 值和排序方向。

#### 2.2.2. `MultiComparator`：多维排序的实现

`MultiComparator` 实现了多维排序（Multi-level Sort）的逻辑，类似于 SQL 中的 `ORDER BY key1, key2`。

**核心逻辑**:
它内部维护一个 `Comparator` 的列表，代表了排序的优先级。比较时，它会按照列表顺序依次使用子比较器进行比较。只有当上一个维度的比较结果为相等时，才会使用下一个维度的比较器。

**关键实现**:
```cpp
// index/inverted_index/truncate/MultiComparator.cpp

bool MultiComparator::LessThan(const DocInfo* left, const DocInfo* right) const
{
    for (size_t i = 0; i < _comps.size(); ++i) {
        // 比较第一个维度
        if (_comps[i]->LessThan(left, right)) {
            return true;
        } 
        // 如果第一个维度不小于，再反向比较一次，判断是否大于
        else if (_comps[i]->LessThan(right, left)) {
            return false;
        }
        // 如果相等，则继续下一个维度的比较
    }
    // 所有维度都相等
    return false;
}
```
这种设计使得实现多维排序变得非常简单和直观。上层逻辑只需按顺序创建并添加多个 `ComparatorTyped` 实例到 `MultiComparator` 中即可。

### 2.3. `Reference`：连接评估器与比较器的桥梁

`Reference` 的设计是整个体系的精髓所在。它解决了评估器和比较器之间如何安全、高效地传递和访问类型化数据的问题。

**核心逻辑**:
`Reference` 及其子类 `ReferenceTyped<T>` 封装了对 `DocInfo` 对象内部数据的访问细节。`DocInfo` 本身可以看作是一个原始的字节缓冲区，而 `Reference` 则为这个缓冲区提供了带类型和偏移量信息的“视图”。

- **偏移量 (`_offset`)**: `Reference` 在创建时被赋予一个偏移量，这个偏移量由 `DocInfoAllocator` 在声明（`DeclareReference`）时动态计算和分配。每个 `Reference` 都知道自己负责的数据存储在 `DocInfo` 缓冲区的哪个位置。
- **类型信息 (`_fieldType`)**: `Reference` 存储了字段的类型，而 `ReferenceTyped<T>` 则通过模板参数 `T` 提供了编译期的类型信息。
- **空值支持 (`_supportNull`)**: `Reference` 知道字段是否支持 `NULL` 值。如果支持，它会在数据前额外存储一个 `bool` 标志位。

**关键实现**:
```cpp
// index/inverted_index/truncate/ReferenceTyped.h

template <typename T>
class ReferenceTyped : public Reference
{
public:
    // ...
    void Get(const DocInfo* docInfo, T& value, bool& isNull) const
    {
        if (!_supportNull) {
            value = *(T*)(docInfo->Get(_offset));
            isNull = false;
            return;
        }

        isNull = *(bool*)(docInfo->Get(_offset));
        if (!isNull) {
            value = *(T*)(docInfo->Get(_offset + sizeof(bool)));
        }
    }

    void Set(const T& value, const bool& isNull, DocInfo* docInfo)
    {
        if (!_supportNull) {
            uint8_t* buffer = docInfo->Get(_offset);
            *((T*)buffer) = value;
            return;
        }

        memcpy(docInfo->Get(_offset), &isNull, sizeof(bool));
        if (!isNull) {
            memcpy(docInfo->Get(_offset + sizeof(bool)), &value, sizeof(T));
        }
    }
    // ...
};
```
通过这种方式，`DocInfo` 的内存布局被动态地构建出来，而评估器和比较器则通过各自持有的 `Reference` 指针，实现了对这块内存的类型安全、结构化的访问，避免了硬编码偏移量和不安全的类型转换。

---

## 3. 技术风险与考量

1.  **性能开销**:
    - **属性读取**: `AttributeEvaluator` 每次评估都需要从磁盘（或文件系统缓存）读取属性值，这可能成为性能瓶颈，尤其是在处理海量文档和复杂排序规则时。尽管有缓存机制，但频繁的 I/O 依然是主要开销。
    - **虚函数调用**: 整个体系大量使用虚函数（`Evaluate`, `LessThan`），这会带来一定的性能开销。但在这种需要高度灵活性的场景下，这是可以接受的权衡。

2.  **内存管理**:
    - `DocInfo` 对象由 `DocInfoAllocator` 分配和管理，后者使用内存池（`RecyclePool`）来提高分配效率。然而，如果排序过程中涉及的文档数量巨大，`DocInfo` 对象本身会占用大量内存。
    - `AttributeDiskIndexer` 在读取数据时也会使用内存池。需要谨慎管理这些内存池的生命周期和大小，防止内存泄漏或过度消耗。

3.  **类型安全与配置正确性**:
    - 整个体系的正确运行高度依赖于配置的正确性。例如，为 `int` 类型的属性配置了 `double` 类型的评估器或比较器，虽然编译时可能通过，但会在运行时导致未定义行为。
    - `dynamic_cast` 的使用（如 `AttributeEvaluator` 的构造函数中）虽然提供了运行时的类型检查，但如果转换失败（返回 `nullptr`），后续的 `assert` 会在 Debug 模式下触发断言，而在 Release 模式下可能导致空指针解引用。这要求上游的创建逻辑（`EvaluatorCreator`, `DocCollectorCreator` 等）必须保证传入的 `Reference` 类型是正确的。

4.  **扩展性与维护**:
    - **优点**: 基于接口和组合的设计使得添加新的评估和比较逻辑变得容易，只需实现相应的接口即可，对现有代码的侵入性很小。
    - **缺点**: 随着支持的排序字段和策略增多，`Creator` 类（如 `EvaluatorCreator`）的逻辑会变得越来越复杂，需要处理各种字段类型和配置组合，增加了维护成本。

---

## 4. 总结

Indexlib 的评估器与比较器模块共同构建了一个功能强大且高度可扩展的排序框架。其核心设计思想在于 **接口隔离**、**组合模式** 和 **类型化引用**。

- **接口隔离**: `IEvaluator` 和 `Comparator` 接口清晰地定义了各自的职责，使得评估逻辑和比较逻辑可以独立演化。
- **组合模式**: `MultiAttributeEvaluator` 和 `MultiComparator` 的设计使得系统可以轻松地将简单的单维评估/比较组合成复杂的多维逻辑，极大地增强了框架的灵活性。
- **类型化引用**: `Reference` 和 `ReferenceTyped<T>` 的设计巧妙地解决了在评估器和比较器之间传递和访问类型化数据的问题，它作为 `DocInfo` 这个通用数据容器的“访问代理”，保证了类型安全和内存布局的灵活性。

尽管存在性能和配置复杂性方面的挑战，但这套设计成功地将截断排序的核心逻辑抽象出来，为实现各种复杂的业务排序需求提供了坚实的基础。
