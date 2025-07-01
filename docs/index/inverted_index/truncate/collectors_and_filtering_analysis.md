
# Indexlib 截断模块收集器与过滤器（Collectors & Filtering）源码分析

**涉及文件:**
* `index/inverted_index/truncate/DocCollector.h`
* `index/inverted_index/truncate/NoSortTruncateCollector.h`
* `index/inverted_index/truncate/NoSortTruncateCollector.cpp`
* `index/inverted_index/truncate/SortTruncateCollector.h`
* `index/inverted_index/truncate/SortTruncateCollector.cpp`
* `index/inverted_index/truncate/IDocFilterProcessor.h`
* `index/inverted_index/truncate/DocFilterProcessorTyped.h`
* `index/inverted_index/truncate/DocPayloadFilterProcessor.h`
* `index/inverted_index/truncate/DocPayloadFilterProcessor.cpp`
* `index/inverted_index/truncate/DocDistinctor.h`
* `index/inverted_index/truncate/DocCollectorCreator.h`
* `index/inverted_index/truncate/DocCollectorCreator.cpp`

---

## 1. 概述

在 Indexlib 的倒排索引截断流程中，**收集器（Collector）** 和 **过滤器（Filter）** 是数据准备阶段的核心。当系统决定对一个词（Term）的倒排链进行截断时，首先需要将满足条件的文档（Document）从原始倒排链中筛选并收集起来，形成一个候选集。这个候选集随后会被排序并最终截断。收集器和过滤器正是负责这一关键任务的组件。

- **收集器 (DocCollector)**: 它的核心职责是遍历倒排链（`PostingIterator`），将文档 ID（`docid_t`）收集到一个临时的列表中。收集器分为两大类：**非排序收集器** (`NoSortTruncateCollector`) 和 **排序收集器** (`SortTruncateCollector`)。前者仅按顺序收集文档，直到达到数量上限；后者则更为复杂，它会根据预定义的排序规则（通过 `BucketMap` 或直接排序）来收集和组织文档，为后续的高效截断做准备。

- **过滤器 (IDocFilterProcessor)**: 在收集过程中，过滤器扮演着“守门员”的角色。它可以在文档被加入收集列表之前，对其进行预筛选。这种筛选可以是基于文档的某个属性值是否在指定范围内（`DocFilterProcessorTyped`），也可以是基于文档的 `payload` 值（`DocPayloadFilterProcessor`）。过滤机制可以显著减少需要处理的文档数量，提高截断效率。

- **去重器 (DocDistinctor)**: 去重器是过滤器的一种特殊形式，主要用于实现多样性（Diversity）需求。例如，在电商搜索中，为了避免搜索结果页被同一个卖家的商品霸屏，可以配置按卖家 ID 进行去重，保留每个卖家的若干个商品。`DocDistinctor` 正是用于实现这种基于某个字段的去重逻辑。

- **创建器 (DocCollectorCreator)**: 这是一个工厂类，它根据索引的截断配置（`TruncateIndexProperty`），动态地创建和组装出合适的收集器、过滤器和去重器实例，将这些组件有机地结合在一起。

这套体系的设计目标是提供一个高效、灵活的文档收集与预处理框架，以支持各种复杂的截断和多样性策略。

---

## 2. 核心设计与实现机制

### 2.1. 文档收集器 (DocCollector) 体系

`DocCollector` 是所有收集器的基类，它定义了收集流程的整体框架。

#### 2.1.1. `DocCollector` 基类

`DocCollector` 定义了文档收集的核心流程和通用属性。

**核心逻辑**:
`CollectDocIds` 方法是整个收集过程的入口。它首先初始化内部状态（如分配用于存储 `docId` 的向量），然后调用 `DoCollect` 这个由子类实现的纯虚函数来执行具体的收集逻辑。收集完成后，调用 `Truncate` 方法进行最终的整理和截断，最后通过 `GetTruncateDocIds` 提供结果。

**关键实现**:
```cpp
// index/inverted_index/truncate/DocCollector.h

class DocCollector
{
public:
    // ...
    void CollectDocIds(const DictKeyInfo& key, const std::shared_ptr<PostingIterator>& postingIt, df_t docFreq)
    {
        // ... 初始化操作 ...
        _docIdVec = _bucketVecAllocator->AllocateDocVector();
        // ...
        if (_filterProcessor) {
            _filterProcessor->BeginFilter(key, postingIt);
        }
        ReInit(); // 子类实现的初始化
        DoCollect(postingIt); // 子类实现的收集逻辑
        Truncate(); // 子类实现的截断逻辑
        if (_docDistinctor) {
            _docDistinctor->Reset();
        }
    }
    // ...
private:
    virtual void ReInit() = 0;
    virtual void DoCollect(const std::shared_ptr<PostingIterator>& postingIt) = 0;
    virtual void Truncate() = 0;

protected:
    uint64_t _minDocCountToReserve; // 最小保留文档数
    uint64_t _maxDocCountToReserve; // 最大保留文档数（用于多样性）
    std::shared_ptr<IDocFilterProcessor> _filterProcessor;
    std::shared_ptr<DocDistinctor> _docDistinctor;
    std::shared_ptr<DocIdVector> _docIdVec;
    // ...
};
```
这种模板方法模式（Template Method Pattern）的设计，使得基类可以控制收集流程的骨架，而具体的收集和截断策略则交由子类实现。

#### 2.1.2. `NoSortTruncateCollector`：非排序收集

这是最简单的收集器。它的策略是按 `docId` 顺序遍历倒排链，收集文档直到达到指定的数量限制 (`_minDocCountToReserve`)。

**核心逻辑**:
`DoCollect` 方法非常直接：它循环调用 `postingIt->SeekDoc()`，将获取到的 `docId` 依次存入 `_docIdVec`，直到向量大小达到 `_minDocCountToReserve` 或 `_maxDocCountToReserve`（如果配置了去重）。它不关心文档的任何属性或 `payload`，因此没有排序行为。

**关键实现**:
```cpp
// index/inverted_index/truncate/NoSortTruncateCollector.cpp

void NoSortTruncateCollector::DoCollect(const std::shared_ptr<PostingIterator>& postingIt)
{
    docid_t docId = 0;
    while ((docId = postingIt->SeekDoc(docId)) != INVALID_DOCID) {
        if (_filterProcessor != nullptr && _filterProcessor->IsFiltered(docId)) {
            continue;
        }
        if (_docDistinctor == nullptr) {
            if (_docIdVec->size() < _minDocCountToReserve) {
                _docIdVec->push_back(docId);
            } else {
                return; // 达到数量上限，提前终止
            }
        } else {
            // ... 处理去重逻辑 ...
        }
    }
}
```
这种收集器适用于那些不需要排序，只需保留倒排链开头的若干文档的简单截断场景。

#### 2.1.3. `SortTruncateCollector`：排序收集

`SortTruncateCollector` 是截断功能的核心，它实现了基于排序的文档收集。它支持两种主要的排序方式：**基于 `BucketMap` 的分桶排序** 和 **基于 `doc_payload` 的直接排序**。

**1. 基于 `BucketMap` 的分桶排序**

这是 `SortTruncateCollector` 的主要工作模式。`BucketMap` 是一个预计算的数据结构，它将所有文档根据其排序属性值映射到不同的“桶”中，并记录了每个文档在桶内的相对顺序。排序属性值越高的文档，其所在的桶的编号和在桶内的位置就越靠前。

**核心逻辑**:
`DoCollect` 方法遍历倒排链，对于每个 `docId`，它从 `_bucketMap` 中查询其所属的桶号（`bucketValue`），然后将 `docId` 放入对应桶的 `DocIdVector` 中。这个过程本身是无序的，只是将文档按预计算的规则进行分组。

`Truncate` 方法是真正的排序和截断发生的地方。它从编号最大的桶（代表排序值最高的文档）开始，依次将桶内的 `docId` 收集到最终的 `_docIdVec` 中。当某个桶可能导致总文档数超过 `docCountToReserve` 时，它会对这个“临界桶”内部的文档进行排序（使用 `std::sort` 和自定义的 `Comparator`），然后只取所需数量的文档，从而完成精确截断。

**关键实现**:
```cpp
// index/inverted_index/truncate/SortTruncateCollector.cpp

void SortTruncateCollector::DoCollect(const std::shared_ptr<PostingIterator>& postingIt)
{
    // ...
    while ((docId = postingIt->SeekDoc(docId)) != INVALID_DOCID) {
        // ... 过滤逻辑 ...
        uint32_t bucketValue = _bucketMap->GetBucketValue(docId);
        // ...
        bucketVec[bucketValue].push_back(docId);
        // ... 动态调整有效桶的范围，进行内存优化
    }
}

void SortTruncateCollector::SortLastValidBucketAndTruncate(uint64_t docCountToReserve)
{
    // ...
    for (size_t i = 0; i < bucketVec.size(); ++i) {
        // ...
        if (totalCount + bucketVec[i].size() >= docCountToReserve || i == bucketVec.size() - 1) {
            // 对临界桶进行内部排序
            SortBucket(bucketVec[i], _bucketMap);
            count = std::min(docCountToReserve - totalCount, bucketVec[i].size());
            finish = true;
        }
        // ...
        docIdVec.insert(docIdVec.end(), bucketVec[i].begin(), bucketVec[i].begin() + count);
        // ...
    }
    // ...
}
```
这种分桶机制是一种非常高效的近似排序。它避免了对整个倒排链中的所有文档进行全排序，而是将排序的开销限制在少数临界桶上，极大地提高了截断性能。

**2. 基于 `doc_payload` 的直接排序**

当截断配置指定按 `doc_payload` 排序时，`SortTruncateCollector` 会采用不同的策略。它会先将所有满足条件的文档（`docId` 和 `payload`）收集到一个 `std::vector<Doc>` 中，然后调用 `std::nth_element` 或 `std::sort` 进行排序。

**核心逻辑**:
`DoCollect` 遍历倒排链，将 `docId` 和 `payload` 存入 `_docInfos` 向量。`Truncate` 方法则调用 `SortWithDocPayload`，后者是一个递归的、支持多维排序的函数。它使用 `std::nth_element` 来高效地找到第 k 个元素（即截断点），然后对排序值相同的元素递归调用下一维度的排序。这种方式实现了对 `payload` 和其他属性组合的复杂排序。

### 2.2. 过滤器与去重器

#### 2.2.1. `IDocFilterProcessor` 接口

`IDocFilterProcessor` 定义了过滤器的标准接口。

```cpp
// index/inverted_index/truncate/IDocFilterProcessor.h

class IDocFilterProcessor
{
public:
    // ...
    virtual bool BeginFilter(const DictKeyInfo& key, const std::shared_ptr<PostingIterator>& postingIt) = 0;
    virtual bool IsFiltered(docid_t docId) = 0;
    // ...
};
```
- `BeginFilter`: 在处理一个新的倒排链之前调用，可以进行一些初始化工作，例如从 `TruncateMetaReader` 中查找该词是否需要过滤。
- `IsFiltered`: 对每个 `docId` 进行判断，返回 `true` 表示该文档应被过滤掉。

#### 2.2.2. `DocFilterProcessorTyped` 和 `DocPayloadFilterProcessor`

- `DocFilterProcessorTyped<T>`: 基于文档属性进行过滤。它读取指定属性的值，并检查该值是否落在配置的 `[min, max]` 范围内。它还支持一个 `mask` 操作，可以对属性值进行位运算后再比较。
- `DocPayloadFilterProcessor`: 基于 `doc_payload` 进行过滤。逻辑与 `DocFilterProcessorTyped` 类似，但数据源是 `postingIt->GetDocPayload()`。

#### 2.2.3. `DocDistinctor`：实现多样性

`DocDistinctor` 用于实现多样性截断。它内部维护一个 `HashMap`，记录已经见过的多样性字段的值（如卖家 ID）。

**核心逻辑**:
`Distinct` 方法接收一个 `docId`，读取其多样性字段的属性值。如果该属性值在 `HashMap` 中已存在，表示该分组的文档已达到数量上限，可以跳过；如果不存在，则将其插入 `HashMap`，并保留该文档。当 `HashMap` 的大小达到预设的 `distCount` 时，`IsFull()` 返回 `true`，表示每个分组都已至少有一个文档，后续的文档处理策略可能会发生变化（例如，开始填充直到满足 `_minDocCountToReserve`）。

**关键实现**:
```cpp
// index/inverted_index/truncate/DocDistinctor.h

template <typename T>
bool DocDistinctorTyped<T>::Distinct(docid_t docId)
{
    T attrValue {};
    // ... 读取属性值 ...
    if (!isNull) {
        _distmap->FindAndInsert(attrValue, true);
    }
    uint32_t curDistCount = _distmap->Size();
    if (curDistCount >= _distCount) {
        return true; // 所有分组都已满足
    }
    return false;
}
```

---

## 3. 技术风险与考量

1.  **内存消耗**:
    - `SortTruncateCollector` 在处理大型倒排链时，其内部的 `_bucketVec` 或 `_docInfos` 可能会消耗大量内存。特别是 `_bucketVec`，它的大小由总文档数决定，即使某个词的倒排链不长，也会分配一个巨大的桶向量。虽然代码中有一些内存优化的尝试（如 `memOptimizeThreshold`），但在极端情况下内存压力依然很大。
    - `DocDistinctor` 内部的 `HashMap` 也会随着多样性字段的基数（Cardinality）增大而消耗更多内存。

2.  **性能瓶颈**:
    - **属性读取**: `DocFilterProcessorTyped` 和 `DocDistinctor` 每次判断都需要读取文档属性，这会引入 I/O 开销，影响收集速度。
    - **排序开销**: `SortTruncateCollector` 虽然通过分桶优化了排序，但在临界桶很大或直接按 `payload` 排序时，`std::sort` 或 `std::nth_element` 的开销依然不可忽视。

3.  **配置复杂性**:
    - 截断、过滤和多样性的组合配置非常灵活，但也容易出错。例如，过滤规则与排序规则可能存在逻辑冲突，导致截断结果不符合预期。`DocCollectorCreator` 中复杂的创建逻辑也反映了这种配置的复杂性。

4.  **`BucketMap` 的依赖**:
    - `SortTruncateCollector` 的高效运行严重依赖于预先构建好的 `BucketMap`。`BucketMap` 的生成本身是一个独立的、耗时的过程（在 `BucketMapCreator` 中实现），它需要在合并（Merge）阶段完成。如果 `BucketMap` 的生成逻辑有误，或者其分布不均，都会直接影响截断的准确性和性能。

---

## 4. 总结

Indexlib 的收集器与过滤器模块为倒排索引的截断功能提供了强大的数据预处理能力。其设计精良，通过组合不同的组件，实现了对复杂业务规则的支持。

- **策略模式的体现**: `NoSortTruncateCollector` 和 `SortTruncateCollector` 是对 `DocCollector` 基类不同策略的实现，清晰地分离了不同场景下的收集逻辑。
- **装饰器模式的应用**: `IDocFilterProcessor` 和 `DocDistinctor` 可以看作是对核心收集逻辑的装饰。它们在不改变收集器核心流程的情况下，为其增加了过滤和去重的能力。
- **性能与功能的权衡**: `SortTruncateCollector` 中基于 `BucketMap` 的分桶排序机制，是典型的用空间换时间、用预计算换取在线高性能的例子。它在保证排序效果的同时，巧妙地避免了对海量数据进行全排序，是一个非常出色的工程优化实践。

总体而言，该模块的设计展示了在满足复杂需求（排序、过滤、多样性）和追求极致性能之间取得平衡的工程智慧。
