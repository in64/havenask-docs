
# IndexLib 文件系统：存储核心抽象 (Storage Core Abstraction)

**涉及文件:**
* `file_system/Storage.h`
* `file_system/Storage.cpp`

## 1. 概述

`Storage` 类是 IndexLib 文件系统的核心组件之一，它定义了存储子系统的通用接口和基础行为。作为一个抽象基类，`Storage` 封装了对底层存储介质（如内存、磁盘）的访问细节，为上层模块提供了一套统一、标准的文件和目录操作 API。这种设计实现了存储实现与文件系统逻辑的分离，极大地增强了系统的灵活性和可扩展性。无论是内存存储、磁盘存储还是更复杂的打包存储（Package Storage），都遵循 `Storage` 类定义的契约，使得文件系统可以在不同的存储后端之间无缝切换，以适应不同的应用场景和性能要求。

`Storage` 的核心职责包括：

*   **生命周期管理**: 提供 `Init` 方法来初始化存储实例，并依赖析构函数来释放资源。
*   **文件和目录操作**: 定义了创建、读取、删除文件和目录的纯虚函数接口，如 `CreateFileReader`、`CreateFileWriter`、`MakeDirectory`、`RemoveFile` 和 `RemoveDirectory`。
*   **文件缓存**: 内置了一个 `FileNodeCache`，用于缓存文件节点（`FileNode`），以加速对文件的访问。
*   **资源管理**: 通过 `BlockMemoryQuotaController` 控制内存使用，并通过 `StorageMetrics` 监控存储系统的各项指标。
*   **多态实现**: 通过工厂方法 `CreateInputStorage` 和 `CreateOutputStorage`，根据配置动态创建不同类型的存储实例（`MemStorage`、`DiskStorage` 等），支持不同的存储策略。

## 2. 系统架构与关键设计

`Storage` 类的设计体现了典型的面向接口编程和策略模式。它不仅定义了接口，还提供了一些通用的功能实现，如资源文件（`ResourceFile`）和切片文件（`SliceFile`）的管理。

### 2.1. 核心数据结构与类关系

*   **`Storage`**: 抽象基类，定义了所有存储类型的公共接口。
*   **`DiskStorage`**: `Storage` 的一个具体实现，用于处理与物理磁盘的交互。
*   **`MemStorage`**: `Storage` 的另一个具体实现，将文件内容存储在内存中。
*   **`PackageDiskStorage` / `PackageMemStorage`**: 更复杂的 `Storage` 实现，用于将多个逻辑文件打包到单个物理文件中。
*   **`FileNodeCache`**: 一个缓存，用于存储 `FileNode` 对象，以避免重复创建和初始化文件节点的开销。
*   **`EntryTable`**: 记录文件系统中所有文件和目录的元信息，`Storage` 通过它来更新文件大小等状态。
*   **`FileSystemOptions`**: 包含文件系统的配置选项，如存储类型、缓存策略等，`Storage` 的行为受这些选项的控制。
*   **`BlockMemoryQuotaController`**: 用于控制和监控内存使用量。

### 2.2. 工厂模式与动态分发

`Storage` 类提供了两个静态工厂方法，`CreateInputStorage` 和 `CreateOutputStorage`，用于根据 `FileSystemOptions` 中的配置创建具体的存储实例。这种设计将对象的创建逻辑与使用逻辑分离，使得系统可以灵活地配置和切换存储后端。

```cpp
// file_system/Storage.cpp

std::shared_ptr<Storage>
Storage::CreateInputStorage(const shared_ptr<FileSystemOptions>& options,
                            const std::shared_ptr<util::BlockMemoryQuotaController>& memController,
                            const std::shared_ptr<EntryTable>& entryTable,
                            autil::RecursiveThreadMutex* fileSystemLock) noexcept
{
    assert(fileSystemLock);
    std::shared_ptr<Storage> storage(new DiskStorage(true, memController, entryTable));
    storage->_fileSystemLock = fileSystemLock;
    auto ec = storage->Init(options);
    if (ec != FSEC_OK) {
        AUTIL_LOG(ERROR, "init storage failed, ec[%d]", ec);
        return std::shared_ptr<Storage>();
    }
    return storage;
}

std::shared_ptr<Storage>
Storage::CreateOutputStorage(const string& outputRoot, const shared_ptr<FileSystemOptions>& options,
                             const std::shared_ptr<util::BlockMemoryQuotaController>& memController,
                             const string& fsName, const std::shared_ptr<EntryTable>& entryTable,
                             autil::RecursiveThreadMutex* fileSystemLock) noexcept
{
    assert(fileSystemLock);
    std::shared_ptr<Storage> storage;
    if (options->outputStorage == FSST_MEM) {
        storage.reset(new MemStorage(false, memController, entryTable));
    } else if (options->outputStorage == FSST_PACKAGE_MEM) {
        storage.reset(new PackageMemStorage(false, memController, entryTable));
    } else if (options->outputStorage == FSST_PACKAGE_DISK) {
        storage.reset(new PackageDiskStorage(false, outputRoot, fsName, memController, entryTable));
    } else {
        storage.reset(new DiskStorage(false, memController, entryTable));
    }

    storage->_fileSystemLock = fileSystemLock;
    auto ec = storage->Init(options);
    if (ec != FSEC_OK) {
        AUTIL_LOG(ERROR, "init storage failed, ec[%d]", ec);
        return std::shared_ptr<Storage>();
    }
    return storage;
}
```

`CreateInputStorage` 通常用于只读场景，默认创建 `DiskStorage`。而 `CreateOutputStorage` 则根据 `options->outputStorage` 的值来决定创建哪种类型的存储，支持 `FSST_MEM`（内存存储）、`FSST_PACKAGE_MEM`（内存打包存储）、`FSST_PACKAGE_DISK`（磁盘打包存储）以及默认的 `DiskStorage`（磁盘存储）。

### 2.3. 文件节点缓存 (`FileNodeCache`)

`Storage` 内部维护一个 `FileNodeCache` 实例，用于缓存 `FileNode`。`FileNode` 是对一个打开文件的抽象，包含了文件的元信息和数据访问接口。通过缓存 `FileNode`，可以避免对同一文件的重复打开和初始化操作，从而提高性能。

`StoreWhenNonExist` 方法是向缓存中添加 `FileNode` 的核心逻辑。它保证了对于同一个逻辑路径，只有一个 `FileNode` 实例存在于缓存中，避免了数据不一致的问题。

```cpp
// file_system/Storage.cpp

ErrorCode Storage::StoreWhenNonExist(const std::shared_ptr<FileNode>& fileNode) noexcept
{
    const string& filePath = fileNode->GetLogicalPath();
    if (IsExistInCache(filePath)) {
        std::shared_ptr<FileNode> nodeInCache = _fileNodeCache->Find(filePath);
        if (nodeInCache.get() == fileNode.get()) {
            return FSEC_OK;
        }
        AUTIL_LOG(ERROR, "store file node [%s] failed, different file node already exist. old len[%lu] new len[%lu] ",
                  filePath.c_str(), nodeInCache->GetLength(), fileNode->GetLength());
        return FSEC_EXIST;
    }
    _fileNodeCache->Insert(fileNode);
    return FSEC_OK;
}
```

### 2.4. 特殊文件类型的处理

除了标准的读写器创建接口，`Storage` 还提供了对两种特殊文件类型——`ResourceFile` 和 `SliceFile`——的创建和获取逻辑。

*   **`ResourceFile`**: 用于管理非结构化的、可变大小的资源。它通常用于存储一些元数据或者小的配置信息。`CreateResourceFile` 和 `GetResourceFile` 方法封装了 `ResourceFileNode` 的创建和缓存逻辑。
*   **`SliceFile`**: 用于存储由多个定长切片（Slice）组成的文件。这种文件结构适用于需要高效随机访问和追加写入的场景。`CreateSliceFileReader` 和 `CreateSliceFileWriter` 负责处理与 `SliceFileNode` 相关的操作。

这些特殊文件类型的处理逻辑被实现在 `Storage` 基类中，因为它们不依赖于具体的底层存储介质，而是构建在 `FileNode` 抽象之上。

## 3. 技术风险与考量

*   **线程安全**: `Storage` 的许多操作都涉及到对共享数据结构（如 `_fileNodeCache` 和 `_entryTable`）的访问。因此，必须确保所有实现都是线程安全的。`Storage` 类本身持有一个 `autil::RecursiveThreadMutex* _fileSystemLock`，子类在必要时需要使用这个锁来保护临界区。
*   **缓存一致性**: `FileNodeCache` 的存在引入了缓存一致性的问题。当底层文件被外部进程修改时，缓存中的 `FileNode` 可能会变得陈旧。虽然在 IndexLib 的使用场景中，文件通常由系统自身管理，但这个问题仍然值得关注。
*   **资源泄漏**: `Storage` 及其子类管理着文件句柄、内存缓冲区等多种资源。必须确保在所有代码路径上都能正确释放这些资源，避免泄漏。`std::shared_ptr` 的广泛使用有助于此，但仍需谨慎处理原始指针和手动资源分配。
*   **错误处理**: 文件系统操作可能会因为多种原因失败（如磁盘空间不足、权限问题、网络中断等）。`Storage` 的接口使用 `ErrorCode` 和 `FSResult` 来传递错误信息，调用者必须正确处理这些错误，以保证系统的健壮性。

## 4. 总结

`Storage` 类作为 IndexLib 文件系统的存储抽象层，其设计清晰、功能强大。通过将存储的接口与实现分离，它为系统提供了高度的灵活性和可扩展性。工厂模式的应用使得存储后端的选择变得简单和透明。内置的 `FileNodeCache` 和对特殊文件类型的支持，进一步提升了文件系统的性能和功能丰富性。理解 `Storage` 的设计和工作原理，是深入掌握 IndexLib 文件系统内部机制的关键一步。
