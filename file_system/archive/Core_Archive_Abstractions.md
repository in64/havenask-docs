
# Indexlib 归档存储核心抽象：`ArchiveDirectory`, `ArchiveFolder`, 与 `ArchiveFile` 代码分析

**涉及文件:**
* `file_system/archive/ArchiveDirectory.h`
* `file_system/archive/ArchiveDirectory.cpp`
* `file_system/archive/ArchiveFolder.h`
* `file_system/archive/ArchiveFolder.cpp`
* `file_system/archive/ArchiveFile.h`
* `file_system/archive/ArchiveFile.cpp`

## 1. 功能目标

该模块旨在提供一个**归档文件系统**，它将大量小文件聚合成少数几个大的日志文件（`LogFile`），以解决文件系统（特别是分布式文件系统）中存在大量小文件时带来的元数据管理开销大、I/O 性能低下的问题。其核心目标是：

*   **虚拟化目录结构**：对外提供标准的目录和文件操作接口（`IDirectory`），但物理上将文件数据和元数据存储在少数几个归档文件中。
*   **高效的小文件存储**：将多个逻辑文件（`ArchiveFile`）的数据顺序写入一个大的物理日志文件（`LogFile`），减少物理文件的数量。
*   **元数据集中管理**：每个逻辑文件的元数据（如在日志文件中的偏移、长度等）被集中记录，从而可以快速定位文件。
*   **兼容性与扩展性**：设计上与 Indexlib 的文件系统接口（`IDirectory`, `FileReader`, `FileWriter`）兼容，可以无缝地替换标准的文件系统实现。

这个系统特别适用于索引构建等场景，因为这些场景会产生大量生命周期相似的临时小文件。通过归档存储，可以显著提升文件操作效率和系统整体性能。

## 2. 系统架构与设计动机

该归档系统的设计遵循了**分层抽象**和**关注点分离**的原则，主要由三个核心类构成，它们共同协作，构建了一个完整的虚拟文件系统。

### 2.1. 核心组件及其关系

1.  **`ArchiveFolder` (归档文件夹)**：
    *   **角色**：归档系统的**顶层管理者和入口**。它负责管理一个或多个物理日志文件（`LogFile`），并维护着整个归档目录的元数据。
    *   **设计动机**：需要一个中心化的组件来协调所有归档操作。`ArchiveFolder` 封装了底层的 `LogFile` 管理逻辑，包括日志文件的创建、打开、读取和版本控制。它对上层屏蔽了物理存储的复杂性。

2.  **`ArchiveDirectory` (归档目录)**：
    *   **角色**：实现了 `IDirectory` 接口，是归档系统对外暴露的**虚拟目录**。用户通过它来进行文件和子目录的创建、查找、删除等操作。
    *   **设计动机**：为了与 Indexlib 的文件系统框架无缝集成，必须提供一个标准的目录接口。`ArchiveDirectory` 将标准的目录操作（如 `CreateFileWriter`, `ListDir`）转换为对 `ArchiveFolder` 的内部调用，充当了**适配器（Adapter）**的角色。它维护了一个相对路径（`_relativePath`），使其能够模拟出层级目录结构。

3.  **`ArchiveFile` (归档文件)**：
    *   **角色**：代表一个**逻辑文件**，其数据被存储在 `LogFile` 中。它负责将大文件拆分成固定大小的块（Block），并记录每个块在 `LogFile` 中的位置。
    *   **设计动机**：为了高效地读写大文件并避免一次性加载整个文件内容到内存，`ArchiveFile` 采用了**分块存储**的策略。这种设计允许对大文件进行随机读取（PRead），并简化了缓冲区的管理。

它们之间的关系如下图所示：

```
+-------------------------+
|      Application        |
+-------------------------+
           |
           v
+-------------------------+       +-------------------------+
|   ArchiveDirectory      |------>|     ArchiveFolder       |
| (IDirectory Interface)  |       | (Manages LogFiles)      |
+-------------------------+       +-------------------------+
           |                                |
           v                                v
+-------------------------+       +-------------------------+
|      ArchiveFile        |------>|        LogFile          |
|   (Logical File)        |       |   (Physical Storage)    |
+-------------------------+       +-------------------------+
```

*   **`ArchiveDirectory`** 将用户的目录操作请求（例如创建文件）转发给 **`ArchiveFolder`**。
*   **`ArchiveFolder`** 接收到请求后，会创建一个 **`ArchiveFile`** 实例来代表这个新的逻辑文件，并从其管理的 **`LogFile`** 池中为这个 `ArchiveFile` 分配写入空间。
*   **`ArchiveFile`** 在写入数据时，会将数据分块并调用 **`LogFile`** 的接口，将数据块和对应的元数据写入物理日志文件中。

### 2.2. 关键设计决策

*   **日志追加模式（Append-Only）**：`ArchiveFolder` 底层依赖的 `LogFile` 以追加模式工作。这种设计简化了并发控制，提高了写入吞吐量，因为写入操作总是顺序的。但缺点是不支持对现有文件的修改和删除，只能通过创建新版本或标记删除来模拟。
*   **元数据与数据分离**：`ArchiveFile` 的元数据（`FileMeta`，包含块信息）与其实际数据是分开存储的。元数据通常在文件关闭时一次性写入，而数据则在写入过程中分块刷盘。这种分离使得元数据可以被快速加载和查询，而无需读取庞大的数据体。
*   **版本管理**：`ArchiveFolder` 通过在 `LogFile` 文件名中加入版本号（`_logFileVersion`）和签名（`signature`）来实现简单的版本控制。这使得系统可以区分不同批次写入的文件，为数据恢复和一致性检查提供了基础。
*   **内存缓存**：`ArchiveFolder` 使用 `_inMemFiles` 集合来缓存当前正在写入的文件列表，以加速对这些文件的存在性检查（`IsExist`），避免了不必要的磁盘I/O。

## 3. 核心逻辑与算法

### 3.1. `ArchiveFolder`：归档的“大管家”

`ArchiveFolder` 的核心职责是管理 `LogFile` 的生命周期和提供对归档文件的统一访问入口。

#### 初始化 (`Open`)

`Open` 方法是 `ArchiveFolder` 的入口点。它会扫描指定目录下所有符合命名规范（`INDEXLIB_ARCHIVE_FILES_*`）的 `LogFile`，并进行初始化。

*   **扫描与加载**：它会列出目录下的所有文件，并通过 `GetLogFiles` 筛选出所有的历史 `LogFile`。
*   **版本排序**：加载的 `LogFile` 会根据版本号进行降序排序（`ReadLogCompare`），这样在查找文件时可以优先从最新的归档中查找，提高了效率。
*   **写实例准备**：它会确定下一个可用的写版本号（`_logFileVersion`），但并不会立即创建 `LogFile` 的写实例（`_writeLogFile`），而是采用**懒加载**模式，在第一次有写请求时才真正创建。

**核心代码 (`ArchiveFolder::InitLogFiles`)**:
```cpp
FSResult<void> ArchiveFolder::InitLogFiles() noexcept
{
    fslib::FileList fileList;
    RETURN_IF_FS_ERROR(_directory->ListDir("", ListOption(), fileList), "ListDir failed");
    fslib::FileList logFiles;
    vector<int64_t> logFileVersions;
    string emptySuffix;
    RETURN_IF_FS_ERROR(GetLogFiles(fileList, emptySuffix, logFiles, logFileVersions), "GetLogFiles failed");
    for (unsigned int i = 0; i < logFiles.size(); i++) {
        ReadLogFile readLogFile;
        readLogFile.logFile.reset(new LogFile());
        RETURN_IF_FS_ERROR(readLogFile.logFile->Open(_directory, logFiles[i], fslib::READ), "Open in [%s] failed",
                           _directory->DebugString().c_str());
        readLogFile.version = logFileVersions[i];
        _readLogFiles.push_back(readLogFile);
    }
    sort(_readLogFiles.begin(), _readLogFiles.end(), ReadLogCompare());
    auto [ec, version] = GetMaxWriteVersion(fileList);
    RETURN_IF_FS_ERROR(ec, "GetMaxWriteVersion failed");
    _logFileVersion = version + 1;
    return FSEC_OK;
}
```

#### 文件创建 (`CreateFileWriter`)

当 `ArchiveDirectory` 请求创建一个文件时，`ArchiveFolder` 的 `CreateFileWriter` 会被调用。

1.  **加锁**：使用 `ScopedLock` 保证线程安全。
2.  **记录内存状态**：将文件名加入 `_inMemFiles` 集合，以便后续 `IsExist` 查询能快速返回结果。
3.  **获取写日志文件**：调用 `GetLogFileForWrite` 获取或创建一个可写的 `LogFile` 实例。
4.  **创建 `ArchiveFile`**：实例化一个 `ArchiveFile` 对象，该对象代表了要创建的逻辑文件。它会生成一个唯一的签名（`signature`），用于在 `LogFile` 中唯一标识该文件的各个数据块。
5.  **返回 `ArchiveFileWriter`**：最后，用创建的 `ArchiveFile` 构造一个 `ArchiveFileWriter` 并返回给上层。

**核心代码 (`ArchiveFolder::CreateInnerFileWriter`)**:
```cpp
FSResult<std::shared_ptr<FileWriter>> ArchiveFolder::CreateInnerFileWriter(const std::string fileName) noexcept
{
    string signature = StringUtil::toString(_logFileVersion) + "_" + StringUtil::toString(_fileCreateCount);
    ArchiveFilePtr archiveFile(new ArchiveFile(false, signature));
    auto [ec, logFile] = GetLogFileForWrite(_logFileSuffix);
    RETURN2_IF_FS_ERROR(ec, std::shared_ptr<FileWriter>(), "GetLogFileForWrite failed");
    RETURN2_IF_FS_ERROR(archiveFile->Open(fileName, logFile), std::shared_ptr<FileWriter>(), "Open [%s] failed",
                        fileName.c_str());
    _fileCreateCount++;
    return {FSEC_OK, ArchiveFileWriterPtr(new ArchiveFileWriter(archiveFile))};
}
```

#### 文件查找 (`IsExist` 和 `CreateFileReader`)

查找文件时，`ArchiveFolder` 会按照以下顺序进行：

1.  **内存查找**：检查 `_inMemFiles` 集合。
2.  **归档日志查找**：遍历 `_readLogFiles`（已按版本号从新到旧排序），调用 `ArchiveFile::IsExist` 在每个 `LogFile` 中查找文件的元数据。
3.  **物理文件系统查找**（`legacyMode`）：如果开启了旧模式，还会检查物理目录中是否存在同名文件。

`CreateFileReader` 的逻辑与 `IsExist` 类似，一旦在某个 `LogFile` 中找到文件，就会创建一个 `ArchiveFile` 实例并用它构造一个 `ArchiveFileReader` 返回。

### 3.2. `ArchiveFile`：逻辑文件的“化身”

`ArchiveFile` 是对存储在 `LogFile` 中的逻辑文件的抽象。它的核心是**分块读写**。

#### 文件写入 (`Write` 和 `Close`)

*   **缓冲写入**：`Write` 方法接收到数据后，首先将其写入内部的 `_buffer`（一个 `FileBuffer` 实例）。
*   **刷盘（DumpBuffer）**：当 `_buffer` 写满时，`DumpBuffer` 方法会被触发。它会：
    1.  为当前数据块生成一个唯一的 `blockKey`（由文件名、签名和块索引构成）。
    2.  将 `_buffer` 中的数据通过 `_logFile->Write` 写入物理日志文件。
    3.  将数据块的元信息（`key` 和 `length`）记录在 `_fileMeta.blocks` 向量中。
    4.  清空 `_buffer`，准备接收下一批数据。
*   **关闭与元数据写入**：`Close` 方法是写入流程的最后一步。它会确保所有剩余在 `_buffer` 中的数据都被刷盘，然后将包含所有块信息的 `_fileMeta` 对象序列化成 JSON 字符串，并以一个特殊的键（`_archive_file_meta` 后缀）写入 `LogFile`。这个元数据是未来读取该文件的关键。

**核心代码 (`ArchiveFile::Close`)**:
```cpp
FSResult<void> ArchiveFile::Close() noexcept
{
    if (_isReadOnly) {
        return FSEC_OK;
    }
    if (_buffer->GetCursor() > 0) {
        RETURN_IF_FS_ERROR(DumpBuffer(), "DumpBuffer failed");
    }
    string fileMeta;
    RETURN_IF_FS_ERROR(JsonUtil::ToString(_fileMeta, &fileMeta).Code(), "ToString failed");
    string fileMetaName = GetArchiveFileMeta(_fileName);
    RETURN_IF_FS_ERROR(_logFile->Write(fileMetaName, fileMeta.c_str(), fileMeta.size()).Code(), "Write failed");
    RETURN_IF_FS_ERROR(_logFile->Flush().Code(), "Flush failed");
    return FSEC_OK;
}
```

#### 文件读取 (`Read` 和 `PRead`)

读取操作的核心是 `CalculateBlockInfo` 和 `FillBufferFromBlock`。

*   **`CalculateBlockInfo`**：当用户请求从某个 `offset` 读取 `length` 长度的数据时，此方法会计算出这次读取请求跨越了哪些数据块（Block），以及在每个块内的起始位置和读取长度。
*   **`FillBufferFromBlock`**：此方法负责按需加载数据块。如果请求的数据块（由 `blockIdx` 标识）不是当前已缓存的块，它会从 `_logFile` 中读取整个块的数据到 `_buffer` 中。
*   **数据拷贝**：一旦数据块被加载到 `_buffer`，`Read` 方法就会从 `_buffer` 中将请求的数据拷贝到用户的缓冲区中。

这种**按需加载**和**块缓存**的机制，使得即使面对非常大的逻辑文件，也只需要在内存中保留一个块大小的缓冲区，极大地节省了内存。

**核心代码 (`ArchiveFile::Read`)**:
```cpp
FSResult<size_t> ArchiveFile::Read(void* buffer, size_t length) noexcept
{
    size_t readLength = 0;
    vector<ReadBlockInfo> readBlockInfos;
    CalculateBlockInfo(_offset, length, readBlockInfos);
    char* cursor = (char*)buffer;
    for (size_t i = 0; i < readBlockInfos.size(); i++) {
        RETURN2_IF_FS_ERROR(FillBufferFromBlock(readBlockInfos[i], cursor), readLength, "FillBufferFromBlock failed");
        readLength += readBlockInfos[i].readLength;
        cursor += readBlockInfos[i].readLength;
        _offset += readBlockInfos[i].readLength;
    }
    return {FSEC_OK, readLength};
}
```

### 3.3. `ArchiveDirectory`：标准接口的“门面”

`ArchiveDirectory` 的实现相对直接，它主要充当一个**适配器**或**门面（Facade）**，将标准的 `IDirectory` 接口调用转换为对 `_archiveFolder` 的调用。

*   **路径转换**：`ArchiveDirectory` 内部维护一个 `_relativePath`。所有针对此目录的文件操作，其路径都会通过 `PathToArchiveKey` 方法与 `_relativePath` 拼接，形成在 `ArchiveFolder` 中全局唯一的键。
*   **接口映射**：
    *   `CreateFileWriter` -> `_archiveFolder->CreateFileWriter`
    *   `CreateFileReader` -> `_archiveFolder->CreateFileReader`
    *   `IsExist` -> `_archiveFolder->IsExist`
    *   `ListDir` -> `_archiveFolder->List`
*   **不支持的操作**：许多 `IDirectory` 的高级功能，如 `RemoveFile`, `Rename`, `Sync` 等，在 `ArchiveDirectory` 中都未被支持（返回 `FSEC_NOTSUP`），这反映了其作为归档系统的设计限制——主要为一次写入、多次读取的场景优化。

## 4. 技术风险与考量

1.  **不支持并发写同一个 `ArchiveFolder`**：虽然 `ArchiveFolder` 内部使用了 `ThreadMutex` 来保护其状态，但其设计（特别是 `_logFileVersion` 的递增机制）并不支持多个进程或机器同时对同一个 `ArchiveFolder` 实例进行写操作。这可能导致版本号冲突和数据不一致。它适用于单个索引构建任务内的文件操作。
2.  **无文件删除和修改机制**：系统是追加式的，不支持物理删除或修改文件。如果需要更新文件，只能写入一个同名的新版本。这会导致旧版本数据成为垃圾，占用存储空间。系统缺乏内置的垃圾回收（GC）机制来清理这些无用数据。
3.  **元数据单点风险**：`ArchiveFile` 的元数据（`_archive_file_meta`）在文件关闭时才写入。如果在此之前进程异常退出，这个逻辑文件就会因为缺少元数据而无法被读取，造成数据丢失。可以考虑引入元数据定期刷盘或预写日志（WAL）来增强鲁棒性。
4.  **`ListDir` 性能问题**：`ListDir` 的实现需要遍历所有 `LogFile` 中的元数据，并结合物理目录的列表。当归档文件数量和 `LogFile` 数量非常多时，这个操作可能会变得很慢。
5.  **`legacyMode` 带来的复杂性**：`legacyMode` 的存在使得系统行为不统一，增加了理解和维护的复杂性。它允许文件同时存在于归档和物理文件系统中，可能会导致路径解析和文件定位的混乱。

## 5. 总结

`ArchiveDirectory`, `ArchiveFolder`, 和 `ArchiveFile` 共同构成了一个功能完善的虚拟归档文件系统。它通过将小文件聚合到大日志文件中的方式，巧妙地解决了海量小文件带来的性能瓶颈。其分层设计、读写分离、分块存储等思想，使其成为一个高效、可扩展的小文件存储解决方案，尤其适用于 Indexlib 的索引构建场景。

尽管存在一些技术风险，如缺乏并发写支持和垃圾回收机制，但其设计清晰地反映了其在特定场景下的权衡与优化：**为一次性、大规模的顺序写和后续的随机读提供高性能**。理解其设计哲学和实现细节，对于深入掌握 Indexlib 的文件系统和性能优化至关重要。
