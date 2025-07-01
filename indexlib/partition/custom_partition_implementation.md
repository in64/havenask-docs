
# Indexlib 自定义分区实现代码分析

**涉及文件:**
* `indexlib/partition/custom_offline_partition.cpp`
* `indexlib/partition/custom_offline_partition.h`
* `indexlib/partition/custom_offline_partition_writer.cpp`
* `indexlib/partition/custom_offline_partition_writer.h`
* `indexlib/partition/custom_online_partition_writer.cpp`
* `indexlib/partition/custom_online_partition_writer.h`

## 1. 系统概述

Indexlib 的自定义分区（Custom Partition）是其强大扩展性的集中体现。在标准的倒排、KV、KKV 索引模型无法满足特定业务需求时，用户可以通过插件化的方式，定义一套全新的索引组织、构建和查询逻辑。这套自定义分区实现的核心思想是**将分区管理的通用框架与具体的业务数据处理逻辑解耦**，Indexlib 提供框架，而用户通过实现 `Table` 系列接口来注入业务逻辑。

该模块主要由以下几个关键组件构成，它们与标准分区的组件一一对应，但其内部逻辑截然不同：

*   **`CustomOfflinePartition`**: 作为 `IndexPartition` 的自定义实现，它负责在离线环境中加载用户自定义的 `Table` 插件。它不关心索引的具体格式，而是扮演一个“容器”和“协调者”的角色，为用户插件的运行提供必要的环境，如文件系统、内存控制和版本管理。所有与数据构建相关的核心任务，都将委托给插件来完成。

*   **`CustomOfflinePartitionWriter`**: 作为 `PartitionWriter` 的自定义实现，它是连接 Indexlib 框架和用户自定义 `TableWriter` 插件的桥梁。它接收上层传来的文档流，但并不自己处理，而是直接调用 `TableWriter` 插件的 `Build` 方法。它负责管理 `TableWriter` 的生命周期、内存使用，并触发其 Dump 操作。

*   **`CustomOnlinePartitionWriter`**: 这是自定义分区的在线写入器，它继承自 `CustomOfflinePartitionWriter`。与标准在线写入器类似，它在离线写入器的基础上增加了一层适配逻辑，主要用于处理在线服务环境下的特定需求，例如根据时间戳过滤过时数据。其核心构建任务依然由 `TableWriter` 插件完成。

整个自定义分区体系的核心是**委托（Delegation）**。Indexlib 框架的 `CustomPartition` 和 `CustomPartitionWriter` 将数据处理的“指挥权”完全交给了用户实现的 `Table` 插件。这种设计提供了无与伦比的灵活性，使得 Indexlib 可以被用于构建各种非传统的、高度定制化的索引结构，例如图索引、向量索引等，极大地拓宽了其应用边界。

## 2. `CustomOfflinePartition`：自定义插件的离线运行容器

`CustomOfflinePartition` 是自定义 `Table` 插件在离线构建环境下的宿主。它的主要职责不是处理数据，而是为插件的运行搭建舞台。

### 2.1 功能目标

*   **插件加载与管理**: 根据 Schema 中定义的 `TableType`，通过 `TablePluginLoader` 加载对应的用户插件（动态链接库 .so 文件），并管理其生命周期。
*   **环境提供**: 为插件提供运行所需的上下文环境，包括：
    *   `IFileSystem`: 供插件读写索引文件的虚拟文件系统。
    *   `PartitionMemoryQuotaController`: 供插件使用的内存配额控制器。
    *   `CounterMap` 和 `MetricProvider`: 供插件上报自定义的统计指标。
*   **构建流程协调**: 创建并管理 `CustomOfflinePartitionWriter`，并通过它驱动整个离线构建流程。
*   **版本管理**: 负责索引版本（`Version`）的提交和管理，这部分逻辑对插件是透明的。
*   **异步 Dump 支持**: 提供了异步 Dump 机制，允许 `TableWriter` 将耗时的 Dump 操作放入后台线程执行，以提高构建效率。

### 2.2 核心逻辑

#### 2.2.1 打开与插件初始化 (`Open`)

`Open` 方法是 `CustomOfflinePartition` 的生命周期起点，其核心任务是加载插件并初始化分区数据。

```cpp
// indexlib/partition/custom_offline_partition.cpp

IndexPartition::OpenStatus CustomOfflinePartition::DoOpen(const string& root, const IndexPartitionSchemaPtr& schema,
                                                          const IndexPartitionOptions& options, versionid_t versionId)
{
    // ...
    // 加载磁盘上的基线版本和 Schema
    Version version;
    VersionLoader::GetVersionS(root, version, versionId);
    // ...
    mSchema = SchemaAdapter::LoadSchema(mRootDirectory, version.GetSchemaVersionId());
    ValidateConfigAndParam(schema, mSchema, mOptions);

    // 1. 初始化内存控制器
    mTableWriterMemController.reset(
        new BlockMemoryQuotaController(mPartitionMemController, mSchema->GetSchemaName() + "_TableWriterMemory"));

    // 2. 加载 Table 插件
    if (!InitPlugin(mIndexPluginPath, mSchema, mOptions)) {
        return OS_FAIL;
    }
    // ...

    // 3. 初始化自定义分区数据 (CustomPartitionData)
    mDumpSegmentQueue.reset(new DumpSegmentQueue());
    if (!InitPartitionData(mFileSystem, version, mCounterMap, mPluginManager)) {
        return OS_FAIL;
    }

    // 4. 创建 TableFactoryWrapper，这是创建 Table 核心组件的工厂
    if (!mTableFactory) {
        mTableFactory.reset(new TableFactoryWrapper(mSchema, mOptions, partitionData->GetPluginManager()));
        if (!mTableFactory->Init()) {
            return OS_FAIL;
        }
    }
    
    // 5. 初始化写入器
    InitWriter(partitionData);
    mClosed = false;
    
    // 6. 准备后台任务
    if (!PrepareIntervalTask()) {
        return OS_FAIL;
    }

    return OS_OK;
}

bool CustomOfflinePartition::InitPlugin(const string& pluginPath, const IndexPartitionSchemaPtr& schema,
                                        const IndexPartitionOptions& options)
{
    // 通过 TablePluginLoader 加载用户实现的 .so 文件
    mPluginManager = TablePluginLoader::Load(pluginPath, schema, options);
    if (!mPluginManager) {
        return false;
    }
    return true;
}
```
`Open` 流程的关键点在于，它并没有像 `OfflinePartition` 那样去初始化具体的 `SegmentWriter` 或 `Modifier`，而是：
1.  **加载插件**: `InitPlugin` 通过 `TablePluginLoader` 加载用户代码，并返回一个 `PluginManager`。这个 `PluginManager` 是后续所有与插件交互的入口。
2.  **创建工厂**: `TableFactoryWrapper` 是一个关键的辅助类，它封装了从 `PluginManager` 中获取 `TableFactory` 的逻辑。用户插件必须实现 `TableFactory` 接口，该接口负责创建 `TableWriter`、`TableReader` 和 `TableResource`。
3.  **初始化写入器**: `InitWriter` 方法会创建一个 `CustomOfflinePartitionWriter`，并将 `TableFactoryWrapper` 传递给它，由写入器来完成后续的 `TableWriter` 创建。

#### 2.2.2 异步 Dump 机制

为了提高构建性能，`CustomOfflinePartition` 支持异步 Dump。当 `TableWriter` 完成一个内存段的构建并决定 Dump 时，它不是立即执行耗时的磁盘写入，而是将这个任务封装成一个 `CustomSegmentDumpItem` 并放入 `DumpSegmentQueue` 中。

`CustomOfflinePartition` 会启动一个后台线程 `AsyncDumpThread`，专门负责处理这个队列。

```cpp
// indexlib/partition/custom_offline_partition.cpp

void CustomOfflinePartition::AsyncDumpThread()
{
    // ...
    while (!mClosed) {
        FlushDumpSegmentQueue();
        usleep(sleepTime);
    }
    // ...
}

void CustomOfflinePartition::FlushDumpSegmentQueue()
{
    // ...
    while (mDumpSegmentQueue && (mDumpSegmentQueue->Size() > 0)) {
        // 从队列中取出一个 Dump 任务
        bool hasDumpedSegment = false;
        bool dumpSuccess = mDumpSegmentQueue->ProcessOneItem(hasDumpedSegment);

        if (!dumpSuccess) {
            // ... 错误处理
        }
        if (hasDumpedSegment) {
            // Dump 成功后，提交版本
            mPartitionDataHolder.Get()->CommitVersion();
            mDumpSegmentQueue->PopDumpedItems();
        } else {
            break;
        }
    }
    // ...
}
```
这种生产者-消费者模式将构建线程（生产者）和 Dump 线程（消费者）解耦，使得构建流程不会因为磁盘 I/O 而被阻塞，从而实现了构建和 Dump 的流水线作业。

### 2.3 设计动机

`CustomOfflinePartition` 的设计动机是**提供一个与具体业务无关的、通用的、可插拔的离线构建框架**。它将所有与特定索引结构相关的逻辑都抽象出去，由用户通过 `Table` 插件来实现。Indexlib 框架只负责通用的部分，如：
*   版本管理和生命周期控制。
*   文件系统和内存的抽象与管理。
*   构建流程的驱动和协调（如触发 Dump、提交版本）。

这种设计极大地增强了 Indexlib 的灵活性和适应性，使其不仅仅是一个倒排索引库，而是一个可扩展的索引平台。

### 2.4 可能的技术风险

*   **插件质量**: 整个系统的稳定性和性能高度依赖于用户实现的 `Table` 插件的质量。如果插件存在内存泄漏、死锁或性能瓶颈，将会直接影响整个构建系统。
*   **接口耦合**: 框架与插件之间的接口（如 `TableWriter`, `TableResource`）需要精心设计。如果接口定义过于复杂或模糊，会增加插件的开发难度；如果过于简单，又可能无法满足复杂的业务需求。
*   **资源隔离**: 框架需要为插件提供有效的资源隔离机制。例如，插件的内存使用必须被严格限制在 `TableWriterMemoryQuotaController` 的配额内，否则失控的插件可能会耗尽整个进程的内存。

## 3. `CustomOfflinePartitionWriter`：插件 `TableWriter` 的驱动者

`CustomOfflinePartitionWriter` 是自定义分区的核心写入器，它负责驱动用户实现的 `TableWriter` 插件来完成实际的文档构建工作。

### 3.1 功能目标

*   **`TableWriter` 管理**: 负责创建、初始化和管理 `TableWriter` 插件的生命周期。
*   **`TableResource` 管理**: 负责创建和管理 `TableResource`。`TableResource` 是一个用户自定义的资源对象，它可以被多个 `TableWriter` 共享（例如，共享的词典、模型等），从而节省内存和初始化开销。
*   **文档委托**: 将上层传入的 `Document` 对象直接委托给 `TableWriter` 的 `Build` 方法进行处理。
*   **Dump 触发与执行**: 监控 `TableWriter` 的状态（如内存占用、是否已满），并在满足条件时，将其封装成 `CustomSegmentDumpItem` 提交给 `DumpSegmentQueue`（在异步模式下）或直接执行 Dump。

### 3.2 核心逻辑

#### 3.2.1 打开与 `TableWriter` 初始化 (`Open` & `ReOpenNewSegment`)

`Open` 方法会初始化一些基本资源，而 `ReOpenNewSegment` 则负责在每次需要构建一个新段时，创建新的 `TableWriter` 实例。

```cpp
// indexlib/partition/custom_offline_partition_writer.cpp

void CustomOfflinePartitionWriter::ReOpenNewSegment(const PartitionModifierPtr& modifier)
{
    // ...
    // 1. 准备 TableResource
    if (!PrepareTableResource(mPartitionData, mTableResource)) {
        INDEXLIB_FATAL_ERROR(BadParameter, ...);
    }

    // ...
    // 2. 准备新段的目录
    DirectoryPtr newSegDir = PrepareNewSegmentDirectory(...);
    DirectoryPtr customDataDir = PrepareBuildingDirForTableWriter(newSegDir);
    
    // 3. 初始化 TableWriter
    mTableWriter = InitTableWriter(newSegData, customDataDir, mTableResource, mPartitionData->GetPluginManager());
    if (!mTableWriter) {
        INDEXLIB_FATAL_ERROR(Runtime, "InitTableWriter ... failed");
    }
    // ...
}

table::TableWriterPtr CustomOfflinePartitionWriter::InitTableWriter(
    const SegmentDataBasePtr& newSegmentData,
    const DirectoryPtr& dataDirectory,
    const TableResourcePtr& tableResource,
    const PluginManagerPtr& pluginManager)
{
    // 1. 从工厂创建 TableWriter 实例
    TableWriterPtr tableWriter = mTableFactory->CreateTableWriter();

    // 2. 准备初始化参数
    TableWriterInitParamPtr initParam(new TableWriterInitParam);
    initParam->schema = mSchema;
    initParam->options = mOptions;
    initParam->tableResource = tableResource;
    // ...
    
    // 3. 为 TableWriter 分配内存控制器
    util::UnsafeSimpleMemoryQuotaControllerPtr segMemControl(
        new util::UnsafeSimpleMemoryQuotaController(mTableWriterMemController));
    // ...
    initParam->memoryController = segMemControl;
    
    // 4. 调用插件的 Init 方法
    if (!tableWriter->Init(initParam)) {
        return TableWriterPtr();
    }
    return tableWriter;
}
```
这个过程的核心是**资源准备和委托初始化**：
1.  **`PrepareTableResource`**: 确保 `TableResource` 已经准备好。`TableResource` 是一个可以跨段共享的资源，它只在第一次创建 `TableWriter` 时初始化，后续会复用。
2.  **`InitTableWriter`**: 这是关键步骤。它首先通过 `mTableFactory`（来自用户插件）创建 `TableWriter` 的实例，然后为其准备一个 `TableWriterInitParam` 对象，该对象包含了 Schema、配置、`TableResource` 以及一个独立的内存控制器。最后，调用 `tableWriter->Init()`，将控制权交给插件代码，让其完成自己的初始化。

#### 3.2.2 文档构建 (`BuildDocument`)

`BuildDocument` 的实现非常简单，它直接将任务委托给了 `mTableWriter`。

```cpp
// indexlib/partition/custom_offline_partition_writer.cpp

bool CustomOfflinePartitionWriter::BuildDocument(const document::DocumentPtr& doc)
{
    ScopedLock lock(mWriterLock);
    assert(mTableWriter);
    // ... 更新元信息
    UpdateBuildingSegmentMeta(doc);
    
    // 直接调用插件的 Build 方法
    typedef TableWriter::BuildResult BuildResult;
    BuildResult result = mTableWriter->Build(docCount, doc);
    
    // ... 根据插件返回的结果进行处理
    mTableWriter->UpdateMemoryUse(); // 通知插件更新其内存使用统计
    // ...
    return true;
}
```

#### 3.2.3 Dump 决策与执行 (`NeedDump` & `DumpSegment`)

`NeedDump` 方法会综合多种条件来判断是否需要触发一次 Dump：
*   文档数量是否达到 `maxDocCount`。
*   调用 `mTableWriter->IsFull()`，询问插件其内部是否已满。
*   内存使用是否超过配额。

当 `NeedDump` 返回 `true` 时，`DumpSegment` 方法会被调用。

```cpp
// indexlib/partition/custom_offline_partition_writer.cpp

void CustomOfflinePartitionWriter::DumpSegment()
{
    // ...
    if (!IsDirty()) { // 如果插件报告没有写入任何数据，则直接清理并返回
        // ...
        return;
    }
    
    // 1. 创建一个 Dump 任务项
    CustomSegmentDumpItemPtr segmentDumpItem(new CustomSegmentDumpItem(mOptions, mSchema, mPartitionName));

    // 2. 初始化任务项，将 mTableWriter 等核心组件传递给它
    segmentDumpItem->Init(..., mTableWriter);

    if (mEnableAsyncDump) {
        // 3a. 异步模式：将任务项推入队列
        mDumpSegmentQueue->PushDumpItem(segmentDumpItem);
    } else {
        // 3b. 同步模式：直接执行 Dump
        segmentDumpItem->Dump();
        CommitVersion(...);
    }
    // ...
    CleanResourceAfterDump();
}
```
`CustomSegmentDumpItem` 的 `Dump` 方法最终会调用 `mTableWriter->Dump()`，将内存中的自定义索引数据写入磁盘。

### 3.3 设计动机

`CustomOfflinePartitionWriter` 的设计完全遵循了**控制反转（Inversion of Control, IoC）** 的原则。它作为一个框架，定义了构建的流程（何时创建 Writer，何时 Build，何时 Dump），但将流程中每一步的具体实现都交给了由外部（插件）注入的 `TableWriter` 来完成。这种设计模式使得框架本身保持了极高的通用性和稳定性，而将业务的易变性和复杂性隔离在了插件内部，是构建可扩展系统的经典范例。

## 4. `CustomOnlinePartitionWriter`：自定义插件的在线适配器

`CustomOnlinePartitionWriter` 的角色与 `OnlinePartitionWriter` 非常相似，它是一个轻量级的适配器，用于将自定义分区对接到在线服务环境中。

### 4.1 功能目标

*   继承 `CustomOfflinePartitionWriter` 的所有功能。
*   增加在线环境特有的逻辑，主要是根据时间戳过滤过时数据。

### 4.2 核心逻辑

它的核心逻辑与 `OnlinePartitionWriter` 如出一辙，在 `Open` 时计算过滤时间戳 `mRtFilterTimestamp`，并在 `BuildDocument` 时进行检查。

```cpp
// indexlib/partition/custom_online_partition_writer.cpp

void CustomOnlinePartitionWriter::Open(const config::IndexPartitionSchemaPtr& schema,
                                       const PartitionDataPtr& partitionData, const PartitionModifierPtr& modifier)
{
    ScopedLock lock(mOnlineWriterLock);
    // 1. 计算过滤时间戳
    Version incVersion = partitionData->GetOnDiskVersion();
    OnlineJoinPolicy joinPolicy(incVersion, schema->GetTableType(), document::SrcSignature());
    mRtFilterTimestamp = joinPolicy.GetRtFilterTimestamp();
    
    // 2. 调用父类的 Open 方法，完成 TableWriter 的初始化
    CustomOfflinePartitionWriter::Open(schema, partitionData, PartitionModifierPtr());
    mIsClosed = false;
}

bool CustomOnlinePartitionWriter::BuildDocument(const DocumentPtr& doc)
{
    // ...
    // 在线特有的逻辑，例如时间戳检查 (当前代码中被注释，但设计意图如此)
    // if (!CheckTimestamp(doc)) {
    //     return false;
    // }
    
    // 调用父类的 BuildDocument，最终委托给 TableWriter 插件
    return CustomOfflinePartitionWriter::BuildDocument(doc);
}
```

### 4.3 设计动机

通过继承 `CustomOfflinePartitionWriter`，`CustomOnlinePartitionWriter` 实现了最大程度的代码复用。它只添加了在线环境所必需的最少逻辑，保持了自身的轻量级。这种清晰的继承关系和职责划分，使得整个自定义分区的代码体系易于理解和维护。
