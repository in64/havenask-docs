
# Indexlib 远程访问：数据修改与增补 (Data Patching)

**涉及文件:**
* `indexlib/partition/remote_access/partition_patcher.h`
* `indexlib/partition/remote_access/partition_patcher.cpp`
* `indexlib/partition/remote_access/attribute_data_patcher.h`
* `indexlib/partition/remote_access/attribute_data_patcher.cpp`
* `indexlib/partition/remote_access/attribute_data_patcher_typed.h`
* `indexlib/partition/remote_access/index_data_patcher.h`
* `indexlib/partition/remote_access/index_data_patcher.cpp`

## 1. 功能概述

在 `Indexlib` 的应用场景中，经常需要对已有的索引数据进行修改或增补，尤其是在 `Schema` 发生变更（例如，增加一个新的属性字段或索引字段）时。这个过程被称为 `Alter Table` 或 `Schema` 演化。数据修改（Data Patching）模块正是为了支持这一核心功能而设计的。

该模块提供了一套机制，允许用户针对一个已有的、不可变的 `OfflinePartition`，生成一套“补丁”数据。这些补丁数据独立于原始数据存储，但在逻辑上与原始数据结合，共同构成一个符合新 `Schema` 的、完整的索引视图。其核心目标是：**在不重写整个索引的情况下，高效地为现有数据增补新的字段信息。**

具体来说，该模块主要负责以下任务：

*   **识别变更**：通过比较新旧两个 `Schema`，精确地计算出需要新增的属性（Attribute）和索引（Index）。
*   **提供修改器（Patcher）**：为每个需要新增的属性和索引，提供一个对应的 `Data Patcher` 实例（`AttributeDataPatcher` 或 `IndexDataPatcher`）。
*   **驱动数据生成**：上层应用（例如，一个 `MapReduce` 任务）可以使用这些 `Patcher`，逐条文档（`docid`）地追加新字段的数据。
*   **数据持久化**：`Patcher` 内部封装了数据的编码、压缩和文件写入逻辑，将生成的补丁数据以 `Indexlib` 标准的格式写入指定的补丁目录中。

## 2. 系统架构与设计

该模块的架构可以分为三层：顶层的 `PartitionPatcher` 作为总协调器，中层的 `AttributeDataPatcher` 和 `IndexDataPatcher` 负责处理特定类型的字段，底层的 `AttributePatchDataWriter`（在另一模块中分析）则负责具体的文件写入。

### 2.1. `PartitionPatcher`：变更的总管理者

`PartitionPatcher` 是数据修改功能的入口和总控制器。它负责初始化整个 `Patch` 过程，并根据 `Schema` 变更情况，创建和管理下属的各种 `Data Patcher`。

#### 设计动机

*   **集中管理变更逻辑**：将 `Schema` 差异计算、变更字段识别、`Patcher` 创建等顶层逻辑集中在 `PartitionPatcher` 中，使得上层调用者无需关心 `Schema` 变更的细节。用户只需提供旧的 `Partition` 和新的 `Schema`，即可获得一个准备就绪的 `Patch` 环境。
*   **隔离不同字段的 Patch 过程**：一个 `Schema` 变更可能涉及多个字段。`PartitionPatcher` 为每个变更的字段创建一个独立的 `Patcher` 实例，使得对不同字段的数据追加过程可以独立进行，互不干扰。
*   **与分区资源解耦**：`PartitionPatcher` 的初始化依赖于 `PartitionResourceProvider` 提供的 `OfflinePartition` 和插件管理器等资源，但它本身不直接拥有这些资源。这种设计使得资源管理和数据修改的职责分离，符合单一职责原则。

#### 核心逻辑与实现

1.  **初始化 (`Init`)**：
    *   接收一个 `OfflinePartition` 实例（代表旧数据）、一个新的 `IndexPartitionSchema`、一个用于存放补丁数据的 `patchDir` 以及一个插件管理器。
    *   核心步骤是调用 `SchemaDiffer::CalculateAlterFields`。这个静态方法会比较新旧 `Schema`，找出所有被新增或修改的属性和索引，并将它们的 `Config` 对象分别存入 `mAlterAttributes` 和 `mAlterIndexes` 成员变量中。这是整个 `Patch` 过程的基础。
    *   它还会保存 `OfflinePartition` 的 `PartitionData` 和合并 IO 配置（`MergeIOConfig`），这些信息将在创建下层 `Patcher` 时使用。

    ```cpp
    // indexlib/partition/remote_access/partition_patcher.cpp
    bool PartitionPatcher::Init(const OfflinePartitionPtr& partition, const IndexPartitionSchemaPtr& newSchema,
                                const file_system::DirectoryPtr& patchDir, const plugin::PluginManagerPtr& pluginManager)
    {
        // ... 参数校验和成员赋值 ...

        string errorMsg;
        auto originSchema = mOldSchema;
        // ... 处理带有多步操作的 Schema ...

        // 计算需要变更的字段
        if (!SchemaDiffer::CalculateAlterFields(originSchema, mNewSchema, mAlterAttributes, mAlterIndexes, errorMsg)) {
            IE_LOG(ERROR, "calculate alter fields fail:%s!", errorMsg.c_str());
            return false;
        }
        return true;
    }
    ```

2.  **创建 `Patcher` (`CreateSingleAttributePatcher`, `CreateSingleIndexPatcher`)**：
    *   这些方法根据字段名（`attrName` 或 `indexName`）和 `segmentId` 创建具体的 `Data Patcher`。
    *   它们会首先检查请求的字段是否确实在 `Init` 阶段计算出的 `mAlterAttributes` 或 `mAlterIndexes` 列表中，确保只为需要变更的字段创建 `Patcher`。
    *   然后，它们会为目标 `segment` 在 `patchDir` 下创建对应的子目录（例如 `segment_0_level_0/`）。
    *   接着，实例化一个 `AttributeDataPatcher` 或 `IndexDataPatcher`，并调用其 `Init` 方法，将字段的 `Config`、目标 `segment` 的目录、总文档数等信息传递下去。
    *   `CreateSingleAttributePatcher` 内部会根据属性的类型（定长、变长、单值、多值）和数据类型（`int`, `float`, `string` 等），通过模板类 `AttributeDataPatcherTyped` 来实例化一个具体的 `Patcher`，这个过程是类型安全的。

### 2.2. `AttributeDataPatcher`：属性数据的增补器

`AttributeDataPatcher` 是负责为单个 `segment` 的单个属性字段生成补丁数据的核心类。它是一个抽象基类，具体的实现由模板子类 `AttributeDataPatcherTyped<T>` 完成。

#### 设计动机

*   **封装属性数据生成逻辑**：将属性数据的编码、追加、默认值处理、Null 值处理等逻辑封装起来，对上层提供简洁的 `AppendFieldValue` 接口。
*   **类型安全**：通过模板类 `AttributeDataPatcherTyped<T>`，将不同数据类型的处理逻辑在编译期确定，避免了运行时的类型转换和错误。
*   **生命周期管理**：`AttributeDataPatcher` 在其生命周期内管理着底层的 `AttributePatchDataWriter`。其 `Close` 方法确保了在 `Patch` 结束时，所有缓冲的数据被刷入磁盘，并且文件被正确关闭。

#### 核心逻辑与实现

1.  **初始化 (`Init`)**：
    *   接收 `AttributeConfig`、`MergeIOConfig`、目标 `segment` 目录和总文档数。
    *   创建一个 `AttributeConvertor`，用于在字符串形式的输入值和内部二进制存储值之间进行转换。
    *   关键步骤是调用虚方法 `DoInit`，在 `AttributeDataPatcherTyped` 子类中，`DoInit` 会根据属性是定长还是变长，来决定创建 `SingleValuePatchDataWriter` 还是 `VarNumPatchDataWriter`。

2.  **数据追加 (`AppendFieldValue`, `AppendEncodedValue`, `AppendNullValue`)**：
    *   `AppendFieldValue(const string& valueStr)` 是最上层的接口。它接收一个字符串表示的值。
    *   内部会调用 `mAttrConvertor->Encode()` 将字符串转换为编码后的二进制 `StringView`。
    *   然后调用纯虚方法 `AppendEncodedValue`，这个方法由 `AttributeDataPatcherTyped` 实现，最终会调用底层 `DataWriter` 的 `AppendValue` 方法。
    *   `AppendNullValue` 用于支持可为空的字段，它会直接调用底层 `DataWriter` 的 `AppendNullValue`。
    *   每次追加都会使内部的 `mPatchedDocCount` 计数器加一。

3.  **关闭 (`Close`)**：
    *   这是一个非常重要的方法，必须在处理完一个 `segment` 的所有文档后调用。
    *   它会检查 `mPatchedDocCount` 是否等于 `mTotalDocCount`。如果不等，说明数据不完整，会抛出致命错误。
    *   如果 `mPatchedDocCount` 为 0（即一次 `Append` 都没调用过），它会调用 `AppendAllDocByDefaultValue`，用字段的默认值或 `Null` 值填充所有文档。
    *   最后，调用底层 `mPatchDataWriter->Close()`，完成文件写入和关闭。

    ```cpp
    // indexlib/partition/remote_access/attribute_data_patcher_typed.h
    template <typename T>
    class AttributeDataPatcherTyped : public AttributeDataPatcher
    {
    public:
        // ...
        void Close() override
        {
            if (!mPatchDataWriter) {
                return;
            }

            if (mPatchedDocCount == 0) {
                AppendAllDocByDefaultValue(); // 处理未 Patch 的情况
            }
            if (mPatchedDocCount > 0 && mPatchedDocCount < mTotalDocCount) {
                INDEXLIB_FATAL_ERROR(InconsistentState, "patchDocCount [%u] not equal to totalDocCount [%u]",
                                     mPatchedDocCount, mTotalDocCount); // 保证数据完整性
            }
            mPatchDataWriter->Close();
            mPatchDataWriter.reset();
        }
        // ...
    };
    ```

### 2.3. `IndexDataPatcher`：索引数据的增补器

`IndexDataPatcher` 负责为单个 `segment` 的单个索引字段（特指倒排索引）生成补丁数据。

#### 设计动机

*   **复用倒排索引构建逻辑**：与属性数据不同，倒排索引的构建过程要复杂得多，涉及到分词、词典创建、倒排链表写入等。`IndexDataPatcher` 的设计巧妙地复用了 `Indexlib` 中已有的 `IndexWriter` 体系，而不是重新实现一套倒排构建逻辑。
*   **模拟文档处理流程**：它通过内部维护一个 `IndexDocument` 对象，模拟了在线构建时文档的处理流程。上层应用通过 `AddField` 添加字段内容，通过 `EndDocument` 触发一次写入，这个接口与 `Indexlib` 的文档处理接口保持了一致性。

#### 核心逻辑与实现

1.  **初始化 (`Init`)**：
    *   接收 `IndexConfig`、目标 `segment` 目录和总文档数。
    *   创建一个 `IndexFieldConvertor`，用于将输入的字符串字段值转换为 `IndexDocument` 中的 `Field` 对象。
    *   核心是调用 `IndexWriterFactory::CreateIndexWriter` 创建一个 `IndexWriter` 实例。这个 `IndexWriter` 就是 `Indexlib` 用于构建倒排索引的核心组件。所有后续的数据都会被写入这个 `Writer`。

2.  **数据追加 (`AddField`, `EndDocument`)**：
    *   `AddField(const string& fieldValue, fieldid_t fieldId)`: 接收一个（通常是分词后、以分隔符连接的）字符串，通过 `IndexFieldConvertor` 将其转换并添加到内部的 `mIndexDoc` 对象中。
    *   `EndDocument()`: 当一个文档的所有字段都通过 `AddField` 添加完毕后，调用此方法。
        *   它将 `mIndexDoc` 中缓存的所有 `Field` 添加到 `mIndexWriter` 中。
        *   调用 `mIndexWriter->EndDocument(*mIndexDoc)`，这会触发 `IndexWriter` 内部的索引构建逻辑（如更新词典、写入倒排链）。
        *   `mPatchDocCount` 计数器加一，并清空 `mIndexDoc` 以备下一个文档使用。

3.  **关闭 (`Close`)**：
    *   与 `AttributeDataPatcher` 类似，它会检查是否所有文档都已被处理。如果一个文档都未处理，它会循环调用 `EndDocument` 来写入空文档，以确保生成的索引文件包含正确的总文档数。
    *   调用 `mIndexWriter->EndSegment()`，通知 `Writer` 一个 `segment` 的构建结束。
    *   最关键的一步是调用 `mIndexWriter->Dump()`，这将把 `IndexWriter` 在内存中构建的所有索引结构（词典、倒排文件等）持久化到 `Init` 时指定的 `segment` 目录中。

## 4. 技术风险与考量

*   **数据一致性**：`Patcher` 的 `Close` 方法中的完整性检查（`mPatchedDocCount == mTotalDocCount`）是保证数据一致性的关键。如果上层应用逻辑有误，导致没有为所有 `docid` 调用 `Append` 或 `EndDocument`，这个检查可以防止生成一个不完整的、损坏的补丁数据。但这也要求上层应用必须保证其迭代逻辑的完备性。
*   **内存消耗**：`IndexDataPatcher` 在构建倒排索引时，会在内存中缓存词典和倒排链等信息。如果单个 `segment` 非常大，或者词典的基数非常高，可能会消耗大量内存。`IndexDataPatcher` 内部有一个简单的内存池回收机制（`mPool->release()`），但主要的内存压力来自 `IndexWriter` 本身，这部分内存由 `BuildResourceMetrics` 监控。
*   **性能**：数据 `Patch` 的性能主要取决于上层应用的数据源读取速度和 `Patcher` 自身的写入性能。对于属性，写入是主要的瓶颈；对于索引，CPU 密集型的索引构建过程和内存消耗是主要瓶颈。`MergeIOConfig` 中的配置（如 `writeBufferSize`）可以用来调优写入性能。
*   **错误处理**：当前的实现中，很多错误（如数据不一致、文件写入失败）都会直接导致 `INDEXLIB_FATAL_ERROR`，这会使整个进程退出。在分布式计算环境中（如 `MapReduce`），这种快速失败是合理的。但在某些场景下，可能需要更精细的错误恢复机制。

## 5. 总结

`Data Patching` 模块是 `Indexlib` 实现 `Schema` 演化的核心支撑。它通过 `PartitionPatcher`、`AttributeDataPatcher` 和 `IndexDataPatcher` 的三层架构，清晰地分离了 `Schema` 变更管理、属性数据生成和索引数据生成的职责。该模块的设计兼顾了易用性、类型安全和对 `Indexlib` 现有成熟组件（如 `IndexWriter`、`AttributeConvertor`）的复用。通过这套机制，`Indexlib` 能够在不牺牲过多性能和资源的情况下，灵活地适应业务发展带来的 `Schema` 变更需求，为大型索引系统的长期维护和演进提供了强大的技术保障。
