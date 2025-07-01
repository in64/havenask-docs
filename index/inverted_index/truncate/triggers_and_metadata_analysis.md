
# Indexlib 截断模块触发器与元数据（Triggers & Metadata）源码分析

**涉及文件:**
* `index/inverted_index/truncate/TruncateTrigger.h`
* `index/inverted_index/truncate/DefaultTruncateTrigger.h`
* `index/inverted_index/truncate/TruncateMetaTrigger.h`
* `index/inverted_index/truncate/TruncateMetaTrigger.cpp`
* `index/inverted_index/truncate/TruncateTriggerCreator.h`
* `index/inverted_index/truncate/TruncateTriggerCreator.cpp`
* `index/inverted_index/truncate/TruncateMetaReader.h`
* `index/inverted_index/truncate/TruncateMetaReader.cpp`
* `index/inverted_index/truncate/TimeStrategyTruncateMetaReader.h`
* `index/inverted_index/truncate/TimeStrategyTruncateMetaReader.cpp`
* `index/inverted_index/truncate/TruncateMetaFileReaderCreator.h`
* `index/inverted_index/truncate/TruncateMetaFileReaderCreator.cpp`

---

## 1. 概述

在 Indexlib 的倒排索引截断体系中，并非所有词（Term）的倒排链都需要被截断。只有当倒排链的长度或其他特征满足特定条件时，截断流程才会被启动。**截断触发器（Truncate Trigger）** 和 **截断元数据（Truncate Metadata）** 正是负责管理这一决策过程和相关信息的关键组件。

- **截断触发器 (TruncateTrigger)**: 它的职责非常明确：判断一个给定的词（Term）是否需要进行截断。这个决策是截断流程的入口，只有当触发器“点头”后，后续的收集、排序、写入等昂贵操作才会被执行。触发器通常基于简单的规则，如文档频率（DF）是否超过某个阈值。

- **截断元数据 (TruncateMetaReader)**: 在某些高级截断策略中，系统需要利用之前截断产生的信息来指导后续的操作。例如，在“按历史排序截断，按时间范围过滤”的场景中，系统需要知道某个词在上次截断时，被保留的文档的最低排序值是多少。这些信息就被记录在截断元数据中。`TruncateMetaReader` 负责读取和解析这些元数据文件，并为触发器和过滤器提供查询服务。

- **元数据文件**: 这是一个简单的文本文件，通常每行记录一个词（`DictKeyInfo`）和它对应的截断阈值（例如，排序分数的最小值或最大值）。这个文件在一次全量合并（Full Merge）的截断过程中由 `SingleTruncateIndexWriter` 生成，并在后续的增量合并（Incremental Merge）中被读取和使用。

- **创建器 (Creator)**: `TruncateTriggerCreator` 和 `TruncateMetaFileReaderCreator` 是工厂类，它们根据配置动态创建合适的触发器和元数据读取器实例，将这些组件与截断流程的其他部分解耦。

这套机制的设计目标是：
1.  **效率**: 通过触发器避免对短的、无需截断的倒排链进行不必要处理，节省计算资源。
2.  **策略复用**: 通过元数据，使得增量构建过程可以复用全量构建时的截断结果，保证截断标准的一致性，并实现更复杂的过滤优化。

---

## 2. 核心设计与实现机制

### 2.1. 截断触发器 (TruncateTrigger) 体系

触发器体系结构简单，主要由一个接口和两个实现组成。

#### 2.1.1. `TruncateTrigger` 接口

`TruncateTrigger` 定义了所有触发器的统一接口。

```cpp
// index/inverted_index/truncate/TruncateTrigger.h

struct TruncateTriggerInfo {
    // ... 包含 DictKeyInfo 和 df_t (文档频率) ...
};

class TruncateTrigger
{
public:
    // ...
    // 核心决策方法
    virtual bool NeedTruncate(const TruncateTriggerInfo& info) const = 0;
    // 为触发器设置元数据读取器
    virtual void SetTruncateMetaReader(const std::shared_ptr<TruncateMetaReader>& reader) {}
};
```
- `NeedTruncate`: 这是唯一的核心方法，接收一个 `TruncateTriggerInfo` 对象，该对象包含了词的 `DictKeyInfo` 和文档频率 `df`。触发器根据这些信息返回 `true`（需要截断）或 `false`（不需要截断）。
- `SetTruncateMetaReader`: 这是一个可选方法，允许为触发器注入一个元数据读取器，用于实现更复杂的、依赖历史截断信息的触发逻辑。

#### 2.1.2. `DefaultTruncateTrigger`：基于阈值的触发

这是最常用、最简单的触发器。它的逻辑是：当一个词的文档频率（DF）超过预设的阈值（`threshold`）时，就触发截断。

**关键实现**:
```cpp
// index/inverted_index/truncate/DefaultTruncateTrigger.h

class DefaultTruncateTrigger : public TruncateTrigger
{
public:
    DefaultTruncateTrigger(uint64_t truncThreshold) : _threshold(truncThreshold) {}
    // ...
    bool NeedTruncate(const TruncateTriggerInfo& info) const override 
    {
        return info.GetDF() > (df_t)_threshold;
    }
private:
    uint64_t _threshold;
};
```
这种策略简单有效，适用于绝大多数“只处理高频词”的场景。

#### 2.1.3. `TruncateMetaTrigger`：基于元数据的触发

这种触发器用于实现依赖历史截断结果的策略。它的决策不依赖于文档频率，而是查询 `TruncateMetaReader`，看当前的词是否存在于元数据中。

**核心逻辑**:
如果一个词在上次（通常是全量）截断时被处理过（即被截断了），那么它的信息就会被记录在元数据文件中。`TruncateMetaTrigger` 的 `NeedTruncate` 方法会查询 `_metaReader`，如果能找到该词的记录，就意味着这次也需要对它进行处理（例如，为了应用新的过滤条件或与增量数据合并）。

**关键实现**:
```cpp
// index/inverted_index/truncate/TruncateMetaTrigger.cpp

bool TruncateMetaTrigger::NeedTruncate(const TruncateTriggerInfo& info) const
{
    if (!_metaReader) {
        AUTIL_LOG(WARN, "truncate meta reader not exist in index");
        return false;
    }
    // 决策依据是元数据中是否存在该词
    return _metaReader->IsExist(info.GetDictKey());
}
```
这种触发器是实现“`truncate_meta`”策略的关键。它确保了在增量合并时，只有那些在全量构建时被截断过的词才会被再次处理，保证了截断范围的一致性。

### 2.2. 截断元数据 (Truncate Metadata) 体系

元数据体系负责截断信息的持久化和读取，是实现跨构建周期的截断策略的基础。

#### 2.2.1. 元数据文件格式

元数据文件是一个简单的制表符分隔的文本文件（TSV）。每一行代表一个被截断的词，格式如下：

```
<dict_key_string>\t<value_string>\n
```

- `<dict_key_string>`: 词的字符串表示，由 `DictKeyInfo::ToString()` 生成。
- `<value_string>`: 截断点的排序值。例如，如果按销量降序截断保留 1000 个，这里就记录第 1000 个商品的销量值。

这个文件由 `SingleTruncateIndexWriter::WriteTruncateMeta` 方法生成。

#### 2.2.2. `TruncateMetaReader`：元数据读取与解析

`TruncateMetaReader` 负责解析元数据文件，并将其内容加载到内存中的一个 `map` 中，供上层查询。

**核心逻辑**:
- **Open**: `Open` 方法接收一个文件读取器（`FileReader`），然后使用 `LineReader` 逐行读取文件内容。对于每一行，它调用 `AddOneRecord` 进行解析。
- **AddOneRecord**: 解析每一行，将第一部分转换成 `DictKeyInfo`，第二部分转换成 `int64_t` 的排序值。然后，根据排序方向（`_desc`），将这个词和它的排序范围（`Range`）存入 `_dict` 中。
- **Lookup**: 提供查询接口。外部组件（如 `TruncateMetaTrigger` 或 `IDocFilterProcessor`）可以通过 `Lookup` 方法查询某个词的截断范围。

**关键实现**:
```cpp
// index/inverted_index/truncate/TruncateMetaReader.cpp

Status TruncateMetaReader::Open(const std::shared_ptr<file_system::FileReader>& file)
{
    // ... 使用 LineReader 逐行读取 ...
    while (reader.NextLine(line, ec)) {
        RETURN_IF_STATUS_ERROR(AddOneRecord(line), "add one record failed");
    }
    // ...
    return Status::OK();
}

Status TruncateMetaReader::AddOneRecord(const std::string& line)
{
    // ... 解析 key 和 value ...
    if (_desc) { // 降序
        // 保留的是 [value, max] 区间
        _dict[key] = std::pair<dictkey_t, int64_t>(value, std::numeric_limits<int64_t>::max());
    } else { // 升序
        // 保留的是 [min, value] 区间
        _dict[key] = std::pair<dictkey_t, int64_t>(std::numeric_limits<int64_t>::min(), value);
    }
    return Status::OK();
}

bool TruncateMetaReader::Lookup(const index::DictKeyInfo& key, int64_t& min, int64_t& max) const
{
    auto it = _dict.find(key);
    if (it == _dict.end()) {
        return false;
    }
    min = it->second.first;
    max = it->second.second;
    return true;
}
```

#### 2.2.3. `TimeStrategyTruncateMetaReader`：时间策略的特殊实现

这是一个 `TruncateMetaReader` 的子类，专门用于处理按时间过滤的场景（例如，保留最近一周的数据）。

**核心逻辑**:
它重写了 `Lookup` 方法。当查询一个词时，它不仅会考虑元数据文件中记录的排序阈值，还会结合全局的时间范围（`_minTime`, `_maxTime`）来动态调整最终的过滤范围。例如，对于降序排序，它会将元数据中的排序下限（`it->second.first`）与全局的时间下限（`_minTime`）取较小值，从而扩大了需要保留的文档范围，以包含新加入的、符合时间要求的文档。

**关键实现**:
```cpp
// index/inverted_index/truncate/TimeStrategyTruncateMetaReader.h

bool TimeStrategyTruncateMetaReader::Lookup(const index::DictKeyInfo& key, int64_t& min, int64_t& max) const
{
    auto it = _dict.find(key);
    if (it == _dict.end()) {
        return false;
    }

    if (_desc) {
        min = std::min(_minTime, it->second.first);
        max = _maxTime;
    } else {
        min = _minTime;
        max = std::max(_maxTime, it->second.second);
    }
    return true;
}
```
这个子类是实现“过滤条件在增量构建时会变化”这一类复杂需求的关键。

---

## 3. 技术风险与考量

1.  **元数据一致性**:
    - 整个 `truncate_meta` 策略的正确性高度依赖于元数据文件与索引数据的一致性。如果元数据文件丢失、损坏，或者与生成它的那次全量索引版本不匹配，可能会导致增量截断的结果出现严重偏差。
    - 系统缺乏对元数据文件版本和有效性的校验机制，这是一个潜在的风险点。

2.  **内存占用**:
    - `TruncateMetaReader` 会将整个元数据文件加载到内存中的 `std::map`。如果需要截断的词非常多（例如，对整个词典进行截断），这个 `map` 可能会占用大量内存。对于超大规模的索引，这可能成为一个瓶颈。

3.  **性能**: 
    - `TruncateMetaReader` 使用 `std::map` 作为底层存储，查询复杂度是 O(logN)，其中 N 是被截断的词的数量。对于高频调用的 `Lookup` 方法，这通常是足够高效的。但如果 N 达到千万甚至亿级别，性能可能会有所下降。
    - 文件 I/O 是 `Open` 方法的主要开销，但由于这通常在合并任务的启动阶段一次性完成，所以对整体性能影响不大。

4.  **策略的局限性**:
    - 当前的元数据只记录了截断点的一个单一阈值。这对于单维排序是足够的，但对于多维排序，这种元数据就无法完全描述截断边界。例如，对于 `ORDER BY price, sales`，元数据可能只记录了 `price` 的阈值，而丢失了在相同 `price` 下 `sales` 的阈值信息。这限制了 `truncate_meta` 策略在多维排序场景下的应用精度。

---

## 4. 总结

Indexlib 的触发器与元数据模块为截断流程提供了必要的控制和优化机制。其设计哲学是 **“先判断，后执行”** 和 **“复用历史，指导未来”**。

- **触发器 (Trigger)** 扮演了 **“守卫”** 的角色，通过简单的规则（如 DF 阈值）或更复杂的元数据查询，有效地防止了不必要的计算开销，是系统性能的重要保障。
- **元数据 (Metadata)** 机制则是一种巧妙的 **“状态持久化”** 方案。它将一次全量截断的结果（即截断边界）持久化下来，使得后续的增量截断能够在该状态的基础上继续进行，不仅保证了截断标准的一致性，还为动态过滤等高级优化策略提供了可能。

`DefaultTruncateTrigger` 和 `TruncateMetaTrigger` 的分离，清晰地体现了两种不同的截断哲学：一种是无状态的、基于当前信息的决策；另一种是有状态的、基于历史信息的决策。这种设计的划分使得整个截断体系既能处理简单的场景，也能支持复杂的、跨构建周期的策略，展示了良好的系统设计和抽象能力。
