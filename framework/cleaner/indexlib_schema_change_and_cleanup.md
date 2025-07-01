
# Indexlib Schema 变更之索引淘汰 (`DropIndex`) 清理机制深度剖析

**涉及文件:**
*   `framework/cleaner/DropIndexCleaner.h`
*   `framework/cleaner/DropIndexCleaner.cpp`

## 1. 系统概述

在搜索引擎和数据库系统的长期演进中，数据模式（Schema）的变更是一项常规但高风险的操作。一个常见的 Schema 变更场景是“下线索引”（Drop Index）：即决定不再需要某个或某些旧的索引，以节省存储空间和提高写入性能。然而，在一个已经包含大量历史数据的系统中，如何安全、高效地移除这些不再需要的索引数据，是一个必须妥善处理的挑战。

`DropIndexCleaner` 正是 Indexlib 中为应对这一挑战而设计的专用组件。它不负责日常的版本或段清理，其**唯一且核心的职责**是：当系统应用一个新的 Schema，而这个新的 Schema 中不再包含某些旧 Schema 中定义的索引时，`DropIndexCleaner` 负责将这些被“淘汰”的索引对应的数据从物理存储中清理掉。

与 `InMemoryIndexCleaner` 和 `OnDiskIndexCleaner` 关注于整个版本（Version）和段（Segment）的生命周期不同，`DropIndexCleaner` 的操作粒度更细，它深入到**每个段（Segment）的内部**，根据新旧 Schema 的差异，精确地识别并删除段内属于被淘汰索引的那些子目录或文件。

这个过程通常发生在一次“alter table”或类似的 Schema 变更操作之后。系统加载了新的 `TabletSchema`，并希望将线上数据迁移至此新 Schema。`DropIndexCleaner` 在此过程中被调用，完成数据的“瘦身”。

## 2. 核心设计理念与架构

`DropIndexCleaner` 的设计简洁而专注，其核心理念可以概括为：

*   **Schema 驱动**: 所有的清理决策都严格基于新旧两个 Schema 的对比。它不关心版本、不关心 `TabletReader` 的引用状态，唯一的依据是：一个索引是否存在于新的目标 Schema (`targetSchema`) 中。这种设计使得其职责非常单一，逻辑清晰，不易出错。

*   **逐段扫描与清理**: Indexlib 的数据是按段组织的，每个段都相对独立，并且拥有自己生成时所依据的 Schema 信息。`DropIndexCleaner` 的实现遵循了这一结构，它会遍历 `TabletData` 中所有的段，对每一个段都独立地执行一次“差量清理”。这种方式天然地支持了增量处理和并行化（尽管当前实现是串行的）。

*   **逻辑删除优先**: `CleanIndexInLogical` 方法明确地使用了逻辑删除选项 (`logicalDelete = true`)。这意味着在执行清理时，它仅仅是在文件系统的元数据中将相关目录标记为已删除，而不会立即触发大量的物理 I/O。这对于在线服务至关重要，因为它能最大限度地减少清理操作对正常读写请求的性能冲击。真正的物理删除可以由后续的合并（Merge）操作或离线的清理工具来完成。

*   **静态工具类设计**: `DropIndexCleaner` 被设计成一个只包含静态方法的工具类 (`private autil::NoCopyable`)，它自身不存储任何状态。所有操作所需的信息（如 `TabletData`、`targetSchema`、`rootDirectory`）都通过参数传入。这种设计使得它易于使用和测试，调用方无需关心其生命周期管理。

### 架构与流程

`DropIndexCleaner` 的工作流程非常直接：

```
+------------------------------------+
|      Schema Change Trigger         |
| (e.g., alter table operation)      |
+------------------------------------+
                 |
                 v
+------------------------------------+
|   DropIndexCleaner::CleanIndexInLogical()  |
|------------------------------------|
| - Input: originTabletData,         |
|          targetSchema,             |
|          rootDirectory             |
|------------------------------------|
| - Create a slice of all segments   |
|   from originTabletData            |
|                                    |
| - for each segment in slice:       |
|   - Get segment's own schema       |
|   - if segment's schema is older   |
|     than targetSchema:             |
|     - for each indexConfig in      |
|       segment's schema:            |
|       - if index not in targetSchema:|
|         - Get index's paths        |
|         - for each path:           |
|           - RemoveDirectory(path,  |
|               logicalDelete=true)  |
+------------------------------------+
```

## 3. 关键实现细节与代码解读

`DropIndexCleaner` 的核心功能由 `CleanIndexInLogical` 方法实现。

```cpp
// framework/cleaner/DropIndexCleaner.cpp

Status DropIndexCleaner::CleanIndexInLogical(const std::shared_ptr<TabletData>& originTabletData,
                                             const std::shared_ptr<config::ITabletSchema>& targetSchema,
                                             const std::shared_ptr<indexlib::file_system::IDirectory>& rootDirectory)
{
    // 获取 TabletData 中所有段的快照
    auto slice = originTabletData->CreateSlice();
    // 设置删除选项为逻辑删除，且允许目录不存在（以增加鲁棒性）
    auto removeOption = indexlib::file_system::RemoveOption::MayNonExist();
    removeOption.logicalDelete = true;

    // 遍历每一个段
    for (auto segment : slice) {
        auto segmentSchema = segment->GetSegmentSchema();
        // 如果段自身的 Schema 比目标 Schema 还要新或一样，说明它已经是按新 Schema 生成的，无需处理
        if (segmentSchema->GetSchemaId() >= targetSchema->GetSchemaId()) {
            continue;
        }

        // 获取该段对应的物理目录
        auto [status, segmentDir] =
            rootDirectory->GetDirectory(segment->GetSegmentDirectory()->GetLogicalPath()).StatusWith();
        RETURN_IF_STATUS_ERROR(status, "get segment dir failed");

        // 获取段自身 Schema 中定义的所有索引配置
        auto indexConfigs = segmentSchema->GetIndexConfigs();
        for (auto indexConfig : indexConfigs) {
            // 在目标 Schema 中查找同名同类型的索引
            if (!targetSchema->GetIndexConfig(indexConfig->GetIndexType(), indexConfig->GetIndexName())) {
                // 如果找不到，说明该索引已被淘汰
                // 获取该索引对应的所有存储路径（一个索引可能对应多个文件或目录）
                auto paths = indexConfig->GetIndexPath();
                for (auto indexPath : paths) {
                    // 在段目录内，逻辑删除该索引的路径
                    auto status = segmentDir->RemoveDirectory(indexPath, removeOption).Status();
                    AUTIL_LOG(INFO, "logical remove directory [%s]",
                              (segmentDir->GetLogicalPath() + "/" + indexPath).c_str());
                    RETURN_IF_STATUS_ERROR(status, "remove logical directory failed");
                }
            }
        }
    }
    return Status::OK();
}
```

**代码分析:**

- **`segmentSchema->GetSchemaId() >= targetSchema->GetSchemaId()`**: 这是一个重要的优化和保护逻辑。`SchemaId` 是一个单调递增的值，用于标识 Schema 的版本。如果一个段是基于一个较新的 Schema 生成的，那么它自然已经符合了目标 Schema 的结构（或者比目标更新），无需再进行清理。这避免了对新数据进行不必要的操作。

- **`segment->GetSegmentSchema()`**: 体现了 Indexlib 的一个关键设计：每个段都内嵌了其构建时所使用的 Schema 信息。这使得即使在跨越多个 Schema 版本的混合数据环境中，系统也能准确地理解每个数据片段的结构，从而实现精确的操作。

- **`indexConfig->GetIndexPath()`**: 一个逻辑上的索引（如一个倒排索引）在物理上可能由多个文件和目录构成（例如，词典文件、倒排链文件、Posting 文件等）。`GetIndexPath()` 封装了这种复杂性，返回了所有需要被清理的底层路径，确保了清理的彻底性。

- **`segmentDir->RemoveDirectory(...)`**: 清理操作是在 `segmentDir`（即段目录）的上下文中执行的，而不是在根目录。这确保了清理操作的范围被严格限制在单个段内，不会意外地删除其他段的数据，增强了操作的安全性。

## 4. 技术风险与考量

1.  **事务性与回滚**: 当前的实现不是事务性的。如果在遍历和删除过程中发生错误或中断，系统将处于一个中间状态：部分段的旧索引数据被清除了，而另一部分则没有。虽然 Indexlib 的加载机制通常能容忍这种不一致性（查询时对于被删除的索引会返回未找到），但在设计上缺乏原子性。对于要求严格一致性的场景，可能需要引入更复杂的事务机制或“两阶段提交”模式。

2.  **对合并（Merge）策略的影响**: 逻辑删除旧索引后，这些被删除的空间并不会立即回收。它需要依赖后续的合并操作来重写段数据，从而在物理上消除这些“空洞”。因此，Schema 变更后，可能需要主动触发一次全量合并（Full Merge）来尽快回收磁盘空间，否则清理的效果将无法立刻体现。

3.  **依赖 Schema 信息的准确性**: 整个 `DropIndexCleaner` 的正确性完全建立在每个段的 `SegmentSchema` 信息以及 `targetSchema` 的准确性之上。任何 Schema 信息的损坏或不一致都可能导致错误地删除或保留数据。

## 5. 总结

`DropIndexCleaner` 是 Indexlib 框架中一个高度专一化、目标明确的清理工具。它通过对比新旧 Schema，以逻辑删除的方式，安全、高效地移除了存量数据中不再需要的索引部分。其设计充分利用了 Indexlib “段内嵌 Schema”的核心特性，实现了对数据结构的精确操作。虽然它不涉及复杂的生命周期管理或引用计数，但它在保障系统平滑演进、实现数据“瘦身”和节省成本方面扮演着不可或- [ ] 缺的关键角色。理解 `DropIndexCleaner` 的工作原理，对于掌握 Indexlib 的 Schema 变更和数据维护机制至关重要。
