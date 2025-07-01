
# IndexLib 文件系统：磁盘存储 (Disk-based Storage)

**涉及文件:**
* `file_system/DiskStorage.h`
* `file_system/DiskStorage.cpp`

## 1. 概述

`DiskStorage` 是 `Storage` 抽象基类的另一个关键实现，它负责管理存储在持久化介质（如本地磁盘、Pangu 等分布式文件系统）上的文件和目录。作为 IndexLib 中最常用、最基础的存储引擎，`DiskStorage` 为数据的长期、可靠存储提供了保障。它直接与底层文件系统接口（通过 `fslib` 库）交互，处理文件的物理读写、创建和删除。

`DiskStorage` 的核心功能和设计目标包括：

*   **持久化存储**: 确保数据被可靠地写入物理存储设备。
*   **灵活的文件加载策略**: 支持多种文件加载和访问模式，如 `mmap`、`cache`（块缓存）、`mem`（完全加载到内存）等。这些策略通过 `LoadConfig` 进行配置，允许用户根据不同的文件类型和访问模式优化性能。
*   **文件节点创建器 (`FileNodeCreator`)**: 使用一个可扩展的创建器系统，根据 `LoadConfig` 和文件类型，动态选择最合适的 `FileNode` 实现（如 `MmapFileNode`、`BlockFileNode`、`MemFileNode`）。
*   **读写分离**: `DiskStorage` 可以被配置为只读模式，这在索引加载和查询服务中非常有用。
*   **生命周期管理**: 支持基于文件路径的生命周期（Lifecycle）配置，可以为不同目录下的文件指定不同的加载策略。

## 2. 系统架构与关键设计

`DiskStorage` 的设计核心在于如何将逻辑文件操作（如 `CreateFileReader`）映射到底层物理文件的访问，并在此过程中应用各种性能优化策略。

### 2.1. 文件加载策略与 `FileNodeCreator`

`DiskStorage` 最具特色的设计之一是其灵活的文件加载机制。这套机制由 `LoadConfigList` 和 `FileNodeCreator` 共同实现。

*   **`LoadConfigList`**: 在 `FileSystemOptions` 中定义，包含了一系列的 `LoadConfig`。每个 `LoadConfig` 定义了一个文件匹配模式（正则表达式）、一个加载策略（如 `mmap`, `cache`）以及相关的参数（如 `lock`, `cache_size`）。
*   **`FileNodeCreator`**: 这是一个抽象基类，负责创建特定类型的 `FileNode`。`DiskStorage` 在初始化时，会为每种加载策略（以及一些默认策略）创建一个对应的 `FileNodeCreator` 实例。
    *   `MmapFileNodeCreator`: 创建 `MmapFileNode`，用于内存映射文件。
    *   `BlockFileNodeCreator`: 创建 `BlockFileNode`，使用文件块缓存。
    *   `MemFileNodeCreator`: 创建 `MemFileNode`，将文件完整加载到内存。
    *   `BufferedFileNodeCreator`: 创建 `BufferedFileNode`，用于带缓冲的读写。

当需要打开一个文件时，`DiskStorage::GetFileNodeCreator` 方法会根据文件的逻辑路径和生命周期设置，在 `_fileNodeCreators` 列表中查找第一个匹配的 `LoadConfig`，并返回其对应的 `FileNodeCreator`。

```cpp
// file_system/DiskStorage.cpp

FSResult<std::shared_ptr<FileNodeCreator>>
DiskStorage::GetFileNodeCreator(const string& logicalFilePath, const string& physicalFilePath, FSOpenType type,
                                const std::string lifeCycleInput) const noexcept
{
    size_t matchedIdx = 0;
    string lifecycle = lifeCycleInput.empty() ? _lifecycleTable.GetLifecycle(logicalFilePath) : lifeCycleInput;
    for (; matchedIdx < _fileNodeCreators.size(); ++matchedIdx) {
        if (_fileNodeCreators[matchedIdx]->Match(logicalFilePath, lifecycle)) {
            // ...
            break;
        }
    }

    if (matchedIdx < _fileNodeCreators.size()) {
        // ...
        return {FSEC_OK, matchCreator};
    }
    return DeduceFileNodeCreator(_defaultCreatorMap.at(FSOT_LOAD_CONFIG), type, physicalFilePath);
}
```

这种设计使得文件加载行为高度可配置，用户可以为不同类型的文件（如索引文件、属性文件、摘要文件）指定最优的加载策略，从而在内存使用和访问性能之间找到最佳平衡。

### 2.2. 文件读取流程

`DiskStorage::CreateFileReader` 的执行流程如下：

1.  **确定打开类型**: 根据 `ReaderOption` 和可能的生命周期配置，确定最终的文件打开类型 (`FSOpenType`)。
2.  **查找缓存**: 在 `_fileNodeCache` 中查找是否已存在该文件的 `FileNode`。如果命中且类型匹配，则直接使用缓存的 `FileNode`，这可以避免重复的文件打开和 `mmap` 操作。
3.  **创建文件节点**: 如果缓存未命中或类型不匹配，则调用 `CreateFileNode`。此方法会：
    a.  调用 `GetFileNodeCreator` 获取合适的 `FileNodeCreator`。
    b.  使用该 `creator` 创建一个新的 `FileNode` 实例。
    c.  调用 `FileNode` 的 `Open` 方法，这会实际执行底层的文件打开或 `mmap` 操作。
    d.  如果配置了缓存 (`options->useCache`)，则将新创建的 `FileNode` 插入到 `_fileNodeCache` 中。
4.  **创建文件读取器**: 基于获取到的 `FileNode`，创建一个 `FileReader`（通常是 `NormalFileReader` 或 `BufferedFileReader`）并返回。

### 2.3. 文件写入流程

`DiskStorage` 的写入操作相对直接。`CreateFileWriter` 通常会创建一个 `BufferedFileWriter`。`BufferedFileWriter` 内部维护一个缓冲区，当缓冲区满或被显式 `Flush` 时，才会将数据写入底层文件系统。这种缓冲机制可以减少 `write` 系统调用的次数，提高写入性能。

```cpp
// file_system/DiskStorage.cpp

FSResult<std::shared_ptr<FileWriter>> DiskStorage::CreateFileWriter(const string& logicalFilePath,
                                                                    const string& physicalFilePath,
                                                                    const WriterOption& writerOption) noexcept
{
    // ... (省略只读和断言检查)
    auto updateFileSizeFunc = [this, logicalFilePath](int64_t size) {
        _entryTable && _entryTable->UpdateFileSize(logicalFilePath, size);
    };
    std::shared_ptr<FileWriter> fileWriter(new BufferedFileWriter(writerOption, std::move(updateFileSizeFunc)));
    auto ec = fileWriter->Open(logicalFilePath, physicalFilePath);
    RETURN2_IF_FS_ERROR(ec, std::shared_ptr<FileWriter>(), "FileWriter Open [%s] ==> [%s] failed",
                        logicalFilePath.c_str(), physicalFilePath.c_str());
    return {FSEC_OK, fileWriter};
}
```

### 2.4. 对打包文件（Package File）的支持

`DiskStorage` 还包含了对打包文件的特殊处理逻辑。当打开一个位于打包文件内部的文件时，它需要共享对外部打包文件的文件句柄或内存映射。`ModifyPackageOpenMeta` 方法负责处理这个逻辑，它会缓存并复用指向同一个打包文件的 `FileCarrier`，从而避免对同一个打包文件的重复打开。

## 3. 技术风险与考量

*   **性能**: `DiskStorage` 的性能直接受限于底层存储介质的 I/O 能力。不合理的 `LoadConfig` 配置（例如，对大文件使用 `mem` 模式，或对频繁随机访问的文件不使用 `cache` 模式）可能会导致性能瓶颈或过高的内存消耗。
*   **Mmap 的限制**: `mmap` 是一个强大的工具，但也有其限制。例如，在 32 位系统上，可映射的地址空间有限。此外，`mmap` 不适用于某些类型的网络文件系统。`DiskStorage::SupportMmap` 方法会检查当前环境是否支持 `mmap`，并在不支持时回退到其他加载模式（如 `mem`）。
*   **文件句柄管理**: `DiskStorage` 会打开大量文件，因此必须有效管理文件句柄。`FileNodeCache` 在这里起到了关键作用，它复用 `FileNode`，从而间接复用了文件句柄。然而，如果缓存策略不当，仍可能导致文件句柄耗尽。
*   **并发和一致性**: 与 `fslib` 的交互、对 `_fileNodeCache` 和 `_packageFileCarrierMap` 的修改都需要在多线程环境下保证安全。`DiskStorage` 依赖于上层 `FileSystem` 传递的 `_fileSystemLock` 来进行同步。

## 4. 总结

`DiskStorage` 是 IndexLib 文件系统的基石，它提供了一个健壮、可配置的接口来访问持久化存储。其最强大的特性在于通过 `LoadConfig` 实现的高度灵活的文件加载策略，这使得 IndexLib 能够针对不同的硬件环境和应用场景进行深度性能优化。虽然它的性能受限于物理 I/O，但通过 `mmap`、块缓存和写缓冲等技术，`DiskStorage` 在很大程度上缓解了 I/O 瓶颈，为构建高性能的搜索引擎和数据分析系统提供了坚实的基础。
