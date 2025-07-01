
# Indexlib 索引数据生命周期管理与清理机制深度剖析

**涉及文件:**
*   `framework/cleaner/InMemoryIndexCleaner.h`
*   `framework/cleaner/InMemoryIndexCleaner.cpp`
*   `framework/cleaner/OnDiskIndexCleaner.h`
*   `framework/cleaner/OnDiskIndexCleaner.cpp`

## 1. 系统概述

在 Indexlib 引擎中，索引数据以“段”(Segment)的形式组织，多个段共同构成一个特定的索引“版本”(Version)。随着数据的不断写入、合并(Merge)和优化，系统会持续产生新的段和版本。为了防止磁盘空间无限增长并维持系统性能，必须有一套机制来清理那些不再被任何活跃查询或后续合并任务引用的“无用”数据。这就是索引数据生命周期管理与清理机制的核心职责。

本文档深入分析的 `InMemoryIndexCleaner` 和 `OnDiskIndexCleaner` 是实现这一核心职责的两个关键组件。它们分别针对不同的场景，共同构成了 Indexlib 回收索引数据的完整解决方案。

- **`InMemoryIndexCleaner`**: 主要负责**运行时**的增量数据清理。它被设计为在系统正常服务期间（Online 模式）由后台线程周期性调用。其核心目标是清理那些**逻辑上**已经废弃，但物理上可能还存在于文件系统中的版本文件(version.x)和索引段目录。它通过与 `TabletReaderContainer` 交互，精确判断哪些版本和段已不再被任何活跃的 `TabletReader` 引用，从而进行安全的逻辑删除。在特定配置下，它也能触发物理删除。

- **`OnDiskIndexCleaner`**: 专注于**离线或全量**场景下的磁盘数据清理。它的应用场景通常是数据恢复、索引切换或运维手动触发的磁盘整理。与 `InMemoryIndexCleaner` 不同，`OnDiskIndexCleaner` 的清理策略不完全依赖于运行时的 `TabletReader` 状态，而是基于一个更宏观的“保留策略”，例如“保留最新的N个版本”以及一个明确的“必须保留的版本列表”。它会全面扫描本地磁盘上的所有文件，与保留策略进行比对，从而计算出需要被物理删除的文件和目录集合。

这两者一个偏向于“在线、逻辑、增量”清理，一个偏向于“离线、物理、全量”清理，共同确保了 Indexlib 在不同运行模式下都能有效地管理磁盘空间。

## 2. 核心设计理念与架构

该模块的设计体现了对数据安全性和系统鲁棒性的高度重视，其核心设计理念包括：

*   **安全第一原则**: 清理操作，尤其是物理删除，是高危操作。两个 Cleaner 的设计都将数据安全放在首位。`InMemoryIndexCleaner` 通过 `TabletReaderContainer` 确认资源无引用后才操作；`OnDiskIndexCleaner` 则通过“保留版本”白名单机制，确保任何正在使用或未来可能被用到的版本数据不会被误删。特别是它有一个保护机制：如果请求保留的版本在本地不存在，清理将被中止，以防止在索引分发过程中误删文件。

*   **逻辑删除与物理删除分离**: `InMemoryIndexCleaner` 主要执行逻辑删除（通过 `Directory` 的抽象接口），将文件或目录标记为待删除，而真正的物理删除可以被延迟。这减少了在线清理任务对系统 I/O 的冲击。`OnDiskIndexCleaner` 则主要负责物理删除，进行彻底的磁盘空间回收。

*   **版本为中心的清理模型**: Indexlib 的所有数据都围绕“版本”进行组织。清理逻辑也严格遵循这一模型。一个段是否无用，取决于它是否还属于任何一个需要保留的版本。一个版本文件是否无用，取决于它本身是否在保留列表中。这种模型清晰且不易出错。

*   **面向复杂文件系统的设计**: 清理逻辑需要处理各种文件和目录类型，包括段目录 (`segment_xxx`)、版本文件 (`version.xxx`)、EntryTable 文件 (`entry_table.xxx`)，以及可能存在于 `fence` 目录下的特殊结构。代码通过路径解析和命名规范来识别不同类型的文件，并应用不同的处理策略。

### 架构与流程

**InMemoryIndexCleaner 工作流程:**

```
+-----------------------+
| ResourceCleaner::Run()|
+-----------------------+
           |
           v
+------------------------------------+
|      InMemoryIndexCleaner::Clean()     |
|------------------------------------|
| - (Acquire cleaner lock)           |
| - DoClean(logical directory)       |
|   - CleanUnusedVersions()          |
|     - List all version files       |
|     - Get in-use versions from     |
|       TabletReaderContainer        |
|     - Remove versions not in use   |
|   - CleanUnusedSegments()          |
|     - Get latest version           |
|     - List all segment dirs        |
|     - Get in-use segments from     |
|       TabletReaderContainer        |
|     - Remove segments not in latest|
|       version AND not in use       |
| - if (cleanPhysicalFiles)          |
|   - DoClean(physical directory)    |
+------------------------------------+
```

**OnDiskIndexCleaner 工作流程:**

```
+------------------------------------+
| External Trigger (e.g., Admin Cmd) |
+------------------------------------+
           |
           v
+------------------------------------+
|      OnDiskIndexCleaner::Clean()       |
|------------------------------------|
| - CollectReservedFilesAndDirs()    |
|   - Input: reservedVersionIds      |
|   - Load all local versions        |
|   - Check if all reserved versions exist (Safety Check) |
|   - Combine reserved IDs with keep-N policy & reader-in-use versions -> final keepVersions & keepSegments |
| - List all files/dirs in root path |
| - CaculateNeedCleanFileList()      |
|   - Compare all files against keepFiles, keepSegments, keepVersions |
| - CleanFiles()                     |
|   - Separate files and dirs        |
|   - Remove files first             |
|   - Remove empty dirs later        |
+------------------------------------+
```

## 3. 关键实现细节与代码解读

### 3.1 `InMemoryIndexCleaner`: 运行时的增量守护者

`InMemoryIndexCleaner` 的核心在于 `DoClean` 方法，它分别调用 `CleanUnusedSegments` 和 `CleanUnusedVersions` 来完成任务。

#### 清理无用段 (`CleanUnusedSegments`)

```cpp
// framework/cleaner/InMemoryIndexCleaner.cpp

Status InMemoryIndexCleaner::CleanUnusedSegments(const std::shared_ptr<indexlib::file_system::Directory>& dir)
{
    fslib::FileList segments;
    // 1. 收集所有不再使用的段目录
    Status status = CollectUnusedSegments(dir, &segments);
    // ... error handling ...

    // 2. 同步文件系统元数据，确保后续删除操作的可见性
    auto [st, f] = dir->GetIDirectory()->Sync(/*wait finish*/ true).StatusWith();
    // ... error handling ...
    f.wait();

    // 3. 遍历并删除收集到的无用段目录
    for (size_t i = 0; i < segments.size(); ++i) {
        indexlib::file_system::RemoveOption removeOption = indexlib::file_system::RemoveOption::MayNonExist();
        st = dir->GetIDirectory()->RemoveDirectory(segments[i], removeOption).Status();
        // ... error handling ...
        TABLET_LOG(INFO, "segment [%s][%s] cleaned", dir->GetOutputPath().c_str(), segments[i].c_str());
    }
    // ...
    return Status::OK();
}

Status InMemoryIndexCleaner::CollectUnusedSegments(const std::shared_ptr<indexlib::file_system::Directory>& dir,
                                                   fslib::FileList* segments)
{
    // a. 获取当前最新版本
    Version latestVersion;
    Status st = VersionLoader::GetVersion(dir, INVALID_VERSIONID, &latestVersion);
    // ...

    // b. 列出所有段目录
    fslib::FileList tmpSegments;
    st = VersionLoader::ListSegment(dir, &tmpSegments);
    // ...

    for (size_t i = 0; i < tmpSegments.size(); ++i) {
        segmentid_t segId = PathUtil::GetSegmentIdByDirName(tmpSegments[i]);
        // 跳过未来才产生的段 (通常不应该发生)
        if (segId > latestVersion.GetLastSegmentId()) {
            continue;
        }
        // c. 如果段在最新版本中，则保留
        if (latestVersion.HasSegment(segId)) {
            continue;
        }
        // d. 如果段仍被某个 TabletReader 使用，则保留
        if (_tabletReaderContainer->IsUsingSegment(segId)) {
            continue;
        }
        // e. 否则，该段是无用的
        segments->emplace_back(tmpSegments[i]);
    }
    return Status::OK();
}
```

**代码分析:**
- **清理条件**: `CollectUnusedSegments` 中的逻辑非常严谨。一个段被判定为“无用”必须同时满足三个条件：
    1.  它不是一个“未来”的段。
    2.  它不属于当前最新的 `Version`。
    3.  没有任何一个 `TabletReader` 实例（即使是老的 `TabletReader`）正在使用它。
- **协同工作**: 这清晰地展示了 `InMemoryIndexCleaner` 是如何与 `TabletReaderContainer` 协同工作的。`TabletReaderContainer` 维护了所有活跃 `TabletReader` 使用的段的集合，为清理决策提供了关键依据。

### 3.2 `OnDiskIndexCleaner`: 离线场景的磁盘管家

`OnDiskIndexCleaner` 的逻辑更为复杂，因为它需要处理更全面的情况和更强的保护机制。其核心是 `Clean(const std::vector<versionid_t>& reservedVersions)` 方法。

#### 收集保留文件 (`CollectReservedFilesAndDirs`)

这是清理决策的核心，它决定了哪些文件最终需要被保留下来。

```cpp
// framework/cleaner/OnDiskIndexCleaner.cpp

Status
OnDiskIndexCleaner::CollectReservedFilesAndDirs(const std::shared_ptr<indexlib::file_system::IDirectory>& rootDir,
                                                const std::set<versionid_t>& reservedVersionIds,
                                                std::set<std::string>* keepFiles, std::set<segmentid_t>* keepSegments,
                                                std::set<versionid_t>* keepVersions, bool* isAllReservedVersionExist)
{
    // 1. 收集本地存在的所有版本信息
    std::vector<Version> releatedVersions;
    RETURN_IF_STATUS_ERROR(
        CollectAllRelatedVersion(rootDir, reservedVersionIds, &releatedVersions, isAllReservedVersionExist),
        "collect releated version failed");
    // 安全检查：如果外部要求的保留版本在本地不存在，则中止清理
    if (!*isAllReservedVersionExist) {
        return Status::OK();
    }

    // 2. 根据保留策略计算最终要保留的版本ID集合
    std::set<versionid_t> reservedVersionIdSet;
    size_t currentKeepCount = 0;
    // 从最新版本开始反向遍历
    for (auto iter = releatedVersions.rbegin(); iter != releatedVersions.rend(); ++iter) {
        auto versionId = iter->GetVersionId();
        // a. 保留最新的 N 个版本 (_keepVersionCount)
        if (currentKeepCount < _keepVersionCount) {
            reservedVersionIdSet.insert(versionId);
            currentKeepCount++;
        }
        // b. 如果版本在外部指定的保留列表中，则保留
        if (reservedVersionIds.count(versionId) > 0) {
            reservedVersionIdSet.insert(versionId);
        }
    }

    // ... (省略了与 TabletReaderContainer 交互的逻辑，它会进一步合并运行时正在使用的版本)

    // 3. 根据最终保留的版本，收集所有需要保留的段ID
    for (const auto& version : releatedVersions) {
        if (reservedVersionIdSet.count(version.GetVersionId()) > 0) {
            for (const auto& segment : version) {
                keepSegments->insert(segment.segmentId);
            }
        }
    }

    // 4. 记录最终保留的版本ID
    for (const auto& versionId : reservedVersionIdSet) {
        keepVersions->insert(versionId);
    }
    return Status::OK();
}
```

**代码分析:**
- **多重保留策略**: `OnDiskIndexCleaner` 的保留策略是复合的，它综合了：
    1.  **外部强制保留**: `reservedVersions` 参数，通常由上层业务逻辑（如部署系统）传入。
    2.  **保留最新N个**: `_keepVersionCount` 参数，这是一个通用的、防止历史版本被过快清理的策略。
    3.  **运行时保留**: 通过 `TabletReaderContainer` 获取当前正在被查询使用的版本和段。
- **安全检查先行**: 在执行任何计算之前，`isAllReservedVersionExist` 的检查是第一道防线，极大地提升了清理操作的安全性，防止了在分布式环境下由于节点间状态不一致导致的严重问题。

#### 计算与执行清理 (`CaculateNeedCleanFileList` & `CleanFiles`)

在收集完所有需要保留的文件、目录、段和版本后，`CaculateNeedCleanFileList` 会遍历磁盘上的所有文件，凡是不在任何一个“保留集合”中的，都会被加入到待清理列表。`CleanFiles` 则负责执行最后的删除操作，它会先删除文件，再尝试删除目录（如果目录已空），这种顺序可以避免删除非空目录的错误。

## 4. 技术风险与考量

1.  **文件系统扫描性能**: `OnDiskIndexCleaner` 需要列出指定目录下的所有文件和目录。对于包含数百万个文件的巨大索引，这个 `ListDir` 操作本身可能会非常耗时，并对文件系统造成较大压力。在设计上，清理任务应该在系统负载较低的时段执行。

2.  **分布式一致性**: 在分布式索引场景下，清理操作需要特别小心。`OnDiskIndexCleaner` 的安全检查（`isAllReservedVersionExist`）部分缓解了这个问题，但一个更鲁棒的系统可能需要一个中心化的元数据服务来协调所有节点的清理决策，以确保全局一致性。

3.  **原子性问题**: 文件系统的删除操作通常不是原子的。如果在 `CleanFiles` 执行过程中发生机器宕机，可能会导致部分文件被删除，而另一些文件残留，形成一个不一致的状态。虽然 Indexlib 的加载流程通常能处理这种中间状态，但这仍然是一个潜在的风险。

## 5. 总结

`InMemoryIndexCleaner` 和 `OnDiskIndexCleaner` 是 Indexlib 存储引擎的“清道夫”，它们从不同维度、针对不同场景，共同承担着回收无效索引数据的关键职责。`InMemoryIndexCleaner` 通过与运行时组件的紧密协作，实现了高效、安全的在线增量清理。而 `OnDiskIndexCleaner` 则提供了一套强大的、基于策略的离线全量清理机制，保证了磁盘空间的长期健康。对这两个组件的深入理解，是掌握 Indexlib 数据生命周期管理和存储优化的核心。
