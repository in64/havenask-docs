### 1. 概述

本部分代码定义了 `indexlib` 系统中用于不同类型度量数据收集、统计和上报的专用上报器（Reporter）以及基础统计工具。它提供了针对输入值、状态、QPS（每秒查询率）和基于 CPU Slot 的 QPS 等不同场景的度量上报机制，并引入了通用的统计类 `Statistic` 和进度跟踪类 `ProgressMetrics`。此外，`TaggedMetricReporterGroup` 实现了对带有标签的度量上报器的动态管理和批量上报，极大地增强了度量系统的灵活性和可扩展性。

### 2. 功能目标

*   **多样化度量上报**: 提供 `InputMetricReporter`、`StatusMetricReporter`、`QpsMetricReporter` 和 `CpuSlotQpsMetricReporter`，以适应不同业务场景下的度量需求。
*   **基础数据统计**: `Statistic` 类提供线程安全的累加求和与计数功能，支持计算平均值，为上报器提供基础数据。
*   **进度跟踪**: `ProgressMetrics` 类用于跟踪任务的整体进度，包括总值、当前完成比例和开始时间戳。
*   **标签化度量管理**: `TaggedMetricReporterGroup` 允许动态创建和管理带有不同标签组合的度量上报器，并支持批量上报和自动清理不再使用的上报器。
*   **性能优化**: `Statistic` 类通过 `ThreadLocalPtr` 和原子操作优化了多线程环境下的统计性能，减少了锁竞争。
*   **CPU Slot QPS 计算**: `CpuSlotQpsMetricReporter` 结合 CPU 使用率和 CPU Slot 概念，提供更精确的 QPS 计算，尤其适用于资源受限或容器化环境。

### 3. 系统架构与设计动机

该模块的系统架构围绕着不同类型的 `MetricReporter` 和一个通用的 `TaggedMetricReporterGroup` 展开，辅以基础统计类 `Statistic` 和 `ProgressMetrics`。

*   **`MetricReporter` 家族 (Input, Status, Qps, CpuSlotQps)**:
    *   **设计动机**: 不同的度量类型（如瞬时值、累积值、速率）有不同的收集和上报逻辑。例如，`InputMetricReporter` 关注输入值的统计特性（和、数量、平均），`QpsMetricReporter` 关注事件发生的频率。将这些逻辑封装在各自的 Reporter 中，使得业务代码可以根据具体需求选择合适的 Reporter，避免了在 `Metric` 层面处理复杂的统计逻辑。
    *   **核心职责**: 每个 Reporter 都持有 `util::MetricPtr`，负责将内部统计的数据转换为 `kmonitor` 可接受的格式并上报。它们还支持添加 `kmonitor::MetricsTags`，以便为度量数据添加维度信息。
    *   **`Statistic` 的作用**: `InputMetricReporter` 和 `QpsMetricReporter` 内部都使用了 `Statistic` 或 `AccumulativeCounter`（`Statistic` 的简化版，用于累积计数）来收集原始数据，然后在其 `Report()` 方法中计算出需要上报的值（如平均值、QPS）。

*   **`Statistic` 类**: 
    *   **设计动机**: 在多线程环境下进行累积统计（如总和、总数）时，直接对共享变量进行操作会引入锁竞争，影响性能。`Statistic` 旨在提供一个高性能、线程安全的累积统计机制。
    *   **核心实现**: 它采用了“线程局部存储（Thread-Local Storage, TLS）+ 合并”的策略。每个线程维护自己的局部统计数据（`ThreadStat`），当线程结束或需要获取全局统计数据时，再将线程局部数据合并到全局的原子变量 `_mergedNum` 和 `_mergedSum` 中。这种设计避免了高频的全局锁，显著提高了并发性能。`ThreadStat` 结构体还考虑了缓存行对齐（`posix_memalign`），以避免伪共享（False Sharing）问题，进一步优化了多核环境下的性能。

*   **`ProgressMetrics` 类**: 
    *   **设计动机**: 在长时间运行的任务中，跟踪任务的完成进度是一个常见的需求。`ProgressMetrics` 提供了一个简单的结构来表示和更新任务进度，方便上层模块查询和展示。
    *   **核心职责**: 维护任务的总量、当前完成比例和开始时间戳。它是一个轻量级的辅助类，不直接涉及 `kmonitor` 上报，但其数据可以被其他 Reporter 转换为度量上报。

*   **`TaggedMetricReporterGroup` 模板类**: 
    *   **设计动机**: 在实际系统中，很多度量需要根据不同的维度（标签）进行细分。例如，不同表、不同分区、不同操作的 QPS。如果为每个标签组合都手动声明一个 `Metric`，代码会变得非常冗余且难以管理。`TaggedMetricReporterGroup` 旨在解决这个问题，它能够动态地为不同的标签组合创建和管理对应的 `MetricReporter` 实例。
    *   **核心实现**: 它是一个模板类，可以管理任何 `MetricReporter` 类型的实例。它内部维护一个 `std::map<uint64_t, MetricReporterPtr>`，其中 `uint64_t` 是标签哈希值，`MetricReporterPtr` 是对应的 Reporter 实例。当业务代码请求一个带有特定标签组合的 Reporter 时，它会计算标签的哈希值，如果已存在则返回现有实例，否则创建一个新的实例。它还实现了 `Report()` 方法，可以遍历所有管理的 Reporter 并触发它们的上报。更重要的是，它包含了一个 `CleanUselessReporter()` 机制，通过检查 `shared_ptr` 的 `use_count()` 来自动清理不再被外部引用的 Reporter，防止内存泄漏和无效度量持续存在。

整个模块通过这种分层和专业化的设计，实现了度量数据从原始采集、统计、标签化到最终上报的完整链路，并兼顾了性能、灵活性和可维护性。

### 4. 核心逻辑与算法

#### 4.1 `Statistic` 类的线程安全统计

`Statistic` 类是多线程环境下高性能累积统计的关键。其核心思想是避免频繁的全局锁，而是利用线程局部存储（TLS）和原子操作。

*   **`ThreadStat` 结构体**: 每个线程通过 `autil::ThreadLocalPtr` 拥有一个 `ThreadStat` 实例。`ThreadStat` 包含该线程局部累积的 `num` 和 `sum`。它还持有指向全局原子变量 `_mergedNum` 和 `_mergedSum` 的指针。
*   **`Record(uint64_t value)` 方法**: 当一个线程调用 `Record` 时，它只更新自己线程局部 `ThreadStat` 中的 `num` 和 `sum`。这是一个无锁操作，非常高效。
    ```cpp
    // util/metrics/Statistic.h
    void Record(uint64_t value)
    {
        auto threadStatPtr = GetThreadStat();
        threadStatPtr->sum += value;
        threadStatPtr->num += 1;
    }
    ```
*   **`MergeThreadStat(void* ptr)` 静态方法**: 这是 `autil::ThreadLocalPtr` 的回调函数，当一个线程的 `ThreadStat` 对象被销毁时（例如线程退出），该函数会被调用。它将线程局部的 `num` 和 `sum` 原子性地累加到全局的 `_mergedNum` 和 `_mergedSum` 中。
    ```cpp
    // util/metrics/Statistic.h
    static void MergeThreadStat(void* ptr)
    {
        auto threadStat = static_cast<ThreadStat*>(ptr);
        *(threadStat->mergedNum) += threadStat->num;
        *(threadStat->mergedSum) += threadStat->sum;
        delete threadStat;
    }
    ```
*   **`GetStatData()` 方法**: 当需要获取全局统计数据时，该方法会遍历所有活跃线程的 `ThreadStat` 数据（通过 `_threadStat->Fold`），将其累加到 `data` 中，然后再加上已经合并到全局原子变量中的数据。最后计算平均值。
    ```cpp
    // util/metrics/Statistic.h
    StatData GetStatData()
    {
        StatData data;
        _threadStat->Fold(
            [](void* entryPtr, void* res) {
                auto globalData = static_cast<StatData*>(res);
                auto threadData = static_cast<ThreadStat*>(entryPtr);
                globalData->num += threadData->num;
                globalData->sum += threadData->sum;
            },
            &data);
        data.num += _mergedNum.load(std::memory_order_relaxed);
        data.sum += _mergedSum.load(std::memory_order_relaxed);
        if (data.num == 0) {
            data.avrage = 0;
        } else {
            data.avrage = static_cast<double>(data.sum) / data.num;
        }
        return data;
    }
    ```
    这种设计在读多写少的场景下表现优异，因为写操作（`Record`）是无锁的，而读操作（`GetStatData`）虽然需要遍历，但频率通常远低于写操作。

#### 4.2 `TaggedMetricReporterGroup` 的动态管理与清理

`TaggedMetricReporterGroup` 实现了对标签化上报器的动态生命周期管理。

*   **`DeclareMetricReporter(const std::map<std::string, std::string>& tagMap)`**: 
    *   根据传入的 `tagMap` 创建或获取一个 `MetricReporter` 实例。
    *   首先创建一个临时的 `reporter`，并为其添加标签，然后计算标签的哈希值 `tagHash`。
    *   使用 `_threadMutex` 加锁，保证 `_reporterMap` 的线程安全。
    *   在 `_reporterMap` 中查找 `tagHash`。如果找到，则返回已存在的 Reporter；否则，将新创建的 Reporter 插入 `_reporterMap`。
    ```cpp
    // util/metrics/TaggedMetricReporterGroup.h
    MetricReporterPtr DeclareMetricReporter(const std::map<std::string, std::string>& tagMap)
    {
        MetricReporterPtr reporter(new T);
        reporter->Init(mTypeMetricItemMetric);
        for (auto kv : tagMap) {
            if (!_limitedTagKeys.empty() && _limitedTagKeys.find(kv.first) == _limitedTagKeys.end()) {
                continue;
            }
            reporter->AddTag(kv.first, util::KmonitorTagvNormalizer::GetInstance()->Normalize(kv.second));
        }
        uint64_t tagHash = reporter->GetTagHashCode();

        autil::ScopedLock lock(_threadMutex);
        auto iter = _reporterMap.find(tagHash);
        if (iter != _reporterMap.end()) {
            reporter = iter->second;
        } else {
            _reporterMap.insert(std::make_pair(tagHash, reporter));
        }
        return reporter;
    }
    ```
*   **`Report()`**: 遍历 `_reporterMap` 中的所有 Reporter，并调用它们的 `Report()` 方法，实现批量上报。
*   **`CleanUselessReporter()`**: 这是自动清理机制的核心。它遍历 `_reporterMap`，检查每个 `MetricReporterPtr` 的 `use_count()`。如果 `use_count()` 大于 1，表示该 Reporter 仍然被外部引用（除了 `_reporterMap` 自身），则不进行清理。如果 `use_count()` 等于 1，表示只有 `_reporterMap` 自身持有该 Reporter 的引用，说明外部已经不再使用它，此时将其从 `_reporterMap` 中擦除，从而释放资源。
    ```cpp
    // util/metrics/TaggedMetricReporterGroup.h
    void CleanUselessReporter()
    {
        autil::ScopedLock lock(_threadMutex);
        for (auto iter = _reporterMap.begin(); iter != _reporterMap.end();) {
            if (iter->second.use_count() > 1) { // 如果被外部引用，则不清理
                iter++;
                continue;
            }
            iter = _reporterMap.erase(iter); // 否则清理
        }
    }
    ```
    这种基于 `shared_ptr::use_count()` 的清理策略是一种常见的智能指针使用模式，用于实现资源的自动回收。

#### 4.3 `CpuSlotQpsMetricReporter` 的特殊 QPS 计算

`CpuSlotQpsMetricReporter` 旨在提供一种更精确的 QPS 计算方式，尤其是在 CPU 资源受限的环境中。它考虑了进程实际获得的 CPU 时间。

*   **`Init()`**: 初始化时，它会尝试从环境变量 `INDEXLIB_QUOTA_CPU_SLOT_NUM` 获取 CPU Slot 数量。如果未设置，则使用系统硬件并发数。它还会初始化 `autil::metric::ProcessCpu` 来获取进程的 CPU 使用率。
*   **`Report()`**: 在上报时，它不仅考虑事件数量的增量，还考虑了在这段时间内进程实际获得的 CPU 时间。它将 QPS 定义为 `(事件增量) / (实际CPU时间)`，其中实际 CPU 时间通过 `(时间间隔 * CPU Slot 数量 * CPU 使用率)` 计算。这使得 QPS 更能反映在给定 CPU 资源下的实际处理能力。
    ```cpp
    // util/metrics/MetricReporter.h
    void Report() noexcept
    {
        int64_t currentTs = autil::TimeUtility::currentTime();
        auto cur = GetTotalCount();
        double usage = _procCpu.getUsage(); // 获取进程CPU使用率
        if (_metric) {
            auto tsGap = currentTs - _latestTimestamp;
            int64_t totalCpuSlotTime = (int64_t)(tsGap * _cpuSlotCount * usage); // 计算实际CPU时间
            if (totalCpuSlotTime > 0) {
                double convertedCpuSlotNum = static_cast<double>(totalCpuSlotTime) / (1000 * 1000);
                _metric->IncreaseQps((double)(cur - _last) / convertedCpuSlotNum, _kmonTags.get()); // 计算并上报QPS
            }
        }
        _last = cur;
        _latestTimestamp = currentTs;
    }
    ```

### 5. 技术栈与关键实现细节

*   **C++11/14**: 广泛使用了 C++11/14 的特性，如 `std::shared_ptr`、`noexcept`、`std::atomic`、`std::map`、`std::set`、`std::vector` 等。
*   **`autil` 库**: 深度依赖 `autil` 库，包括 `autil::Log`、`autil::ScopedLock`、`autil::ThreadLocalPtr`、`autil::TimeUtility`、`autil::EnvUtil`、`autil::StringUtil` 以及 `autil::metric::ProcessCpu`。
*   **`kmonitor` 客户端库**: 作为底层度量上报的接口，`MetricReporter` 家族最终通过 `util::MetricPtr` 调用 `kmonitor::MetricsReporter` 进行上报。
*   **线程安全**: `Statistic` 使用 `std::atomic` 和 `autil::ThreadLocalPtr` 实现无锁或低锁的并发统计。`TaggedMetricReporterGroup` 使用 `autil::RecursiveThreadMutex` 保护其内部 `_reporterMap` 的并发访问。
*   **内存对齐**: `Statistic::ThreadStat` 使用 `posix_memalign` 进行内存对齐，以优化多核处理器上的缓存性能，避免伪共享。
*   **宏定义**: `Monitor.h` 中定义的宏（如 `IE_INIT_LOCAL_INPUT_METRIC`）简化了 `MetricReporter` 的初始化。
*   **标签规范化**: `MetricReporter` 家族在添加标签时，会调用 `util::KmonitorTagvNormalizer::GetInstance()->Normalize(v)` 对标签值进行规范化，确保符合 `kmonitor` 的命名要求。

### 6. 可能的技术风险

*   **`shared_ptr::use_count()` 的局限性**: `CleanUselessReporter()` 依赖 `shared_ptr::use_count()` 来判断 Reporter 是否被外部引用。虽然在大多数情况下有效，但 `use_count()` 并非完全线程安全，且在某些复杂场景（如弱引用、循环引用）下可能不准确。如果外部存在 `weak_ptr` 引用，`use_count()` 不会增加，可能导致 Reporter 被过早清理。然而，对于本场景，只要外部持有 `shared_ptr`，`use_count()` 就会大于 1，因此风险相对可控。
*   **性能开销**: 尽管 `Statistic` 优化了并发写入，但 `GetStatData()` 仍然需要遍历活跃线程的 TLS 数据，在高并发且频繁读取统计数据的场景下，这可能成为性能瓶颈。`TaggedMetricReporterGroup` 的 `Report()` 方法会遍历所有 Reporter，如果 Reporter 数量非常庞大，也会有性能开销。
*   **CPU Slot QPS 的准确性**: `CpuSlotQpsMetricReporter` 的计算依赖于 `ProcessCpu` 获取的 CPU 使用率和 `INDEXLIB_QUOTA_CPU_SLOT_NUM` 配置。这些值的准确性直接影响 QPS 的计算结果。如果 `ProcessCpu` 无法准确获取进程 CPU 使用率，或者 `INDEXLIB_QUOTA_CPU_SLOT_NUM` 配置不当，计算结果可能失真。
*   **标签哈希冲突**: `TaggedMetricReporterGroup` 使用标签哈希值作为 `map` 的键。虽然哈希冲突的概率较低，但一旦发生，可能导致不同的标签组合被错误地映射到同一个 Reporter，从而导致数据混淆。通常，`kmonitor::MetricsTags::Hashcode()` 应该足够健壮。
*   **内存管理**: `TaggedMetricReporterGroup` 的自动清理机制虽然有效，但如果 Reporter 被外部长时间持有引用，即使不再活跃，也不会被清理，可能导致内存占用持续增长。需要确保业务代码及时释放不再需要的 Reporter 引用。

### 7. 核心代码示例

#### 7.1 `Statistic` 类的核心实现

```cpp
// util/metrics/Statistic.h
class Statistic
{
public:
    struct StatData {
        uint64_t sum = 0;
        uint64_t num = 0;
        double avrage = 0.0;
        StatData() = default;
    };

private:
    struct ThreadStat {
        std::uint_fast64_t num;
        std::uint_fast64_t sum;

        std::atomic_uint_fast64_t* mergedNum;
        std::atomic_uint_fast64_t* mergedSum;

        ThreadStat(uint64_t _num, uint64_t _sum, std::atomic_uint_fast64_t* _mergedNum,
                   std::atomic_uint_fast64_t* _mergedSum)
            : num(_num)
            , sum(_sum)
            , mergedNum(_mergedNum)
            , mergedSum(_mergedSum)
        {
        }

        void* operator new(size_t size)
        {
            void* p;
            int ret = posix_memalign(&p, size_t(128), size);
            if (ret != 0) {
                throw std::bad_alloc();
            }
            return p;
        }

        void operator delete(void* p) { free(p); }
    };

public:
    Statistic() : _threadStat(new autil::ThreadLocalPtr(&MergeThreadStat)) {}

    void Record(uint64_t value)
    {
        auto threadStatPtr = GetThreadStat();
        threadStatPtr->sum += value;
        threadStatPtr->num += 1;
    }

    StatData GetStatData()
    {
        StatData data;
        _threadStat->Fold(
            [](void* entryPtr, void* res) {
                auto globalData = static_cast<StatData*>(res);
                auto threadData = static_cast<ThreadStat*>(entryPtr);
                globalData->num += threadData->num;
                globalData->sum += threadData->sum;
            },
            &data);
        data.num += _mergedNum.load(std::memory_order_relaxed);
        data.sum += _mergedSum.load(std::memory_order_relaxed);
        if (data.num == 0) {
            data.avrage = 0;
        } else {
            data.avrage = static_cast<double>(data.sum) / data.num;
        }
        return data;
    }

public:
    static void MergeThreadStat(void* ptr)
    {
        auto threadStat = static_cast<ThreadStat*>(ptr);
        *(threadStat->mergedNum) += threadStat->num;
        *(threadStat->mergedSum) += threadStat->sum;
        delete threadStat;
    }

private:
    std::unique_ptr<autil::ThreadLocalPtr> _threadStat;
    std::atomic_uint_fast64_t _mergedNum {0};
    std::atomic_uint_fast64_t _mergedSum {0};
};
```

#### 7.2 `TaggedMetricReporterGroup` 的 `DeclareMetricReporter` 和 `CleanUselessReporter`

```cpp
// util/metrics/TaggedMetricReporterGroup.h
template <typename T>
class TaggedMetricReporterGroup
{
public:
    typedef std::shared_ptr<T> MetricReporterPtr;
    typedef std::map<uint64_t, MetricReporterPtr> ReporterMap;

public:
    MetricReporterPtr DeclareMetricReporter(const std::map<std::string, std::string>& tagMap)
    {
        MetricReporterPtr reporter(new T);
        reporter->Init(mTypeMetricItemMetric);
        for (auto kv : tagMap) {
            if (!_limitedTagKeys.empty() && _limitedTagKeys.find(kv.first) == _limitedTagKeys.end()) {
                continue;
            }
            reporter->AddTag(kv.first, util::KmonitorTagvNormalizer::GetInstance()->Normalize(kv.second));
        }
        uint64_t tagHash = reporter->GetTagHashCode();

        autil::ScopedLock lock(_threadMutex);
        auto iter = _reporterMap.find(tagHash);
        if (iter != _reporterMap.end()) {
            reporter = iter->second;
        } else {
            _reporterMap.insert(std::make_pair(tagHash, reporter));
        }
        return reporter;
    }

private:
    void CleanUselessReporter()
    {
        autil::ScopedLock lock(_threadMutex);
        for (auto iter = _reporterMap.begin(); iter != _reporterMap.end();) {
            if (iter->second.use_count() > 1) {
                iter++;
                continue;
            }
            iter = _reporterMap.erase(iter);
        }
    }

private:
    ReporterMap _reporterMap;
    mutable autil::RecursiveThreadMutex _threadMutex;
};
```

#### 7.3 `CpuSlotQpsMetricReporter` 的 `Report` 方法

```cpp
// util/metrics/MetricReporter.h
class CpuSlotQpsMetricReporter
{
public:
    void Report() noexcept
    {
        int64_t currentTs = autil::TimeUtility::currentTime();
        auto cur = GetTotalCount();
        double usage = _procCpu.getUsage();
        if (_metric) {
            auto tsGap = currentTs - _latestTimestamp;
            int64_t totalCpuSlotTime = (int64_t)(tsGap * _cpuSlotCount * usage);
            if (totalCpuSlotTime > 0) {
                double convertedCpuSlotNum = static_cast<double>(totalCpuSlotTime) / (1000 * 1000);
                _metric->IncreaseQps((double)(cur - _last) / convertedCpuSlotNum, _kmonTags.get());
            }
        }
        _last = cur;
        _latestTimestamp = currentTs;
    }

private:
    util::MetricPtr _metric;
    autil::metric::ProcessCpu _procCpu;
    volatile int64_t _last = 0;
    volatile int64_t _latestTimestamp = 0;
    volatile uint32_t _cpuSlotCount = 0;
    AccumulativeCounter _counter;
    std::shared_ptr<kmonitor::MetricsTags> _kmonTags;
};
```
