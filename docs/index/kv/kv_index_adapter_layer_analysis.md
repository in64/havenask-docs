
# Indexlib KV 索引适配器层源码分析

**涉及文件:**
* `index/kv/AdapterKVSegmentReader.h`
* `index/kv/AdapterKVSegmentReader.cpp`
* `index/kv/AdapterKVSegmentIterator.h`
* `index/kv/AdapterKVSegmentIterator.cpp`
* `index/kv/AdapterIgnoreFieldCalculator.h`
* `index/kv/AdapterIgnoreFieldCalculator.cpp`

---

## 1. 概述

在大型、长生命周期的系统中，数据模式（Schema）的演进是不可避免的。对于 Indexlib 的 KV 索引而言，这意味着 Value 的结构可能会随着业务需求的变化而改变，例如增加新字段、删除旧字段或修改字段类型。当线上系统同时存在使用不同版本 Schema 构建的索引数据（Segment）时，如何保证上层应用能够用统一的方式、无感知地读取这些异构数据，就成了一个核心挑战。

**KV 索引适配器层（Adapter Layer）** 正是为了解决这一挑战而设计的。它位于上层查询逻辑和底层具体的 Segment 读取器之间，扮演着一个“翻译官”和“协调者”的角色。其核心目标是：**将底层不同 Schema 版本的数据，动态地转换成上层应用当前期望的 Schema 版本，从而对上层调用者屏蔽底层数据的异构性。**

这个适配器层主要由以下三个关键组件构成：

1.  **`AdapterKVSegmentReader`**: 这是一个装饰器（Decorator），它包装了底层的 `IKVSegmentReader`。当上层调用 `Get` 方法查询一个 Key 时，它首先通过底层 Reader 获取原始的、未经转换的 Value 数据，然后利用 `PackValueAdapter` 将这个 Value 从旧 Schema 格式转换成新 Schema 格式，最后再返回给调用者。

2.  **`AdapterKVSegmentIterator`**: 与 `Reader` 类似，这是一个用于遍历整个 Segment 数据的迭代器装饰器。它包装了底层的 `IKVIterator`，在 `Next` 方法被调用时，对返回的每条记录（Record）的 Value 执行与 `Reader` 相同的格式转换。

3.  **`AdapterIgnoreFieldCalculator`**: 这是适配逻辑中的一个关键辅助组件。在 Schema 演进过程中，一个字段可能被删除后又被重新以相同的名字、不同的类型添加回来。在这种情况下，从旧 Schema 向新 Schema 转换时，必须忽略掉这个旧字段，因为它与新字段已无关联。`AdapterIgnoreFieldCalculator` 的职责就是通过分析整个 Schema 的演进历史，计算出在进行数据转换时需要被忽略的字段列表。

通过这套适配器机制，Indexlib 实现KV索引的平滑在线 Schema 变更，极大地提升了系统的灵活性和可维护性。

---

## 2. 核心设计与实现机制

### 2.1. `AdapterKVSegmentReader`：单点查询的适配

`AdapterKVSegmentReader` 是适配器模式和装饰器模式的典型应用。它在不改变 `IKVSegmentReader` 接口的前提下，为其增加了“数据格式转换”的额外功能。

**设计思想**:
- **包装 (Wrapping)**: 它在构造时接收一个指向底层具体 `IKVSegmentReader` 实现的共享指针（`_reader`），从而持有了对原始数据的访问能力。
- **功能增强 (Enhancement)**: 它还持有一个 `PackValueAdapter` 实例，这是执行具体格式转换的核心工具。`PackValueAdapter` 知道源 Schema 和目标 Schema 的详细布局，能够将一个序列化的 Value 数据（`autil::StringView`）从源格式解析并重新序列化为目标格式。
- **透明性 (Transparency)**: 它自身也实现了 `IKVSegmentReader` 接口，这意味着对于上层调用者来说，它和一个普通的 `IKVSegmentReader` 没有任何区别。调用者像往常一样调用 `Get` 方法，而无需关心其内部发生的复杂转换。

**关键实现**:
```cpp
// index/kv/AdapterKVSegmentReader.cpp

FL_LAZY(indexlib::util::Status)
AdapterKVSegmentReader::Get(keytype_t key, autil::StringView& value, uint64_t& ts, autil::mem_pool::Pool* pool,
                            KVMetricsCollector* collector, autil::TimeoutTerminator* timeoutTerminator) const
{
    // 1. 调用被包装的 reader，获取原始的、未经转换的 value
    auto status = FL_COAWAIT _reader->Get(key, value, ts, pool, collector, timeoutTerminator);
    
    // 2. 如果成功获取到数据
    if (status == indexlib::util::OK) {
        // 3. 调用 adapter 进行格式转换，转换后的新 value 会覆盖旧的 value
        value = _adapter->ConvertIndexPackValue(value, pool);
    }
    
    // 4. 返回结果
    FL_CORETURN status;
}
```
这段代码清晰地展示了装饰器模式的流程：先执行原始功能，然后执行增强功能。这里的 `FL_COAWAIT` 是 Indexlib 中基于协程的异步编程范式，`Get` 方法是一个异步操作，但这不影响适配器模式的核心思想。

### 2.2. `AdapterKVSegmentIterator`：全量遍历的适配

`AdapterKVSegmentIterator` 遵循与 `Reader` 完全相同的设计思想，但应用于全量数据遍历的场景。

**设计思想**:
- **迭代器包装**: 它包装了底层的 `IKVIterator`，确保了遍历的逻辑（如 `HasNext`, `Seek` 等）得以复用。
- **逐条转换**: 它的核心逻辑在 `Next` 方法中。每次从底层迭代器获取一条记录（`Record`）后，它会提取出其中的 `value`，调用 `PackValueAdapter` 进行转换，然后用转换后的新 `value` 替换记录中的旧 `value`。
- **变长/定长处理**: KV 索引的 Value 可以是定长的也可以是变长的。对于变长 Value，其存储格式通常是 `[count][data]`，其中 `count` 是一个编码后的长度信息。`AdapterKVSegmentIterator` 在处理时，需要正确地解码出原始数据部分（`packValue`），转换后再根据目标 Value 是否定长来决定是否需要重新编码新的长度信息。

**关键实现**:
```cpp
// index/kv/AdapterKVSegmentIterator.cpp

Status AdapterKVSegmentIterator::Next(autil::mem_pool::Pool* pool, Record& record)
{
    // 1. 调用底层迭代器获取下一条记录
    auto s = _iterator->Next(pool, record);
    if (!s.IsOK() || record.deleted) {
        return s;
    }
    assert(record.value.size() > 0);

    // 2. 处理变长编码，提取纯数据
    size_t countLen = 0;
    if (!_currentValueFixLength) {
        autil::MultiValueFormatter::decodeCount(record.value.data(), countLen);
    }
    autil::StringView packValue = autil::StringView(record.value.data() + countLen, record.value.size() - countLen);
    
    // 3. 进行格式转换
    auto convertValue = _adapter->ConvertIndexPackValue(packValue, pool);
    
    // 4. 根据目标格式，决定是否需要重新编码长度
    if (_targetValueFixLength) {
        record.value = convertValue;
        return s;
    }

    char encodeLenBuf[4];
    size_t dataSize = autil::MultiValueFormatter::encodeCount(convertValue.size(), encodeLenBuf, 4);
    char* buffer = (char*)pool->allocate(dataSize + convertValue.size());
    memcpy(buffer, encodeLenBuf, dataSize);
    memcpy(buffer + dataSize, convertValue.data(), convertValue.size());
    record.value = autil::StringView(buffer, dataSize + convertValue.size());
    return s;
}
```
通过这种方式，无论是单点查询还是全量遍历，上层应用都能获得格式统一的数据视图，底层的 Schema 差异被完全屏蔽。

### 2.3. `AdapterIgnoreFieldCalculator`：处理被删除字段的“幽灵”

在 Schema 演进中，最棘手的问题之一是字段的“删除后重加”。假设演进路径如下：
- `Schema V1`: `value` 包含字段 `A(int), B(string)`
- `Schema V2`: 删除字段 `B`。此时 `value` 只包含 `A(int)`。
- `Schema V3`: 重新加入字段 `B`，但类型变为 `double`。此时 `value` 包含 `A(int), B(double)`。

当一个使用 `Schema V3` 的应用需要读取一个用 `Schema V1` 创建的 Segment 时，适配器需要将 `V1` 的 `value` 转换成 `V3` 的格式。此时，它不能直接把 `V1` 中的 `B(string)` 拿来用，因为它的类型和含义已经与 `V3` 中的 `B(double)` 完全不同。因此，在转换过程中，必须 **忽略** `V1` 中的字段 `B`。

`AdapterIgnoreFieldCalculator` 就是用来计算这个“忽略列表”的。

**核心逻辑**:
1.  **收集 Schema 历史**: 它首先从 `TabletData` 中获取当前线上所有 Segment 使用过的全部 Schema 版本，并按 `schema_id` 排序。
2.  **逐版本比较**: 它依次比较相邻的两个 Schema 版本（`V(i-1)` 和 `V(i)`）。对于 `V(i-1)` 中的每个字段，它会检查该字段是否存在于 `V(i)` 的 `ValueConfig` 中。如果不存在，就意味着这个字段在 `V(i)` 版本被删除了。
3.  **记录删除信息**: 它将“在哪个版本被删除的字段列表”记录在一个 `map` (`_removeFieldRoadMap`) 中，`key` 是目标 Schema ID，`value` 是被删除的字段名列表。
4.  **计算忽略列表**: 当外部调用 `GetIgnoreFields(beginSchemaId, endSchemaId)` 时，它会遍历 `_removeFieldRoadMap` 中所有位于 `(beginSchemaId, endSchemaId)` 区间内的记录，将所有被删除过的字段名收集起来，形成最终的忽略列表。

**关键实现**:
```cpp
// index/kv/AdapterIgnoreFieldCalculator.cpp

bool AdapterIgnoreFieldCalculator::Init(...) {
    // ... 获取所有相关的 Schema 版本 ...
    auto schemas = tabletData->GetAllTabletSchema(beginSchemaId, endSchemaId);
    if (schemas.size() < 2) { return true; }

    // 遍历相邻的 Schema 版本对
    for (size_t i = 1; i < schemas.size(); i++) {
        auto current = schemas[i - 1];
        auto target = schemas[i];
        std::vector<std::string> removeFields;
        // ... 获取 current 和 target 的 ValueConfig ...
        
        // 找出在 target 中被删除的字段
        for (size_t j = 0; j < currentValueConfig->GetAttributeCount(); j++) {
            auto& attrName = currentValueConfig->GetAttributeConfig(j)->GetAttrName();
            if (fieldSet.find(attrName) == fieldSet.end()) { // 在 target 的字段集合中未找到
                removeFields.push_back(attrName);
            }
        }
        _removeFieldRoadMap.insert(std::make_pair(target->GetSchemaId(), std::move(removeFields)));
    }
    return true;
}

std::vector<std::string> AdapterIgnoreFieldCalculator::GetIgnoreFields(schemaid_t beginSchemaId, schemaid_t endSchemaId) {
    std::set<std::string> ignoreFields;
    // 找到所有在 (begin, end) 区间内被删除过的字段
    auto iter = _removeFieldRoadMap.upper_bound(beginSchemaId);
    for (; iter != _removeFieldRoadMap.end(); iter++) {
        if (iter->first >= endSchemaId) {
            break;
        }
        auto& removeFields = iter->second;
        for (auto& field : removeFields) {
            ignoreFields.insert(field);
        }
    }
    return std::vector<std::string>(ignoreFields.begin(), ignoreFields.end());
}
```
这个计算结果最终会被传递给 `PackValueAdapter`，指导它在进行格式转换时，正确地跳过那些应该被忽略的字段。

---

## 3. 技术风险与考量

1.  **性能开销**: 
    - 适配器层的核心是在线数据转换，这无疑会带来额外的 CPU 开销。`PackValueAdapter` 内部需要解析二进制数据、查找字段、拷贝数据、重新序列化，对于 Value 较大或结构复杂的场景，这个开销可能比较显著。
    - 这种开销是为实现 Schema 灵活性付出的代价。在性能极其敏感的场景，应尽量避免频繁的 Schema 变更，或者在业务低峰期通过“合并（Merge）”操作将所有 Segment 统一到最新的 Schema 版本，从而消除运行时的适配开销。

2.  **逻辑正确性**: 
    - 适配器的逻辑正确性高度依赖于 `PackValueAdapter` 和 `AdapterIgnoreFieldCalculator` 的实现。任何在字段映射、类型转换或忽略字段计算上的 Bug，都可能导致数据错乱或程序崩溃。
    - 对默认值的处理也需要特别小心。当从一个不包含字段 `C` 的旧 Schema 转换到一个包含字段 `C` 的新 Schema 时，适配器需要为新记录的字段 `C` 填充上正确的默认值。

3.  **内存管理**: 
    - 转换后的新 Value 是在 `autil::mem_pool::Pool` 中分配的。上层调用者必须保证这个 `Pool` 的生命周期足够长，至少要覆盖转换后 Value 的使用周期，否则会导致悬挂指针问题。

4.  **不支持的变更**: 
    - 当前的适配器机制主要处理字段的增加和删除。对于更复杂的变更，如修改字段类型（例如 `int` -> `string`），或者修改字段的 MultiValue 属性（单值变多值），可能无法支持或需要更复杂的转换逻辑，这会增加适配器的实现难度和风险。

---

## 4. 总结

Indexlib 的 KV 索引适配器层是一个设计精良的解决方案，它通过应用**装饰器模式**和**适配器模式**，优雅地解决了在线服务中因 Schema 演进而导致的数据异构性问题。

- **解耦与透明**: 它成功地将“数据访问”和“格式转换”两个关注点分离，并对上层应用保持了接口的透明性，使得系统核心的查询逻辑无需关心底层的物理数据细节。
- **对复杂场景的周全考虑**: `AdapterIgnoreFieldCalculator` 的设计表明，该方案不仅仅是简单的字段映射，还深入考虑了 Schema 演进中“字段删除后重加”这类棘手的边界情况，体现了其设计的健壮性。
- **灵活性与性能的权衡**: 该机制为系统带来了极高的灵活性，支持了服务的平滑升级和业务的快速迭代。虽然引入了运行时的性能开销，但这是一个经过深思熟虑的、值得的权衡。

总体而言，KV 索引适配器层是 Indexlib 系统中一个体现深厚工程实践的优秀范例，它为构建一个可长期演进、易于维护的大型索引系统提供了坚实的基础。
