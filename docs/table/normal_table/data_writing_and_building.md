
# Havenask Normal Tablet 数据写入与构建模块深度解析

**涉及文件:**
*   `table/normal_table/NormalTabletWriter.cpp`
*   `table/normal_table/NormalTabletWriter.h`
*   `table/normal_table/NormalTabletParallelBuilder.cpp`
*   `table/normal_table/NormalTabletParallelBuilder.h`
*   `table/normal_table/NormalTabletModifier.cpp`
*   `table/normal_table/NormalTabletModifier.h`
*   `table/normal_table/NormalTabletPatcher.cpp`
*   `table/normal_table/NormalTabletPatcher.h`

## 1. 概述

Havenask `Normal Table` 的数据写入与构建模块是整个索引系统的心脏，它负责接收外部传入的文档，经过一系列处理（重写、分发、构建索引），最终将数据高效、准确地转化为可供检索引擎使用的倒排索引、正排索引、Source、Summary等数据结构。此模块的设计目标是在保证数据一致性的前提下，最大化地提升数据写入的吞吐量和并发性能。

本文将深入剖tuning该模块的四个核心组件：`NormalTabletWriter`、`NormalTabletParallelBuilder`、`NormalTabletModifier` 和 `NormalTabletPatcher`，揭示其内部的工作机制、核心算法、设计哲学以及它们之间如何协同工作，共同完成复杂的数据构建任务。

## 2. 核心组件协同工作流程

这四个组件构成了一个清晰的数据处理流水线（Pipeline），如下图所示：

```mermaid
graph TD
    A[外部调用: Build(IDocumentBatch)] --> B(NormalTabletWriter);
    B --> C{构建模式判断};
    C -- Stream Mode --> D[串行处理];
    C -- Batch Mode --> E[并行处理];

    subgraph "串行处理流程 (Stream Mode)"
        D --> F(分发DocId);
        F --> G(重写文档);
        G --> H(修改文档: NormalTabletModifier);
        H --> I(构建文档: NormalMemSegment);
    end

    subgraph "并行处理流程 (Batch Mode)"
        E --> J(分发DocId);
        J --> K(重写文档);
        K --> L(并行构建: NormalTabletParallelBuilder);
        L --> M[提交构建任务到线程池];
    end

    subgraph "数据恢复与加载 (Reopen/Load)"
        N[Reopen/Load] --> O(NormalTabletPatcher);
        O --> P{加载Patch文件};
        P --> Q(应用Patch到Indexer);
    end

    style B fill:#f9f,stroke:#333,stroke-width:2px
    style L fill:#ccf,stroke:#333,stroke-width:2px
    style H fill:#cfc,stroke:#333,stroke-width:2px
    style O fill:#fcf,stroke:#333,stroke-width:2px
```

- **NormalTabletWriter**: 作为数据写入的统一入口，负责接收 `IDocumentBatch`，并根据配置的构建模式（`Stream` 或 `Batch`）选择不同的处理路径。它是整个流程的协调者和调度者。
- **NormalTabletParallelBuilder**: 在 `Batch` 模式下，`Writer` 将构建任务委托给 `ParallelBuilder`。`ParallelBuilder` 负责将一个 `DocumentBatch` 拆解成针对不同索引（倒排、正排、主键等）的构建任务，并利用线程池进行并行处理，极大地提升了构建效率。
- **NormalTabletModifier**: 在 `Stream` 模式下，或是在数据恢复（Reopen）过程中，`Modifier` 负责处理文档的更新和删除操作。它通过主键查找旧文档，然后在 `DeletionMap` 中标记删除，并对需要更新的字段直接在内存中进行修改（In-place Update）或生成Patch文件。
- **NormalTabletPatcher**: 在 `Reopen` 或数据加载时，`Patcher` 负责加载增量构建或合并产生的 `Patch` 文件，并将这些变更应用到对应的索引中，确保数据的一致性和完整性。

## 3. `NormalTabletWriter`：写入流程的指挥官

`NormalTabletWriter` 是数据写入流程的顶层封装，它屏蔽了底层复杂的构建细节，为上层提供了简洁的 `Build` 接口。

### 3.1. 主要职责

- **接收数据**: 作为写入流程的入口，接收 `IDocumentBatch` 对象。
- **模式选择**: 根据 `TabletOptions` 中配置的构建模式（`BuildMode`），决定是采用串行流式处理还是并行批处理。
- **文档预处理**: 在将文档分发给构建器之前，执行一系列关键的预处理操作，包括：
    - **有效性验证 (`ValidateDocumentBatch`)**: 过滤掉无效或过时的文档（例如，根据 `Locator` 判断）。
    - **DocId 分发 (`DispatchDocIds`)**: 为新增文档分配 `docid`，并处理 `UPDATE` 和 `DELETE` 操作的 `docid` 转换。
    - **文档重写 (`RewriteDocumentBatch`)**: 执行 `DocumentRewriteChain`，例如，根据 `AddToUpdate` 规则将 `ADD` 操作重写为 `UPDATE`。
- **任务调度**:
    - 在 `Stream` 模式下，依次调用 `Modifier` 和 `NormalMemSegment` 的 `Build` 方法，串行处理文档。
    - 在 `Batch` 模式下，将整个 `DocumentBatch` 委托给 `NormalTabletParallelBuilder` 进行并行处理。
- **状态同步**: 在构建完成后，调用 `PostBuildActions` 更新 `TabletData` 和 `NormalTabletInfo` 的状态，如文档总数等。

### 3.2. 核心实现分析

#### 构建模式切换

`NormalTabletWriter::Open` 方法是理解其工作模式的关键。它会根据传入的 `OpenOptions` 来初始化或切换构建模式。

```cpp
// table/normal_table/NormalTabletWriter.cpp

Status NormalTabletWriter::Open(const std::shared_ptr<framework::TabletData>& tabletData,
                                const BuildResource& buildResource, const OpenOptions& openOptions)
{
    // ... 省略部分代码 ...

    // 如果只是更新控制流，则仅切换模式，不重新初始化
    if (openOptions.GetUpdateControlFlowOnly()) {
        if (_parallelBuilder == nullptr) {
            return Status::InternalError("parallel builder is not intialized");
        }
        AUTIL_LOG(INFO, "switch build mode from [%d] to [%d]", (int)(_parallelBuilder->GetBuildMode()),
                  (int)(openOptions.GetBuildMode()));
        return _parallelBuilder->SwitchBuildMode(openOptions.GetBuildMode());
    }
    
    // ... 省略初始化代码 ...

    // 初始化 ParallelBuilder
    _parallelBuilder = std::make_unique<indexlib::table::NormalTabletParallelBuilder>();
    RETURN_IF_STATUS_ERROR(_parallelBuilder->Init(_normalBuildingSegment, _options,
                                                  buildResource.consistentModeBuildThreadPool,
                                                  buildResource.inconsistentModeBuildThreadPool),
                           "init parallel builder failed[%s]", status.ToString().c_str());
    
    // 根据 OpenOptions 切换到指定的构建模式
    RETURN_STATUS_DIRECTLY_IF_ERROR(_parallelBuilder->SwitchBuildMode(openOptions.GetBuildMode()));

    // 如果是并行模式，需要提前准备好所有 SingleBuilder
    if (openOptions.GetBuildMode() != OpenOptions::BuildMode::STREAM) {
        RETURN_IF_STATUS_ERROR(_parallelBuilder->PrepareForWrite(_schema, _buildTabletData),
                               "prepare parallel builder failed");
    }
    
    // ... 省略其他初始化 ...
    return Status::OK();
}
```

这段代码清晰地展示了 `Writer` 如何根据 `OpenOptions` 来管理 `_parallelBuilder` 的状态和构建模式。特别地，`UpdateControlFlowOnly` 选项允许在不中断写入的情况下动态切换构建策略，这为在线服务的灵活性提供了保障。

#### 文档处理流水线

`DoBuildAndReportMetrics` 方法是写入逻辑的核心，它串联起了文档处理的各个阶段。

```cpp
// table/normal_table/NormalTabletWriter.cpp

Status NormalTabletWriter::DoBuildAndReportMetrics(const std::shared_ptr<document::IDocumentBatch>& batch)
{
    // ... 省略异常处理 ...
    
    // 1. 文档校验
    ValidateDocumentBatch(batch.get());

    if (_parallelBuilder->GetBuildMode() != OpenOptions::STREAM) {
        // --- 并行模式 ---
        // 2. 分配 DocId
        DispatchDocIds(batch.get());
        // 3. 重写文档
        status = RewriteDocumentBatch(batch.get());
        RETURN_IF_STATUS_ERROR(status, "rewrite document batch failed");
        ReportBuildDocumentMetrics(batch.get());
        // 4. 委托给 ParallelBuilder 进行并行构建
        status = _parallelBuilder->Build(batch);
        RETURN_IF_STATUS_ERROR(status, "parallel build document batch failed");
        // 5. 更新状态
        PostBuildActions();
    } else {
        // --- 串行模式 ---
        // 2. 将 batch 按主键冲突拆分
        auto subDocBatches = SplitDocumentBatch(batch.get());
        for (auto subDocBatch : subDocBatches) {
            // 3. 分配 DocId
            DispatchDocIds(subDocBatch.get());
            // 4. 重写文档
            status = RewriteDocumentBatch(subDocBatch.get());
            RETURN_IF_STATUS_ERROR(status, "rewrite document batch failed");
            // 5. 修改/删除旧文档
            status = ModifyDocumentBatch(subDocBatch.get());
            RETURN_IF_STATUS_ERROR(status, "modify document batch failed");
            // 6. 构建新文档
            status = _normalBuildingSegment->Build(subDocBatch.get());
            RETURN_IF_STATUS_ERROR(status, "build document batch failed");
            // 7. 更新状态
            PostBuildActions();
        }
        ReportBuildDocumentMetrics(batch.get());
    }
    // ...
    return status;
}
```

这个函数完美地诠释了 `Writer` 的指挥官角色。它定义了清晰的执行流程，并根据模式选择不同的执行路径，确保数据被正确、高效地处理。

### 3.3. 设计动机与技术风险

- **设计动机**:
    - **统一入口**: 提供一个统一、简洁的写入接口，隐藏内部实现的复杂性。
    - **灵活性与可扩展性**: 通过构建模式切换和 `DocumentRewriteChain` 机制，提供了极高的灵活性，方便未来扩展新的构建策略和文档处理逻辑。
    - **职责分离**: `Writer` 专注于流程控制和调度，将具体的构建、修改任务分别委托给 `ParallelBuilder` 和 `Modifier`，符合单一职责原则。

- **技术风险**:
    - **串行模式性能瓶颈**: 在 `Stream` 模式下，所有操作都是串行的，在高并发写入场景下可能会成为性能瓶颈。
    - **DocId 分发逻辑复杂性**: `DispatchDocIds` 的逻辑需要精确处理 `ADD`、`UPDATE`、`DELETE` 等多种操作，代码逻辑复杂，需要有完备的测试来保证其正确性。

## 4. `NormalTabletParallelBuilder`：并行构建的加速引擎

当系统处于 `Batch` 构建模式时，`NormalTabletParallelBuilder` 便登场了。它的核心使命是将一个 `DocumentBatch` 的构建任务分解，并利用多线程并行执行，从而最大化地利用多核 CPU 资源，缩短构建时间。

### 4.1. 主要职责

- **初始化构建器**: 在 `PrepareForWrite` 阶段，根据 `Schema` 初始化所有需要的 `SingleBuilder`，如 `SingleInvertedIndexBuilder`、`SingleAttributeBuilder` 等。
- **任务分解与派发**: 在 `Build` 方法中，为每一种索引类型创建一个或多个 `BuildWorkItem`，并将这些 `WorkItem` 推入 `GroupedThreadPool` 中。
- **线程管理**: 内部通过 `GroupedThreadPool` 来管理和调度构建任务，确保相同索引的构建任务可以被分组执行，减少线程同步开销。
- **内存控制**: 使用 `WaitMemoryQuotaController` 来控制 `DocumentBatch` 的内存使用，防止因 `Batch` 过大导致内存溢出。
- **生命周期管理**: 提供了 `WaitFinish` 方法，用于等待所有后台构建任务完成，确保数据在 `Dump` 前的完整性。

### 4.2. 核心实现分析

#### 构建任务的创建与分发

`Build` 方法是 `ParallelBuilder` 的核心，它展示了如何将一个 `DocumentBatch` 分解为多个并行的 `BuildWorkItem`。

```cpp
// table/normal_table/NormalTabletParallelBuilder.cpp

Status NormalTabletParallelBuilder::Build(const std::shared_ptr<indexlibv2::document::IDocumentBatch>& batch)
{
    // ...
    _buildThreadPool->StartNewBatch();

    // 主键和 DeletionMap 的构建必须同步或在其他任务之前完成，以保证 DocId 分配的正确性
    for (const auto& singleBuilder : _singlePrimaryKeyBuilders) {
        auto buildWorkItem = std::make_unique<index::PrimaryKeyBuildWorkItem>(singleBuilder.get(), batch.get());
        if (_buildMode == OpenOptions::CONSISTENT_BATCH) {
            _buildThreadPool->PushWorkItem(buildWorkItem->Name(), std::move(buildWorkItem));
        } else {
            buildWorkItem->process(); // INCONSISTENT_BATCH 模式下立即执行
        }
    }
    // ... DeletionMap 处理逻辑类似 ...

    // 在 INCONSISTENT_BATCH 模式下，提前更新 Segment 元数据
    if (_buildMode == OpenOptions::INCONSISTENT_BATCH) {
        _normalBuildingSegment->PostBuildActions(lastLocator, maxTimestamp, maxTTL, addDocCount);
    }

    // 为其他所有索引创建并行的 BuildWorkItem
    CreateBuildWorkItems<...>(_singleAttributeBuilders, batch);
    CreateBuildWorkItems<...>(_singleVirtualAttributeBuilders, batch);
    CreateBuildWorkItems<...>(_singleInvertedIndexBuilders, batch);
    // ... 其他索引类型 ...

    // ... 释放文档内存的 Hook ...

    // 在 CONSISTENT_BATCH 模式下，等待所有任务完成后再更新 Segment 元数据
    if (_buildMode == OpenOptions::CONSISTENT_BATCH) {
        _buildThreadPool->WaitCurrentBatchWorkItemsFinish();
        _normalBuildingSegment->PostBuildActions(lastLocator, maxTimestamp, maxTTL, addDocCount);
    }
    // ...
    return Status::OK();
}
```

这段代码揭示了并行构建的关键设计：
1.  **依赖性处理**: 识别出主键和 `DeletionMap` 是后续构建步骤的依赖，并根据构建模式（`CONSISTENT` vs `INCONSISTENT`）决定是同步执行还是异步执行。这体现了对数据一致性和性能之间权衡的深刻理解。
2.  **任务抽象**: `BuildWorkItem` 是一个优秀的设计，它将“构建一个特定索引”这个动作封装成一个独立的、可并行的任务单元。
3.  **模板化**: `CreateBuildWorkItems` 模板函数的运用，极大地简化了为不同索引类型创建 `WorkItem` 的代码，提高了代码的复用性和可维护性。

### 4.3. 设计动机与技术风险

- **设计动机**:
    - **性能最大化**: 核心目标是利用多核 CPU 并行处理能力，大幅提升数据构建的吞吐量，满足海量数据实时写入的需求。
    - **解耦**: 将并行逻辑从 `NormalTabletWriter` 中剥离出来，使得 `Writer` 的逻辑更清晰，同时 `ParallelBuilder` 专注于性能优化。
    - **资源隔离**: 通过 `GroupedThreadPool`，可以对不同类型的构建任务进行分组，为未来更精细化的资源控制（如IO密集型 vs CPU密集型任务）提供了可能性。

- **技术风险**:
    - **线程安全**: 并行构建引入了复杂的线程安全问题。所有 `SingleBuilder` 的实现都必须是线程安全的，任何不当的共享状态访问都可能导致数据不一致或程序崩溃。
    - **死锁与活锁**: 复杂的任务依赖关系和线程同步机制，增加了死锁和活锁的风险，需要仔细设计和严格测试。
    - **调试困难**: 并行程序的调试本身就比串行程序困难得多，定位问题需要更高级的工具和技巧。

## 5. `NormalTabletModifier` & `NormalTabletPatcher`：数据一致性的守护者

`Modifier` 和 `Patcher` 共同构成了 `Normal Table` 数据更新和恢复的基石，它们确保了在各种场景下（实时更新、版本加载）数据的一致性。

### 5.1. `NormalTabletModifier`：实时更新的执行者

`Modifier` 主要用于处理文档的 `UPDATE` 和 `DELETE` 操作。

- **职责**:
    - **删除操作**: 通过 `DeletionMapModifier` 在内存中的位图里标记指定 `docid` 为已删除状态。
    - **更新操作**:
        - **In-place Update**: 对于定长、非索引字段的更新，直接在内存中修改 `Attribute` 的值。这是最高效的更新方式。
        - **Patch Generation**: 对于变长字段、索引字段或是在 `opLog2PatchDir` 模式下，`Modifier` 会生成 `Patch` 文件，记录变更信息。

- **核心实现**:
  `ModifyDocument` 方法是其核心，它首先处理删除，然后对属性和倒排索引进行更新。

  ```cpp
  // table/normal_table/NormalTabletModifier.cpp
  Status NormalTabletModifier::ModifyDocument(document::IDocumentBatch* batch)
  {
      // 1. 先处理删除操作
      auto st = RemoveDocuments(batch);
      RETURN_IF_STATUS_ERROR(st, "remove documents failed ");

      // 2. 更新正排索引 (Attribute)
      assert(_inplaceAttrModifier);
      st = _inplaceAttrModifier->Update(batch);
      RETURN_IF_STATUS_ERROR(st, "update attr failed ");

      // 3. 更新倒排索引 (Inverted Index)
      assert(_inplaceInvertedIndexModifier);
      st = _inplaceInvertedIndexModifier->Update(batch);
      RETURN_IF_STATUS_ERROR(st, "update inverted index failed ");
      return Status::OK();
  }
  ```

### 5.2. `NormalTabletPatcher`：数据恢复的修复师

`Patcher` 在 `Reopen` 或加载新 `Version` 时被调用，负责将 `Patch` 文件中的变更应用到新加载的 `Segment` 中。

- **职责**:
    - **发现 Patch**: 通过 `PatchFileFinder` 找到指定 `Segment` 和 `Schema` 版本对应的所有 `Patch` 文件。
    - **加载 Patch**: 读取 `Patch` 文件的内容。
    - **应用 Patch**:
        - 对于属性 `Patch`，通过 `AttributePatchReader` 读取变更，并更新 `AttributeDiskIndexer`。
        - 对于倒排 `Patch`，通过 `InvertedIndexPatchIteratorCreator` 创建迭代器，遍历 `Patch` 中的词条变更，并更新 `InvertedDiskIndexer`。
        - 对于删除 `Patch`，更新 `DeletionMap`。

- **核心实现**:
  `LoadPatch` 是其入口，它分门别le地为不同类型的索引加载 `Patch`。

  ```cpp
  // table/normal_table/NormalTabletPatcher.cpp
  Status NormalTabletPatcher::LoadPatch(const std::vector<std::shared_ptr<framework::Segment>>& diffSegments,
                                        const framework::TabletData& newTabletData,
                                        const std::shared_ptr<config::ITabletSchema>& schema,
                                        const std::shared_ptr<indexlib::file_system::IDirectory>& opLog2PatchRootDir,
                                        NormalTabletModifier* modifier)
  {
      // ...
      RETURN_IF_STATUS_ERROR(LoadPatchForDeletionMap(diffSegments, newTabletData, schema),
                             "load patch for deletion map failed");
      RETURN_IF_STATUS_ERROR(LoadPatchForAttribute(segmentPairs, schema, opLog2PatchRootDir, modifier),
                             "load patch for attribute indexer failed");
      RETURN_IF_STATUS_ERROR(LoadPatchForInveredIndex(diffSegments, schema, opLog2PatchRootDir, modifier),
                             "load patch for inverted index indexer failed");
      return Status::OK();
  }
  ```

### 5.3. 设计动机与技术风险

- **设计动机**:
    - **读写分离**: `Patch` 机制实现了读写操作的分离。写入操作（更新/删除）只产生 `Patch` 文件，不直接修改已生成的索引，避免了昂贵的写时复制和锁竞争，保证了查询性能的稳定。
    - **数据一致性**: `Patch` 文件作为增量变更的载体，确保了在 `Reopen` 后，新的 `Reader` 能够加载一个包含所有历史变更的、完全一致的数据视图。
    - **原子性**: `Patch` 文件的生成和切换是原子性的，保证了更新操作的完整性。

- **技术风险**:
    - **Patch 文件膨胀**: 如果更新操作非常频繁，可能会产生大量的 `Patch` 文件，不仅占用大量磁盘空间，还会影响 `Reopen` 的速度，因为需要加载和应用所有 `Patch`。这通常需要定期的合并（Merge）操作来解决。
    - **Patch 应用性能**: 在 `Reopen` 时应用大量 `Patch` 可能非常耗时，影响新版本上线的速度。`Patcher` 的实现需要高度优化，以减少这部分开销。
    - **逻辑复杂性**: `Patch` 机制涉及多种索引类型和复杂的变更逻辑，实现起来非常复杂，容易出错。

## 6. 总结

Havenask `Normal Table` 的数据写入与构建模块是一个设计精良、高度优化的系统。它通过 `Writer`、`ParallelBuilder`、`Modifier` 和 `Patcher` 四个核心组件的协同工作，实现了高效、灵活、可靠的数据写入和更新。

- **`NormalTabletWriter`** 作为总指挥，提供了清晰的写入流程和模式切换能力。
- **`NormalTabletParallelBuilder`** 作为加速引擎，通过并行化大幅提升了构建性能。
- **`NormalTabletModifier`** 和 **`NormalTabletPatcher`** 作为一致性守护者，通过 `Patch` 机制确保了数据在实时更新和版本切换过程中的一致性和完整性。

对该模块的深入理解，不仅有助于我们更好地使用和运维 Havenask 系统，也为我们设计类似的高性能数据处理系统提供了宝贵的借鉴和启示。
